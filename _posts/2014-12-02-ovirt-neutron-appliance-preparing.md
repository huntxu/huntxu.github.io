---
layout: post
title: 從 oVirt 的 Neutron appliance 開始(1) - Neutron appliance 的準備工作
---

{% capture series %}{% include ovirt-neutron-appliance-and-more.md %}{% endcapture %}
{{ series | markdownify }}

### Neutron appliance 的啟動
大概說一下流程就好，具體的細節可以參考 oVirt wiki 的[這篇文章](http://www.ovirt.org/Features/NeutronVirtualAppliance)。

首先，從 glance.ovirt.org 上導入 neutron-appliance 的鏡像，當然你也可以自己編譯鏡像然後放到本地的一個 glance 裏面，這是我們上文說過的 [glance 集成](http://www.ovirt.org/Features/Glance_Integration)的功能。如果你的網絡連接情況不是很理想的話，則請做好心理准備這一步驟可能需要花費一定的時間。至于導入成模板還是單獨一台虛擬機，那就看具體的需求了，如果有多次部署的需要，則選擇導入成模板。這裏我只為了做測試用，所以直接導入成為一台虛擬機了。當然，我有需要的話也可以做一定的修改之後再將虛擬機制作成模板，靈活程度還是很高的。

好在上面的導入過程并不影響我們進行其它的準備工作，我們可以繼續下去。值得注意的是，從 oVirt 社區的 glance 上導入的 neutron appliance 默認是使用 vlan 作為租戶（這裏為了方便我沿用 OpenStack 中的說法）網絡類型的，我們接下來的討論也先以如何使得 neutron appliance 與這種方式進行協調工作來繼續下去。關于如何調整參數以使用 vlan 以外的方式，我們下部分會討論到；關于如何制作一個默認直接使用非 vlan 類型的 neutron appliance，我們下下部分會討論到。

在 oVirt 中添加一個[邏輯網絡]()到對應的數據中心，并且調整其[網卡配置集](http://www.ovirt.org/Features/Vnic_Profiles)的參數使得其能夠使用 macspoof 的 vdsm hook，具體可以看[這裏](https://github.com/oVirt/vdsm/tree/master/vdsm_hooks/macspoof)。

> Q: what is this hook and why do we use it?
>
> A: 在 oVirt 中，為了安全性，默認使用 ebtables 將從某個網卡出來的非特定源 MAC 地址的包全部過濾掉了，使用了 [libvirt 中的 nwfilter 的功能](http://libvirt.org/formatnwfilter.html)。我們知道在 neutron 節點中，qrouter-\* 的 namespace 中的內網路由虛擬網卡 IP 地址所對應的 MAC 地址，以及 qdhcp-\* 的 namespace 中用以作為 dhcp 服務器的虛擬網卡 IP 地址所對應的 MAC 地址都有可能和用以連接其它節點網卡的 MAC 地址不相同，且從上面兩個虛擬網卡發出來的網絡包有必要經過該網卡去到其它機器，所以我們要把這個 anti-mac-spoofing 的功能給關閉掉以免將這些有效的網絡包過濾掉。

第一步中的 neutron appliance 導入完成之後，我們接著對它做如下操作：確認該虛擬機有兩塊網卡（如果沒有則添加），然後我們讓第一個網卡連接至 oVirt 默認的 ovirtmgmt 網絡，第二個網卡連接至我們上面新建的邏輯網絡，使用剛才配置的網卡配置集。第一個網卡連接至 ovirtmgmt 邏輯網絡，是為了使得 oVirt Engine 能夠訪問 appliance 上面的 Neutron server 提供的 API，如果你的 oVirt 環境中使用的是其它邏輯網絡來提供數據接入你也可以使用其它邏輯網絡來連接 appliance 的第一個網卡，只要確保 oVirt Engine 能夠訪問得到該 appliance 即可。當然，通常最保險的做法還是連接至 ovirtmgmt 邏輯網絡，使得該 Neutron server 位于 oVirt 的管理網絡之中。有心的讀者可能會發現，我們少了連通租戶網絡以及外網的一個接口，這個我們留到下面修改配置的部分來討論。

然後選擇編輯虛擬機，我們使用 [cloud-init](http://www.ovirt.org/Features/Cloud-Init_Integration) 來進行一些基礎的配置。該功能在 oVirt 中選擇編輯虛擬機即可使用。我們新建一個作為 sudoer 的用戶（appliance 中的 root 用戶默認被鎖），然後為該用戶設置密碼或者添加 SSH 公鑰到虛擬機中的 authorized_keys 文件之中。然後對于第一個網卡，亦即連接至 ovirtmgmt 的網卡，我們給它設置一個屬于 oVirt 管理網絡之中的靜態的 IP 地址并設置為開機啟動，當然如果環境中用 DHCP 并且能為該虛擬機一直保留一個固定 IP 的話，也可以選擇使用 DHCP。對于第二個網卡，即連接至我們用以連接租戶的邏輯網絡那個網卡，我們簡單設置為開機啟動，而不設置任何 IP 信息（我們并不需要通過該網卡訪問該 appliance）。

我們就要啟動這台 neutron 虛擬機啦。請確保用以運行這台虛擬機的主機上安裝了 vdsm 的 macspoof hook，如果沒有的話可以運行 `yum -y install vdsm-hook-macspoof` 來安裝。然後就點擊運行虛擬機，稍等一段時間之後，就可以看到該虛擬機的 IP 地址在 oVirt Engine 中顯示出來（由于 appliance 中安裝了 ovirt-guest-agent）。

虛擬機啟動之後，我們就可以通過剛才設置的 SSH 信息連接到該虛擬機裏面了。執行 `sudo -i` 獲得一個 root shell。執行 `source /root/keystonerc_admin` 可以完成一些環境變量的設置以便執行 neutron 相關的命令。執行 `openstack-status` 可以看到相關服務的狀態。

**強烈建議重新設置 neutron 服務的密碼，執行 `keystone user-password-update neutron` 并重啟 neutron 服務即可。**

至此，我們對這台虛擬機做的相關操作就完成了。

### 在 oVirt Engine 中使用 Neutron appliance 中提供的 Neutron 服務作為外部網絡提供程序

首先在運行 oVirt Engine 的主機上執行 `engine-config -s KeystoneAuthUrl=http://NEUTRON_SERVER_IP_ADDRESS:35357/v2.0/` 并重啟 oVirt Engine 服務。NEUTRON_SERVER_IP_ADDRESS 即是我們之前給 neutron appliance 虛擬機設置的地址。

然後訪問 oVirt Engine 的 web 界面，添加一個外部供應商，類型選擇 `OpenStack Network`，插件選擇 `Open vSwtich`，地址填 `http://NEUTRON_SERVER_IP_ADDRESS:9696`，用戶名為 `neutron`，密碼為我們之前設置的密碼，租戶名稱為 `services`，然後可以點擊下該對話框的“測試”按鈕確認是否能夠正常連接至 neutron server。然後在代理配置的頁面中，將橋接映射設置為 `vmnet:br-之前添加的邏輯網絡名稱`，消息隊列服務的 Broker 類型選擇 `RabbitMQ`，地址依然是 `NEUTRON_SERVER_IP_ADDRESS`，端口為 `5672`，用戶名和密碼都為 `guest`。以上這些設置，了解 OpenStack 的人應該都清楚，只是一些最簡單的設置而已。

然後確定添加該外部網絡供應商即可。至此，我們完成了 oVirt Engine 上使用該 Neutron appliance 提供的 Neutron 服務的所需步驟。

### 添加一台能夠使用該網絡供應商的主機

首先需要確認主機有兩個網卡，一個用于 ovirtmgmt，一個用于連接剛才建立的 neutron 網絡。需要注意的還有，如果使用 vlan 方式的話，需要保証用以連接 neutron 網絡的網卡，和運行著 neutron appliance 的主機上對應的網卡有物理的連通，并且交換機上面做了相應的配置以允許帶 vlan tag 的網絡包通過，這些都屬于 Neutron 的基本配置，不贅述。

在該主機上執行 `yum install -y http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-3.noarch.rpm` 以添加 rdo icehouse 的源。然後在 oVirt Engine 的界面上添加主機，在添加主機時配置主機上的網絡供應商，選擇我們之前添加的那個，并設置橋接映射為之前設置的內容，然後就是開始添加該主機的過程了。

主機添加完成之後，在 oVirt Engine 界面上的設置主機網絡部分，配置我們第一步創建的那個用以連接租戶網絡的邏輯網絡，將其與對應的網卡連接起來。然後到該主機上執行 `ovs-vsctl add-port br-internal internal`(以該邏輯網絡的名稱為 internal 為例)。
> Q: 上面執行的命令是什麽意思？
>
> A: 在 neutron 中，neutron agent 會自動配置好橋接，也就是說在 neutron appliance 中和我們剛才新建的主機之中，都存在一個對應于 vmnet 的虛擬交換（這個映射是我們上面在 oVirt Engine 上設置的）。在 neutron appliance 中，這個設備名稱為 br-eth1，連接的是運行該 neutron appliance 虛擬機的主機上用以租戶網絡的網卡。而在我們剛新建的主機中，該橋接名稱為 br-internal，沒有連接接到對應的網卡，該網卡在 oVirt 中會歸屬到以對應地邏輯網絡名稱為名的橋接下，因此為了讓兩個 br-internal 能夠成功的連接到一起，我們需要執行以上操作，將 internal 這個虛擬網卡設備橋接至 br-internal 橋中，類比兩個交換機進行級聯。請看下面的圖示。

### 此時的網絡連接情況

我們來看圖說話，如果圖片超過一屏幕請進行適當的縮放，或者考慮更換高分辨率顯示器。

                                                                          oVirt Engine
                                                                                ^
    +----------------------------------------------------------+                |
    |                Host                                      |                |                 +--------------------------------------------------------------+
    |                                                          |           +----v-----+           |                       Host                                   |
    |  +------------------------------+     +---------+   +----+           |  Switch  |           +----+  +---------+                          +---------------+ |
    |  | Neutron appliance VM    +----+     | linux   <--->eth0<----------->  <---    <----------->eth0<--> linux   |                          | Network Stack | |
    |  |   Network Stack <------->eth0<-----> bridge  |   +----+           |    --->  |           +----+  | bridge  |                          |       ^       | |
    |  |     ^    ^              +----+     +=========+        |           +----------+           |       +=========+                          |    +--v--+    | |
    |  |     |    |  +--------+       |     |ovirtmgmt|        |                                  |       |ovirtmgmt|                          |VM  |eth0 |    | |
    |  |     |    +--> br-int |       |     +----^----+        |                                  |       +----^----+                          +----+--^--+----+ |
    |  |     |       +========+       |          |             |                                  |            |                 +--------+            |         |
    |  |     |       | Open   |       |          |             |                                  |            |    +------------> br-int |            |         |
    |  |+----v---+   |vSwitch |       |          |             |                                  |            |    |            +========+            |         |
    |  ||br-eth1 |   +----^---+       |          v             |                                  |            v    v            | Open   <------------+         |
    |  |+========+        |           |  Host Network Stack    |                                  |  Host Network Stack<------+  |vSwitch |                      |
    |  || Open   |      +-v---------+ |          ^             |                                  |                           |  +----^---+                      |
    |  ||vSwitch <--+   |int-br-eth1| |          |             |                                  |            X              |       |                          |
    |  |+--^-----+  |   +-* veth *--+ |          |             |                                  |            ^              |       |                          |
    |  |   |        +--->phy-br-eth1| |          |             |                                  |            +---------+    |       +------------+             |
    |  |+--v-+          +-----------+ |          |             |                                  |            |         |    |                    |             |
    |  ||eth1|                        |    +-----v----+        |                                  |        +---v------+  |  +-v----------+         |             |
    |  ++^---+------------------------+    | internal |        |           +----------+           |        | internal |  |  | br-internal|     +---v-----------+ |
    |    |                                 +==========+   +----+           |  Switch  |           +----+   +==========+  |  +============+     |int-br-internal| |
    |    +--------------------------------->  linux   <--->eth1<----------->  <---    <----------->eth1<--->  linux   |  |  |   Open     |     +---* veth *----+ |
    |                                      |  bridge  |   +----+           |    --->  |           +----+   |  bridge  |  +-->  vSwitch   <----->phy-br-internal| |
    |                                      +----------+        |           +----------+           |        +----------+     +------------+     +---------------+ |
    |                                                          |                                  +--------------------------------------------------------------+
    +----------------------------------------------------------+

預設我們用以連接 neutron 租戶網絡的 oVirt 邏輯網絡的名稱為 internal，oVirt 的管理網絡為默認的 ovirtmgmt。

首先說上節最後的問題，觀察上圖右邊的 Host 部分，原本，名為 internal 的 bridge 設備上的 internal 端口是直接連接至主機的網絡棧的，而我們上面運行的命令，是將該 internal 端口連接至 br-internal 的虛擬交換上，從而使得兩個對應于 neutron 的 vmnet 的虛擬交換連接在一起；從而，兩邊兩個 br-int 也連接到一起。

上圖中有幾個主要值得注意的地方。我將橋接畫成了一個框，橋接與其上對應于主機網絡棧的設備之間用一條雙橫線(`===`)來分隔。我們簡單將橋接當作一個二層的交換機，同樣的做法對待 OpenvSwitch 的虛擬交換。當在主機中新建一個虛擬交換的同時，該虛擬交換上便會有一個接口連接至主機的網絡棧。如果將主機上的某個網卡連接至虛擬交換上，則此時主機的網絡棧與該網卡已經**毫無關係**，主機網絡棧通過一個虛擬的接口連接至該虛擬交換/網橋，通常這個接口的名稱會和網橋同名所以容易混淆。這點上，我覺得 OpenvSwitch 的 internal port 的概念比起 linux bridge 的好理解很多。這也是為什麽當橋接了之後，不應該在原來的網卡上設置 IP 地址的原因，因為此時該網卡和主機的網絡棧毫無關係，我們此時應該將主機的 IP 地址設置在連接至該虛擬交換的接口上，即通常和該網橋同名的設備上。簡而言之，**IP 地址應該被設置在直接連接網絡棧的設備之上**，只要記得這條規則就好了。

另一個值得注意的地方是 veth，即圖中中間的橫線中有 `* veth *` 的組件。我在之前[介紹 lxc 的文章](./2014-11-17-first-impression-of-lxc.html)中說過 veth 和 OpenvSwitch 的 patch port 其實是差不多的東西，有空再寫 blog 討論這個好了。其實，就是一個雙向的管道。

通過觀察上圖中間上方的交換機以及連接情況，我們可以知道，oVirt Engine 通過訪問這個網絡同時訪問主機的網絡棧（vdsm 在此監聽），以及 Neutron appliance 虛擬機中的網絡棧（Neutron Server 在此監聽）。從網絡的角度上看，圖中只有幾個 `Network Stack` 部分屬于具體的機器（物理機/虛擬機），其他部分，都可以認為是不屬于主機的網絡設備，只不過有物理和虛擬化的區別而已。

關于圖中右上角的 VM，事實上我們此時還沒有啟動它，暫時我們知道啟動虛擬機之後是這樣連接的就可以了。在此 VM 中，eth0 連接至此台虛擬機的網絡棧（假設該虛擬機中沒有橋接）。

總結一句話，連同 `oVirt Engine` 表示的機器在內，圖中總共有 5 台網絡上的主機，其它的，都是連接設備。

### 結束語

到這裏，Neutron appliance 啟動完畢，準備工作也已經就緒。我們準備進入下一部分，關于具體如何在 oVirt 中使用該 Neutron 服務以及連接至其上面定義的網絡，我們還會講到如何調整默認的 Neutron appliance 以及相關的參數，將租戶的網絡類型從 vlan 改成使用 gre 隧道。
