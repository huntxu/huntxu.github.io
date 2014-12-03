接觸 Neutron 這個東西并不是很多，裏面有很多概念都還沒有很熟悉，很多功能也還沒有實驗過；所以此 blog 也可能存在多多少少的錯誤之處，請批判性閱讀。

能力所限，只能簡單的介紹些皮毛，且文字功底更是一般，只能盡可能把想說的內容說清楚吧。

自 oVirt 3.3 以來，oVirt 提供了 [Neutron 的集成](http://www.ovirt.org/Features/Quantum_Integration)，即是在 oVirt 中直接使用 Neutron 來對網絡進行管理。這樣做主要有兩個好處，第一是避免了重復的工作，沒必要將 Neutron 實現了的功能在 oVirt 中重新給實現一次；第二則是提供了 OpenStack 到 oVirt 的“軟著陸”，使得這兩個東西更有機會也更容易地集成到一起。不過，以目前的狀態來說，在 oVirt 中的 Neutron 集成功能上還是相當的有限，有不少受限制的地方，以後的改進應該不會少。另外插句題外話，在 Neutron(OpenStack Networking) 集成之外，oVirt 中集成的“外部供應商”，即是某服務的非 oVirt 內生的提供者，還包括 Foreman 以及 OpenStack Image(即 glance)兩種類型，有興趣者可以自行查閱相關的內容。

而眾所周知，部署一套簡單的 OpenStack 環境，對于普通用戶，乃至熟悉 OpenStack 的技術人員來說，即使不困難，也會稍顯繁瑣。況且當要部署多個環境的時候，重復性的工作很容易就能磨滅人的創造力以及不斷挑戰的精神，這太不符合今日 plug and play 的一般游戲規則了。所以就有了 appliance 這種東西，簡單講，這個東西就是讓服務變得能夠“即插即用”，或者只需要非常少量的配置就能夠使用，去除了很多機械性的工作重復，從而提高生產力。

有了 Neutron appliance，于是有了我使用這個 appliance 的一些實驗，以及實驗過程中接觸到的其它東西，包括 Neutron 本身，底下的 OpenvSwitch，用到的 tunnel 技術以及隔離方法等等，所以有了系列文章。

從大的結構看，本系列會大致分為三大部分。第一部分是使用以及其它和這個 appliance 本身相關的內容，我們會先從如何準備這個 appliance 開始，到如何調整 oVirt 中與 Neutron 集成的方法以及 Neutron 的簡單使用，再簡單地了解分析一下這樣一個 appliance 生成的過程。第二部分是這個 appliance 中所包含的部分服務，以及相關的一些技術的說明和討論，先從 Neutron 最核心部分的網絡以及路由服務的提供開始介紹，接著講各種租戶網絡的隔離方式的本質如何，然後看看常用的 OpenvSwitch 這個虛擬交換機在 Neutron 中是如何被使用的，~~本來我還想討論下 Neutron 中的 LoadBalancer 和 Firewall 服務以及安全組的，但是苦于對 Neutron 了解不深，怕說得不好，只能等到有更深的了解之後，在另外的文章裏面在討論了~~。最後一部分是各種吐槽和亂七八糟的未成型的想法。

以下是系列文章列表

1. [Neutron appliance 的準備工作](./2014-12-02-ovirt-neutron-appliance-preparing.html)
1. [Neutron appliance 的使用](./2014-12-03-ovirt-neutron-appliance-usage.html)

---

華麗的分割線以後，以下是正文部分。
