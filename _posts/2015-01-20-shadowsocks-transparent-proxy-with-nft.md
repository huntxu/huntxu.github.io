---
layout: post
title: 使用 nftables 与 ss-redir 实现 shadowsocks transparent proxy
---

新年快乐！

新年伊始，原来一直使用的 PPTP 来翻墙的服务突然出问题了，直接就是各种连不上服务器。后来想想，好像和人合租的这个 VPS 已经到期了，果然借别人的机器墙外机器都访问不到。在天朝翻不过墙几乎就等于上不了网，于是没办法，得找其它的方式解决。

我本来是比较倾向使用 VPN 方式来翻的，因为这样子不需要特别的设置，相对讲更为方面一些。shadowsocks 之前就有听说过，但是似乎都是浏览器上使用多点。于是昨天跑到 [github 上 shadowsocks 项目的项目主页](https://github.com/shadowsocks)上面看，发现 shadowsocks-libev 提供了 ss-redir 这样一个客户端能够实现 transparent proxy，果断尝试使用。服务器自然已经有很多朋友在用，临时就借用了一个来用，如果有必要的话后面自己可能得买个 VPS 或者怎么的再说。客户端没什么说的，aur 上面找到有人已经写好的 PKGBUILD，然后添加个为 ss-redir 的 systemd service，打包装上就好了。

[这里的例子](https://github.com/shadowsocks/shadowsocks-libev#advanced-usage)中给出的是使用 iptables，其原理是将特殊的流量重定向到本地的 shadowsocks 服务端口，于是我又想起前段时间学习过一些 nftables 的东西，不妨就此试试也当练练手，因此有了这篇文章。

根据[这里](http://lwn.net/Articles/626524/) nftables 0.4 的发布邮件，刚刚好在 linux 3.19-rc 里面支持了 redirect 功能。

> * Redirect support (available since upcoming Linux kernel 3.19-rc).
>
>        # nft add rule nat prerouting tcp dport 22 redirect to 2222

那就果断上吧，好在我一直以来都是自己编译的内核，本地有个内核代码的仓库。为了简单起见，个人桌面使用的内核也没有去编译相关的 netfilter 之类的模块，所以反正都是要编译新内核的了。看了下内核的提交，刚好是 3.19-rc5，那就编译吧。需要用到的模块主要有 netfilter 相关的 conntrack 和 nat 和 redir，nftables 和它的 nat 和很新的 redirect 功能，ipv4 对应上面这些的相关选项，以下是不完全列表(RBTREE 和 HASH 两个我记得是因为性能原因我才选上的，本来想在 nftables 的 set 定义时用的)：

```
CONFIG_NF_CONNTRACK=m
CONFIG_NF_NAT=m
CONFIG_NF_NAT_NEEDED=y
CONFIG_NF_NAT_REDIRECT=m
CONFIG_NF_TABLES=m
CONFIG_NFT_CT=m
CONFIG_NFT_RBTREE=m
CONFIG_NFT_HASH=m
CONFIG_NFT_REDIR=m
CONFIG_NFT_NAT=m
CONFIG_NF_CONNTRACK_IPV4=m
CONFIG_NF_TABLES_IPV4=m
CONFIG_NF_NAT_IPV4=m
CONFIG_NFT_CHAIN_NAT_IPV4=m
CONFIG_NFT_REDIR_IPV4=m
```

编译完成之后就用上了新内核，装好 nftables 相关的用户态的包，然后启动 ss-redir，试着写了几个简单的 nftables 规则，发现总是连接失败，抓包看到的结果是连本地的代理都连接不上，代理倒是返回了 SYN,ACK，可是程序却直接返回了一个 RST。

换成 ss-local 和浏览器代理的方式，又一切正常，说明服务端是没有问题的。花了几个小时，始终无法找到是哪里出了问题。最终一个同事耐心的建议我再仔细研究下防火墙的配置，最终我终于发现，原来我把 nftables 很重要的一项设置给忘了。

事情是这样的，原来在 iptables 里面的时候，table 和 chain 都有预定义，在 nftables 里面，是没有的。我添加了 chain，hook 了 output，添加了重定向的规则，这也算是某种 nat，还**至少需要定义一条空的 chain 去 hook input**，这样子才能让对应那个重定向返回来的包能够正确返回。添加上去，一切就正常工作了。这下再也不会忘记这个了...

最后就是一些收尾的工作了，按照原来使用 PPTP 的方式，用路由来决定哪些地址走本地的网络，现在就是将地址直接写在 nftables 的规则里面让访问那些地址的内容不重定向就好了。还是照样，从 apnic 下载下来国内的地址信息然后处理了下。再加上个 pdnsd 用来 tcp 查询路由，避免了 dns 的污染，世界又重新变得美好了。

每次添加都添加那么多条显然是不现实的，好在 nft 有个好处，可以用 -f 参数直接添加整个表。于是就把这个表存起来，在 ss-redir 的 systemd service 文件里写了 ```ExecStartPre``` 将表给读取，写了 ```ExecStopPost``` 将这个定义的表给删除。

最后吐槽，nftables 还是很 buggy 的，但是简单的使用还是能令人看好其以后的发展前景。我本来想定义命名 set 来包含各种不需要重定向的地址，然后动态往这个 set 里面添加和删除内容就好了，谁知道我用 ```nft add element``` 却发现没办法接受 CIDR 的地址格式，只能一个一个写，所以放弃了这个想法。当然另外也有一个值得赞的地方，我从 apnic 下载下来的文件然后生成 cn 的地址段，就是不需要重定向的那些，最终生成了 5000 多条地址段的信息，nftables 读取之后智能地把连续的地址段都转化成其内部的 interval 表示了，减少了大概一半的数量。还有个地方是 ```nft -f``` 是一个原子性的操作，而且速度还很快，至少比之前使用脚本一条一条路由添加的方式要快。至于性能，暂时倒没有看到什么特别严重的影响，可能个人使用还是看不出来，有空再往各种服务器上使用 nftables 试试。

写了个简单的 perl 脚本 ```ssnft.pl```，用来生成 nftables 的规则表。

```
#!/usr/bin/perl -w
use strict;

# curl http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest| perl ./ssnft.pl > nfttables
# nft -f nfttables

my $local_port = 8888;              # 本地 ss-redir 服务的端口
my @noss = (                        # 不需要重定向的预定义地址，千万记得将 ss 远程服务器的地址加上！
    "127.0.0.0/8",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "169.254.0.0/16",
    "224.0.0.0/4",
    "240.0.0.0/4",
);

my %prefixlen;
foreach (1..31) {
    $prefixlen{2**$_} = 32-$_;
}

while(<>) {                         # 具体用法看上面的注释，从 apnic 下载地址信息处理下
    chomp;
    next unless /^apnic\|CN\|ipv4/;
    my @ent = split /\|/;
    push @noss, sprintf "%s/%s", $ent[3], $prefixlen{$ent[4]};
}

my $result = <<__NFT;
table ip shadowsocks {
    chain output {
        type nat hook output priority -100;                 # netfilter internal operation: NF_IP_PRI_NAT_DST (-100)
        ip daddr { NOSS_ADDRESSES } return                  # 不需要重定向的地址就 return 吧
        tcp sport {32768-61000} redirect to $local_port     # /proc/sys/net/ipv4/ip_local_port_range，本地发起的连接的端口范围
    }
    chain input {                                           # 即使这条 chain 是空的，也需要定义
        type nat hook input priority -100;
    }
}
__NFT

my $noss_addresses = join ", ", @noss;
$result =~ s/NOSS_ADDRESSES/$noss_addresses/;
print $result;
```
