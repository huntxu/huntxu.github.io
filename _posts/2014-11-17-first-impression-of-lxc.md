---
layout: post
title: LXC 初體驗
---

還沒有看過代碼，簡單地跟著教程做了一些測試而已，所以大概記錄下初體驗的感覺吧。

聽說 LXC 已經很久很久了。當然其實沒有那麽久，但是到現在確實已經發展到一定程度了，現在才開始接觸，不得不說已經是有些晚了，再加上年紀又慢慢地長了，有些擔心跟不上時代的變化和技術的發展了。不過，聖人倒是說“朝聞道，夕死可以”，這麽看起來，其實也還不算晚，慢慢趕上就是了。

首先推薦下[這個網站](http://www.flockport.com/)，基本上我把上面的文章差不多都看完了，而且是很細很細地讀，當然，這麽做也花了不少時間。這個站點上的文章特別之處在于，除了一些教程式的或者技術參考式的小品文章之外，還有一些從大局上分析 LXC 與其接近的解決方案之間的關聯和區別，以及各自的，~~優缺點~~不好說我就不說了，畢竟各人的評判標準不同，統一叫做特點好了。至少在我認為，在風風火火的概念炒作之後，有必要停下來仔細想想每日頭條上的東西究竟適不適合自身的情況，而不是看誰嗓門大吆喝聲響亮就投奔誰。就算賺到了一時的眼球之後又該何去何從？這是一些關于 LXC、虛擬機、docker 選擇上的吐槽，同樣吐槽的還有對 scale up/out 本質的理解。關于後者，似乎不少觀點會認為把兩個 1 堆到一起反正就變成 2 了，殊不知，關鍵的，是中間那個 + 號。

---

華麗的分割線以後，以下是正文部分。

關于基本操作沒什麽好說的，堆安裝教程什麽的不是我的風格，都有空來逛 blog 了自然更有空去找官方的 tutorial，少我一篇復制來的基本教程不少，多我一篇可就多了。既然沒什麽坑的話，就不浪費時間了。所以以下就零零散散地記錄一些個人認為比較有意思的或者值得一些關注的點好了。

### 基本操作
就 lxc-* 幾個命令，--help 的幫助寫得很清晰，基本使用起來無難度，常用的命令也就那麽幾個，多打幾遍就熟悉了。

幾種接入方法，lxc-console，lxc-attach，或者 ssh，各有各自特點，一個是終端登入，一個是直接連接并可以直接運行命令，一個是通過網絡。

關于模板和 container 的創建，官方提供的包帶的那些系統的模板已經差不多了，有特殊需求的話再說，模板文件也許值得好好研究下，通過腳本構建一個最基本的系統。

### 關于網絡
沒什麽說的，橋接或者 NAT，萬變不離其宗，後者一般跑個 dnsmasq 來做 DHCP 服務器，這貨真心是殺人防火打家劫舍必備，又快又好。

至于和 host 的連接，veth 也行，直接 pass-through 也好，都是那麽些事，和虛擬機的應用沒啥不同，不過是 veth 對應虛擬化網卡而已。不過我倒是記得 OpenvSwitch 可以有 internal port，直接把那個端口放到某個 namespace 裏面就能直接給某個 container 用了，似乎 OpenStack 裏面見到很多類似做法。本質上 veth 和 ovs 的 internal port 是一路貨色，從該端口接收是從某個網絡棧復制內容到底層，發送至該端口是把底層數據交給某網絡棧（在多 namespace 的情況下網絡棧有多個，這個兩三年前用 mininet 模擬好多好多機器的網絡時已經有過了解），考慮寫另外一篇 blog 來介紹下。

接著就是各種隧道和 VPN 了，gre 也就那麽用，vxlan 拿來哄哄外行還可以（外圍很多東西都還沒有跟上，軟硬件都是），flockport 上面介紹的 tinc 沒用過，有必要的時候再試試好了。

### 關于存儲
應該很少需要和存儲打交道，各種 backend 的介紹網上一搜一大把，慢慢實驗再結合具體情況才能確定用哪個，很多都是要針對不同問題分析。

再者，存儲的 backend 問題只是一部分的管理問題而已。上面說過的，沒有加號多少個 1 堆在一起都不會變成 2。Scale-out 的情形裏面，如何處理 container 自身以外的狀態數據才是重點。這點上，各種存儲什麽的還能接著用，bind mount 可以同時挂載給多個 container 使用，集群文件系統什麽的可能在這個上面可以做做文章，畢竟有需要鎖的時候。

當然，如果是 PaaS 的話，管什麽 Scale-out 呢，本來 PaaS 就是要很多個并列的 1 而已，這種情況下倒值得好好研究下各種 backend。

### 關于其它
Nginx 在 host 上反代，可以實現一部分的負載均衡/高可用功能。HAProxy 還沒怎麽接觸，有空再試試，OpenStack 的 LBaaS 炒得那麽天花亂墜其實也就是 HAProxy。其實我個人是不太喜歡這種哲學的，把很貼近應用層面的東西放到 neutron 這個所謂網絡模塊裏面。

當然，反代還可以解決那些在公共 IP 資源有限的情況下需要把 container 中的服務開放出去的問題。

VRRP 式的解決方案，有 keepalived/vrrpd 可以選，基本能滿足需要了。
