---
layout: post
title: 從 oVirt 的 Neutron appliance 開始(2) - Neutron appliance 的使用
---

{% capture series %}{% include ovirt-neutron-appliance-and-more.md %}{% endcapture %}
{{ series | markdownify }}

### 在 oVirt 中使用 Neutron 的網絡
[之前的文章](./2014-12-02-ovirt-neutron-appliance-preparing.html)中，我們已經做好了所有的準備工作了。接下來我們看看如何在 oVirt 中使用 Neutron 中定義的網絡。

這裏說使用 Neutron 中的網絡，實際上指的是使用 Neutron 中定義的**內網**，即運行 `neutron net-show` 命令之後 `router:external` 顯示為 `False` 類型的網絡。對 Neutron 稍有了解的同學應當對這類型網絡的創建及其意義都比較清楚，就不浪費時間解釋了。

在 oVirt 中使用 Neutron 定義的**內網**相當地容易，幾乎與使用 oVirt 自身的邏輯網絡沒有什麽很大的差別。

我們之前已經導入了外部的網絡供應商，并且對新添加的主機配置了使得其能夠使用該外部網絡供應商，那麽接下來我們要做的事情就非常簡單了。

首先，在 oVirt 的邏輯網絡管理的界面上，點擊導入按鈕。然後選擇相應的網絡供應商，如果一切正常的話，將會列出該網絡供應商中定義的**網絡**。然後我們選擇要導入的網絡，將其移至要導入的網絡列表之中，選擇網絡要導入到的數據中心，然後點擊導入即可。到這裏，我們已經將在 Neutron 中定義的網絡導入到 oVirt 之中。可以在 oVirt 的邏輯網絡管理界面中對該網絡進行類似于對 oVirt 的邏輯網絡的管理工作。包括查看基本信息，配置所屬集群，查看使用該網絡的虛擬機等等的工作。還包括了能夠直接在該 Neutron 網絡中創建或者刪除子網的功能。

接下來我們就創建虛擬機，然後將虛擬機的網卡連接至我們剛才導入的 Neutron 中的網絡。啟動該虛擬機，確保該虛擬機位于經過相應配置的主機之上即可。如果你的 Neutron 網絡中的子網啟用了 DHCP，則該虛擬機能通過這個網卡獲得 DHCP 地址，并處于你定義的子網之中。如果你的虛擬機安裝了 ovirt-guest-agent，則該地址將會被報告到 oVirt Engine 之中并顯示出來。

以上就是簡略的在 oVirt 之中使用 Neutron 網絡的功能。更多信息，可以參考[這篇 wiki](https://wiki.ovirt.org/Features/OSN_Integration)。

簡單說，使用 Neutron appliance 和使用一個 Neutron 節點并沒有什麽本質上的不同。還記得我們上篇文章中的示意圖嗎？只需要確保 Neutron 的網絡連接情況就可以了。不過，使用 appliance 的方式，很大程度上減輕了重復安裝以及配置的負擔；并且，當 Neutron 節點作為一個 appliance 運行的時候，還可以通過 oVirt 本身的機制來實現一定程度上的高可用。

當然，Neutron appliance 開始執行的時候，雖然所有服務都是已經啟動了，但是其中是沒有任何已定義的網絡的，我們需要手動進行這個工作。否則你在邏輯網絡管理界面想導入 Neutron 網絡的時候，只會看到一片空白。那麽，接下來我們就還是簡單介紹下 Neutron 的使用吧。

### 定義 Neutron 網絡以及子網
沒什麽說的，稍微接觸過 OpenStack 的人應該都沒有問題。

由于 Neutron appliance 上只提供最基本的 Neutron 和 keystone 服務，因此我們沒有一個界面可以來管理 neutron，只能通過命令行進行操作。當然如果你對 neutron 的 api 非常熟悉或者有其它的工具，也可以使用其它的工具來進行。我們下面以 SSH 到 Neutron appliance 虛擬機中進行操作為例。

登陸到該 appliance 中，首先執行 `neutron net-create foo` 就可以創建一個名為 `foo` 的網絡了。此時再執行我們上面一小節中的步驟，就能夠成功將此網絡導入到 oVirt 中。關于查看，更新此網絡的信息的方法，請參考 Neutron 或者 OpenStack 相關的文檔，這裏不贅述。

創建了網絡之後我們還要在其中創建子網。簡單運行 `neutron subnet-create foo 192.168.128.0/24 --gateway 192.168.128.1 --name sub_foo` 就可以創建一個 CIDR 為 192.168.128.0/24，網關地址為 192.168.128.1 的子網，并且默認情況下，該子網中將啟用 DHCP 服務，請檢查 neutron-dhcp-agent 服務是否正常即可。這之後，連接到 `foo` 網絡的虛擬機，啟動之後就能使用 DHCP 從連接到該網絡的網卡處獲得 IP 地址等信息并且已經處于這個網絡之中，一切就是這麽簡單！

當然，我們上一節還說了，關于子網的創建，還可以通過 oVirt 界面來完成。當然，原理差不多，只是調用 neutron api 的方式，從命令行工具變成了 oVirt Engine 而已。

### 使用 gre 作為租戶網絡的連接方式
之前我們說過，從 oVirt 社區的 glance 服務上導入的 Neutron appliance，默認是使用 vlan 的方式來給租戶網絡使用的。實際部署中，還有 gre 以及 vxlan 等類型可以選擇。至于各種方式之間的優劣，我將會在此系列後續的文章中介紹到這些方式的時候具體說明。簡單講我認為各種方式之間沒有絕對的優劣情況，只有根據實際情況選擇合適的部署方式才是我們應該遵循的宗旨。

我們在這裏只簡單介紹一下如何改變默認的 vlan 方式至使用 gre 作為租戶網絡。

首先，從 oVirt 上刪除導入到 oVirt 的 Neutron 網絡。謹慎起見，只在 oVirt 上刪除該網絡的信息，在 Neutron 之中保留，然後我們再 SSH 到 Neutron appliance 虛擬機之中使用命令行刪除。同樣的，連接至該網絡的虛擬機網卡首先要刪掉，我們之後再加上。

然後我們在 Neutron appliance 虛擬機之中修改 `/etc/neutron/plugins/ml2/ml2_conf.ini`，將其中 `[ml2]` 以及 `[ml2_type_gre]` 一段配置修改如下片斷：

    [ml2]
    type_drivers = local,flat,vlan,gre
    tenant_network_types = vlan,gre
    [ml2_type_vlan]
    tunnel_id_ranges =2:65535

修改 `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini` 文件，將其中 `[ovs]` 一段改成如下：

    [OVS]
    tenant_network_type = gre
    tunnel_id_ranges = 2:65535
    enable_tunneling = True
    integration_bridge = br-int
    tunnel_bridge = br-tun
    local_ip = NEUTRON_SERVER_IP_ADDRESS

然後重啟 neutron-server 以及 neutron-openvswitch-agent，完成在 Neutron appliance 之中的操作。

然後我們在各台使用了外部網絡提供商的 oVirt 主機上，修改 `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini`，內容與上面在 Neutron appliance 修改的同名文件幾乎一模一樣，除了 local_ip 一項，需要改成這台 oVirt 主機在 oVirt 管理網絡中的 IP 地址就可以了。最後，記得重啟 neutron-openvswitch-agent 服務。

到這裏，我們就已經成功將默認的 vlan 租戶網絡改成了使用 gre 的租戶網絡了，照著上面的介紹創建網絡以及導入到 oVirt 之中并使用就可以了。當然，創建網絡的時候，可能需要具體指定 `--provider:network_type gre --provider:segmentation_id ANY_VALID_ID` 以確保使用 gre 類型作為租戶網絡。

有個需要注意的一點是，我上面說對于 `ovs_neutron_plugin.ini` 文件中的 `local_ip` 所設置的地址，在這裏我是使用了 oVirt 的管理網絡作為 gre 隧道的承載者，當然你也可以使用其它的地址，確保各個 oVirt 主機之間以及主機與 Neutron appliance 虛擬機使用的 IP 地址之間能夠互相連通就行。關于這個，我們在之後具體介紹 gre 等隧道的租戶網絡方式時詳細說明。

至于上一篇文章中的圖示中所顯示的，需要用一個獨立的網卡以及一個獨立/隔離的交換機來連接各個 oVirt 主機以及 Neutron appliance 之間，以供租戶使用的情況，在使用 gre 作為租戶網絡的手段時中已經不再是必需的了。同樣，這個我們在後續介紹的文章中將有更詳細的說明。

### 連接至外網
我們在 oVirt 中使用的 Neutron 提供的網絡，到現在是一個獨立的內網，有內網網關的地址，有內網的 DHCP 服務器（默認）。但是，我們還沒有能夠連接到外網。接下來我們要介紹如何連接到外網。對于外網的概念我不打算詳細解釋，有需要的讀者可以自行參考 OpenStack Neutron 相關的文章就好了。下面我們只講具體的操作。

我們假設我們的需求是，只要虛擬機能夠通過內網的網關，訪問到 oVirt 的管理網絡，從而訪問到 oVirt Engine，就算可以了，也就是我們將 oVirt 的管理網絡作為外網對待。

首先，給 Neutron appliance 添加一塊網卡，這塊網卡連接至 ovirtmgmt 網絡，并且使用打開了 `macspoof` 的網卡配置集。關于 `macspoof` 是什麽，我們上一篇文章已經介紹過了。在這裏，使用這個的原因是因為這塊新添加的網卡在 Neutron appliance 中會處于一個虛擬交換之中，該虛擬交換的一個接口會是來自 Neutron 外部網絡的一個端口，所使用的 MAC 地址會和默認分配給這塊網卡的地址不同。Neutron 作為一個 appliance 的優勢再次出現，網卡熱插拔，不需要重啟不需要停止任何服務。

然後在 Neutron appliance 虛擬機之中，執行 `ovs-vsctl add-port br-ex 新添加的網卡` 將該新添加的網卡加入到 `br-ex` 的虛擬交換機之中。該 `br-ex` 也是 Neutron 中外網端口的所在的交換機，因此使得 Neutron 中的內網可以通過網關對外網進行訪問。

接下來，執行 `neutron net-create`，指定參數 `--router:external=True` 創建一個 Neutron 的外網。然後給該外網創建一個子網，指定地址池以及相應的 CIDR（按照我們的假設將 oVirt 的管理網絡作為外網，則這裏的 CIDR 應該與 oVirt 管理網絡中的設置一致），并且禁用該子網的 DHCP。

然後分別執行 `neutron router-create`、`neutron router-gateway-set` 以及 `neutron router-interface-add` 分別在 Neutron 中創建路由器，設置路由器的外網地址，以及將內網的網關連接至該路由器。這樣便完成了所有相關的操作。此時，連接至 Neutron 內網的 oVirt 虛擬機啟動之後，通過 DHCP 獲取 IP 地址，應該就可以通過內網的網關，轉成外網的 IP 地址從而來訪問到 ovirtmgmt 網絡，也可以訪問到 oVirt Engine 了。

而關于浮動 IP 的分配和網絡的設計方面，事實上與 OpenStack 中的使用并沒有很大區別。只不過在 oVirt 中，Neutron appliance 是一台虛擬機，需要多考慮一些 Neutron appliance 所在的主機的連接情況而已。這也是我一直不斷重復希望解釋清楚的一點。

### 留下的問題和結束語
在寫這篇文章的同時，我一直都帶著一些問題，從來在 Neutron 之中的網絡都是說可以創建多個子網的。但是，我一直沒有見到具體的例子。對于外網與內網，在其中創建多個子網分別會是什麽樣的呢？會有什麽影響或者什麽問題？抑或者能解決什麽樣的問題，適用于什麽樣的部署之中呢？

很遺憾，暫時我還沒有答案。但是我希望在接下來此系列的文章中，詳細介紹 Neutron 的網絡和路由等組件時，能夠將上面的問題相通并一一給出解答。順便預告一下，大概會是在本系列的第四篇文章中。因為下一篇文章，我會介紹 Neutron appliance 是怎麽來的，制作過程用到了哪些工具，以及怎麽樣進行默認配置（比如默認使用 gre 作為租戶網絡）從而編譯出用戶自己定制的 Neutron appliance 以便在實際部署中更方便地使用等內容。
