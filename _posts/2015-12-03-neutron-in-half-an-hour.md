---
layout: post
title: 半小时 neutron 教程
---

首先，neutron 是什么就不介绍了，这里的 neutron 指的是 OpenStack 的一个子项目，不知道是什么的可以搜索，或者点[这里](https://wiki.openstack.org/wiki/Neutron)去慢慢了解。

这次的题目是起得有点大了。想要半个小时就明白 neutron 中的方方面面以及大部分细节，是几乎不可能的。但是，半个小时却足够大概了解 neutron 的框架，如果学得足够快也有测试环境，半个小时到一个小时之后，你就可以对 neutron 进行 debug，甚至于给 neutron 写插件或者加功能了。

说到 neutron 的插件，插句题外话。在我刚开始学着给 neutron 写插件的时候，相关的文档很难找到，当时就只能找到[这篇文章](http://control-that-vm.blogspot.jp/2014/05/writing-api-extensions-in-neutron.html)，然后参考着写出来了第一个 extension。于是我今天都一直留着在我的收藏夹之中，准备哪一天我又忘了怎么写的时候可以随时再查到。为什么插这句题外话呢，是因为这篇文章中有一句吐槽非常精准，原文如下： The following link about [API Extensions](https://wiki.openstack.org/wiki/NeutronDevelopment#API_Extensions) is self-explanatory only if you know how to write an API extension (:P)。链接指向 OpenStack wiki 上关于 neutron 开发的页面，翻译为“如果你知道怎么给 neutron 写 API 扩展，那么 wiki 的这篇教程就是不解自明的了”。其实就是吐槽社区 wiki 上的内容语焉不详。

我不喜欢满篇都是谈论又空又泛的话题诸如整体架构如何设计，前景如何，优缺点在哪里而根本不提代码的风格，所以下面的分析还是会结合代码来看的。

我也不喜欢无脑过代码，主张把代码通读一遍再开始进行修改创作这种风格的。理由非常简单，给汽车换轮胎并不需要了解发动机内部的细节。在开源项目的大背景下，代码更新有时甚至比人读代码的速度还要快，等到读完了发现上游代码又焕然一新了，于是再从头读一遍。所以对代码进行详尽的分析就别指望了，不喜欢把函数调用栈打出来然后就说是这个样子的，更不喜欢的是贴很长很长一段代码然后说明信息比代码少了又少的（我甚至怀疑这种风格的文章代码之后的文字说明是不是从别处抄来的）。其实有基础的话，大家需要的都不是“这个代码是怎样的”（因为每个人自己都能看懂），而是“我该如何入手”。

直到今天，我都没看过多少 neutron 的代码，但我已经给 neutron 写过多种类型的扩展（扩展现有的 resource，使用相同 plugin 增加新的 resource，新的 plugin + 新的 resource + 独立 agent），并且到今天它们还算正常的在运行着（当然不保证没有 bug）。而且，我也有信心运行中出现的问题能够很快地定位到并且找到原因。原因嘛，并非因为我有多了不起，而是因为 neutron 整个逻辑确实非常清晰单纯，（刨去厂商各自的实现之后）代码量也不大，所以大概的结构了解了之后，结合调试日志等，对 neutron 进行 debug 甚至改动都不是多难的事情。

还是先给出张图吧，结合着图表来说问题更清晰点（我越来越喜欢 vim 的 DrawIt! 了）：

            +-------------------------------------------------+
            |        Neutron Server                           |
            |                                                 |
            |    ExtensionManager           NeutronManager    |
            |   ------^-----------         --------^-------   |
            |         |                            |          |
            |         |   +---------------------+  |          |
            |         |   | Neutron Plugin      <--+          |
            |         |   +--^----+             |             |            +---------------+
            |         |      |    |             |             |            | Message Queue |
            | A|      |  +---v-+  |      +------+             |            |   <======     |              +-------+
    人------> P<-+    +--> api |  |      | rpc  <-------------+------------>     ======>   <--------------> Agent |
            | I| |       +--^--+  +--^---+------+             |            +---------------+              +-------+
            |    +----------+        |                        |
            +------------------------+------------------------+
                                     |
                             +-------v----+
                             |            |
                             |  DataBase  |
                             |            |
                             +------------+

上面的图就是 neutron 简单版本的结构，下面是文字版本的 API 流程说明

1. 人发起 API 请求
2. API 请求到达 Neutron Server 运行着的 wsgi（上图 Neutron Server 框框内部左下角的竖着写的 API）
3. wsgi 根据请求匹配路由，这部分在上图 Neutron Plugin 左下角的 api 小框中进行，这些 API Resource 由 ExtensionManager 生成
4. 路由之后选择对应的 controller 完成任务，上图中 api 小框到 Neutron Plugin 的连接
5. 如果需要的话，Neutron Plugin 和数据库交互
6. 如果需要的话，Neutron Plugin 通过 rpc 和 agent 交互
7. 一路返回结果给人
8. Agent 有定时任务通过 rpc 接口和 plugin 交互，plugin 同样看需要和数据库交互并且将 Agent 请求的内容返回给 Agent

简化了之后就是上面所说的这样子。我们从上面的图可以看到，整个 Neutron Server 有两个地方向外暴露，一个是和人打交道的 api，wsgi 跑着一个 app 接收 RESTful 请求并且返回。另外一个是和 agent 打交道的是 rpc，通过 rabbitmq 等消息队列进行远程调用。那么我们现在来慢慢的将上面的模型复杂化。

首先是 Neutron Plugin，上面的图中画了一个 plugin。而其实在 Neutron Server 内部存在的 plugin 有很多个。Neutron 的结构是插件化的，核心的 plugin 只负责三种资源：network，subnet 以及 port。其他的功能，包括乱七八糟的 router 以及它的衍生功能，防火墙，负载均衡，VPN 等等，都是以 service plugin 的形式存在的。这些 plugin 做的事情逻辑上是相同的：

1. 和人打交道：处理人类的请求和返回
2. 和数据库打交道：增加、更新、删除或者获取数据库中的内容(CRUD)
3. 和 agent 打交道：通过 rpc 通知 agent 情况发生改变，接收 agent 的 callback 返回相应的内容

以上三个部分，在一个 plugin 中可有可无（当然正常情况下应该都有），并且会错综复杂交织在一起。例如：通过 API 请求创建某个项目，写入数据库，通知 agent 进行更新这样一个典型的流程。

然后是 API 和 RPC 的部分，这两个对外暴露的地方有个区别，API 的出入口是统一的，而 RPC 没有这回事。从 API 这边看，整个 Neutron Server 就是一个 WSGI APP，接收请求，匹配路由，发给对应的控制器，得到处理结果并且返回。而 RPC 这边，每一个 plugin 对 agent 进行远程调用时，都会直接发送出消息完成调用。但是如果是 agent 进行 callback，则通常时 plugin 提前注册回调的 endpoint，在回调的处理中设法找到这个 plugin，再执行 plugin 中的函数（并返回）。这两个部分只在 Neutron Server 启动时需要加以关注，过程稍显复杂，但是一旦启动之后，则会按照预定的方式工作，不需要过多关注，因此通常重点都会是在上面所说 plugin 的部分。

下面我们开始代码之旅，结合着代码来分析上面所说的这些。

### 程序入口
程序启动的入口在`neutron/server/__init__.py`

```
def main():
    ...
    try:
        ...
        neutron_api = service.serve_wsgi(service.NeutronApiService)
        ...
        try:
            neutron_rpc = service.serve_rpc()
```

忽略掉无关紧要的内容，我们可以看到这里两个重点就是上面说过的 API 和 RPC 两大巨头。
    
相比之下，这里的 neutron_rpc 并不是重点，上面说过了，RPC 的入口不是统一的，这里的 serve_rpc 其实只是设置了 Neutron core plugin 这个插件的 rpc callback 入口而已，至于其他服务插件的，则是在它们的初始化过程中完成的，这些过程就是上面说的为了 agent 进行回调而提前注册的 endpoint。所以这里就不讲这个 rpc 了，放到某个插件的初始化之中讲效果是一样的。

那么接下来看重点 neutron_api，看字面意思，这里就是启动了一个 wsgi app。

### WSGI 部分
这些代码都在`neutron/service.py`之中

```
class WsgiService(object):
    ...
    def __init__(self, app_name):
        self.app_name = app_name
        self.wsgi_app = None

    def start(self):
        self.wsgi_app = _run_wsgi(self.app_name)
    ...


class NeutronApiService(WsgiService):
    ...
    @classmethod
    def create(cls, app_name='neutron'):
        ...
        service = cls(app_name)
        return service


def serve_wsgi(cls):
    try:
        service = cls.create()
        service.start()
    ...
    return service


def _run_wsgi(app_name):
    app = config.load_paste_app(app_name)
    if not app:
        LOG.error(_('No known API applications configured.'))
        return
    server = wsgi.Server("Neutron")
    server.start(app, cfg.CONF.bind_port, cfg.CONF.bind_host,
                 workers=cfg.CONF.api_workers)
    ...
    return server
```

这个流程并不复杂，只不过包装来包装去而已。最终就是初始化了一个 NeutronApiService，这个 NeutronApiService 中有个与 app 对应的 server。server 监听网络端口交给 app 去处理。Neutron 启动时的代码就是这两大部分。台已经搭好，准备唱戏了。唱戏的就是这个名为`neutron`的 app，它是通过`paste.deploy`加载进来的。

### api-paste 配置文件
这个配置文件在`etc/api-paste.ini`，这里已经过滤掉无关紧要的内容了。

```
[composite:neutron]
use = egg:Paste#urlmap
/: neutronversions
/v2.0: neutronapi_v2_0

[composite:neutronapi_v2_0]
use = call:neutron.auth:pipeline_factory
noauth = request_id catch_errors extensions neutronapiapp_v2_0
keystone = request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0
...
[filter:extensions]
paste.filter_factory = neutron.api.extensions:plugin_aware_extension_middleware_factory
...
[app:neutronapiapp_v2_0]
paste.app_factory = neutron.api.v2.router:APIRouter.factory
```

这里不介绍 paste.deploy 的使用，稍微值得关注的是 paste.deploy 中的 filter 的概念，和 python 的修饰函数差不多，只不过修饰的对象是 app，返回一个修饰过的 app。

我们需要重点关注的是名为 neutronapiapp_v2_0 的 app 和名为 extensions 的 filter。通常情况下 filter 是用来对 app 进行包装的，相当于把输入输出什么的处理一下。但是在 neutron 之中，使用 filter 则是为了对核心部分进行扩展（下面我们会看到）。并且，这里面大量使用了 Routes 模块来创建 RESTful API 并且自动完成 URL 到 controller 的处理。

## 魔法的背后
到这里，我们已经知道了上面跑着一个 wsgi app，然后使用 Routes 匹配 URL 到不同的 controller 之上。所以需要关注的问题已经变成了以下几个：

1. Neutron 中那么多的插件是何时加载的，加载来干嘛
2. Neutron 中那么多的 RESTful 资源是怎么样创建出来的，就是说，Routes 要匹配哪些 URL
3. Routes 匹配到 URL 交给 controller，controller 和 Neutron 插件之间是什么关系

### Q: Neutron 中的插件是何时加载的，有什么作用
在最开始画的图我们就已经知道了，neutron plugin 是真正处理 api 请求的地方。也就是说，Routes 匹配 URL 找到 controller，controller 最终调用的就是 plugin 中对应的函数。我们先来看它是怎么加载进来的，这段代码在`neutron/api/v2/router.py`中

```
class APIRouter(wsgi.Router):
    ...
    def __init__(self, **local_config):
        plugin = manager.NeutronManager.get_plugin()
    ...
```

manager.NeutronManager 是一个单实例（我们后面要讲的 ExtensionManager 也是），这里是第一次用到它的地方，于是乎会初始化它。代码看`neutron/manager.py`

```
class NeutronManager(object):
    def __init__(self, options=None, config_file=None):
        ...
        plugin_provider = cfg.CONF.core_plugin
        LOG.info(_("Loading core plugin: %s"), plugin_provider)
        self.plugin = self._get_plugin_instance('neutron.core_plugins',
                                                plugin_provider)
        ...
        self.service_plugins = {constants.CORE: self.plugin}
        self._load_service_plugins()
```

这里不做展开，下面如果有需要讲到这个模块的某个细节的话会单独讲。可以看到其实逻辑很清晰，就是加载核心插件然后加载服务插件。然后保存了个 dict 方便随时查回 plugin，这个 dict 的 key 除了这个`constants.CORE`之外，其他的是来自于插件中的`get_plugin_type()`函数的返回值。这里通过读取配置文件加载插件，可以使用 stevedore 以 entrypoint 的方式加载，也可以使用 importutils 的方式从特定路径加载插件。

### Q: Neutron 中的 RESTful 资源是怎么样创建出来的
答案是，通过 Routes.mapper.resource 或者 Routes.mapper.collection 自动创建的。来看一下上面说过的那个 app 和 filter 各自开始的地方：

在`neutron/api/v2/router.py`中

```
class APIRouter(wsgi.Router):
    def __init__(self, **local_config):
        mapper = routes_mapper.Mapper()
        ...
        ext_mgr = extensions.PluginAwareExtensionManager.get_instance()
        ext_mgr.extend_resources("2.0", attributes.RESOURCE_ATTRIBUTE_MAP)

        col_kwargs = dict(collection_actions=COLLECTION_ACTIONS,
                          member_actions=MEMBER_ACTIONS)

        def _map_resource(collection, resource, params, parent=None):
            ...
            controller = base.create_resource(
                collection, resource, plugin, params, allow_bulk=allow_bulk,
                parent=parent, allow_pagination=allow_pagination,
                allow_sorting=allow_sorting)
            ...
            mapper_kwargs = dict(controller=controller,
                                 requirements=REQUIREMENTS,
                                 path_prefix=path_prefix,
                                 **col_kwargs)
            return mapper.collection(collection, resource,
                                     **mapper_kwargs)

        ...
        for resource in RESOURCES:
            _map_resource(RESOURCES[resource], resource,
                          attributes.RESOURCE_ATTRIBUTE_MAP.get(
                              RESOURCES[resource], dict()))
```

而在`neutron/api/extensions.py`中，有如下代码：

```
class ExtensionMiddleware(wsgi.Middleware):
    """Extensions middleware for WSGI."""

    def __init__(self, application,
                 ext_mgr=None):
        self.ext_mgr = (ext_mgr
                        or ExtensionManager(get_extensions_path()))
        mapper = routes.Mapper()

        # extended resources
        for resource in self.ext_mgr.get_resources():
            ...
            mapper.resource(resource.collection, resource.collection,
                            controller=resource.controller,
                            member=resource.member_actions,
                            parent_resource=resource.parent,
                            path_prefix=path_prefix)

        ...
```

这里非常明显使用了 Routes 这个模块自动地完成对 RESTful 资源的创建，这样一来匹配到 URL 之后就会交给 controller 完成相应的任务。这里的 controller 是通过调用 base.create_resource 来完成创建的，我们留待下面一部分讲这个。

值得注意的是，上面已经提到过了，这里的 ExtensionManager 是另外一个单实例，这个对象初始化的时候，通过读取配置文件中的路径实现对 extension 的加载，并且会检查已经加载的 plugin 中支持的 alias 是否全部被 extension 所支持的 alias 所满足（这个 alias 相当于从 extension 找 plugin 的一个方式）。

还有一个值得注意的地方是，extension 的加载，找的是 extension 的 .py 文件之中与文件名相同但首字母大写的类。

接下来在`APIRouter`的初始化之中还调用了一次`ext_mgr.extend_resources("2.0", attributes.RESOURCE_ATTRIBUTE_MAP)`，这儿其实就是对定义的资源进行扩展，方便给 resource 增加属性之类的工作。细节展开的话有些繁琐，最终的效果其实就是对每个 extension 中的`RESOURCE_ATTRIBUTE_MAP`进行扩展，从而为下面进行`mapper.resource()`做准备。

### Q: controller 怎么找到 plugin
这里有个转换过程，Routes 匹配到 URL 之后交给的是通过 base.create_resource 完成创建的 controller，我们看下这个 controller 怎么创建，在`neutron/api/v2/base.py`中：

```
def create_resource(collection, resource, plugin, params, allow_bulk=False,
                    member_actions=None, parent=None, allow_pagination=False,
                    allow_sorting=False):
    controller = Controller(plugin, collection, resource, params, allow_bulk,
                            member_actions=member_actions, parent=parent,
                            allow_pagination=allow_pagination,
                            allow_sorting=allow_sorting)

    return wsgi_resource.Resource(controller, FAULT_MAP)
```

这个代码中，wsgi_resource.Resource 是完成 xml/json 等的 serialize 与 deserialize 的操作的，所以无须关注细节。
我们只需要知道这个 Controller 是怎么样的就行。首先看上面这部分代码，注意到有个`plugin`参数在`Controller`初始化时传入，另外还有个`params`，前者就是对应的 neutron service plugin，后者就是 resource 对应的属性，比如说一个网络的名称等这些 property。Controller 的执行过程，大致上就是检查传入内容，找到 plugin 中对应的函数执行它，然后处理返回结果这样一个典型流程。

我们结合下面这个代码，来简单看下 neutron 中的资源是如何创建的。在`neutron/api/v2/base.py`中：

```
class Controller(object):
    LIST = 'list'
    SHOW = 'show'
    CREATE = 'create'
    UPDATE = 'update'
    DELETE = 'delete'

    def __init__(self, plugin, collection, resource, attr_info,
                 allow_bulk=False, member_actions=None, parent=None,
                 allow_pagination=False, allow_sorting=False):
        ...
        self._plugin = plugin
        ...
        self._plugin_handlers = {
            self.LIST: 'get%s_%s' % (parent_part, self._collection),
            self.SHOW: 'get%s_%s' % (parent_part, self._resource)
        }
        for action in [self.CREATE, self.UPDATE, self.DELETE]:
            self._plugin_handlers[action] = '%s%s_%s' % (action, parent_part,
                                                         self._resource)

    def create(self, request, body=None, **kwargs):
        """Creates a new instance of the requested entity."""
        ...
        action = self._plugin_handlers[self.CREATE]
        ...
        if self._collection in body and self._native_bulk:
            # plugin does atomic bulk create operations
            obj_creator = getattr(self._plugin, "%s_bulk" % action)
            objs = obj_creator(request.context, body, **kwargs)
            ...
            return notify({self._collection: [self._filter_attributes(
                request.context, obj, fields_to_strip=fields_to_strip)
                for obj in objs]})
        else:
            obj_creator = getattr(self._plugin, action)
            if self._collection in body:
                # Emulate atomic bulk behavior
                objs = self._emulate_bulk_create(obj_creator, request,
                                                 body, parent_id)
                return notify({self._collection: objs})
            else:
                kwargs.update({self._resource: body})
                obj = obj_creator(request.context, **kwargs)
                self._send_nova_notification(action, {},
                                             {self._resource: obj})
                return notify({self._resource: self._view(request.context,
                                                          obj)})
```
非常简单吧，Routes 会把对某个资源的 POST 定向到这个 controller 中的 create 函数。Controller 中的 create 的核心就是找到 plugin 中对应创建的函数，执行它并获得结果。 

Controller 通过其自身的`_plugin_handlers`这个表来完成到 plugin 中的函数的转换。有个非常简单的对应规则（还有复杂的 member action 我这里没有提到），`create_`前缀对应创建，`update_`对应更新，`delete_`对应删除，`get_`前缀并且后面是单数对应`show`，`get_`前缀的复数对应的是`list`也就是 RESTAPI 的`index`操作。

至此，我们从 API 就转入到 plugin 的流程里面了。

### 补充说明 RPC
其实 rpc 没有什么说的，隐藏在上面的 plugin 的加载之中。每个 plugin 的初始化，几乎都会设置好相应的 rpc 回调。Agent 进行 rpc 回调时，会到达这些设置好的回调`endpoint`，这些`endpoint`里面无一例外的，都是找到创建它的那个 plugin，执行 plugin 中的函数并依据情况返回。

其实整个的流程和 API 这边的逻辑没有什么不同，都是找到 plugin 并执行逻辑代码并返回。还有不明白的话，就好好领会最开始的那幅图吧，区别只在于访问 plugin 的方式。

## Neutron 的 plugin 是怎样做出来的
这里的 plugin，既包括 neutron 本身的 Core Plugin，也包括 Service Plugin，大家其实都是一样的，只不过代码位置略有不同罢了。

于是，如果要给 neutron 写扩展或者进行修改的话，无外乎以下几个步骤：

1. 定义 extension，声明它的资源及资源的属性，以及 extension alias
2. 定义 plugin，让 wsgi controller 能够找到这个 plugin 执行里面的函数，基本的函数无非就是上面提到过的前缀分别为`create_`，`update_`这几种。另外， plugin 还需要和数据库打交道。
3. 如果需要 agent 的支持，则需要设置好 rpc，让 plugin 和 agent 之间能够相互通信

实现良好的 plugin，一般至少是继承一个 Dbonly 的类，以及一个 rpc 类。使用前者的函数完成纯粹的数据库操作，使用后者的函数通知 agent 有新情况发生。以此为基础，再进行扩展。

所以其实前面所说的一堆分析，在你只是要给 neutron 做简单扩展或者 debug 的时候，其实没多大的用处。在写一个扩展的时候，可以把上面说的一堆乱七八糟的 API 和 RPC 等等的调用过程直接忽略，完全可以认为是已经实现了的库函数，直接使用就行。所以按照固定的 extension 和 plugin 模板填空即可，只需要记住以上三部曲，文章最开头的一个小时能够开始给 neutron 写扩展便是真的。

当然再好的库函数也不能保证没有 bug，所以知道大概的整个流程也是很有必要的。只不过这本来就是联系并不那么紧密的东西，可以认为是多个不同的模块，而 neutron 的核心，其实在 plugin 而不在 API 与 RPC 这些内容之上，一个是核心逻辑，一个是表面接口。而如果想了解 neutron plugin 的话，因为每个 plugin 都是要完成不同的业务的，只能靠着刚才上面所说的框架，去阅读每个模块的代码即可。而且，由于这些模块之间相对的独立性是比较高的，因此无论是出问题的时候要定位问题，还是需要给某个 plugin 增加新的功能，都不会是多复杂的事情，当然，前提是了解各个 plugin 的内在逻辑，以及它背后所使用的网络基础功能。

- - -
参考资料

* [Routes 模块的 RESTful](http://routes.readthedocs.org/en/latest/restful.html)
* [paste.deploy](http://pythonpaste.org/deploy/)
* [stevedore](https://github.com/openstack/stevedore)
