---
layout: post
title: arp_ignore 和 arp_filter
---

先說結論好了

  1. `arp_filter`有個表哥叫做`rp_filter`，這裏的`rp`是`Reserve Path`的意思，其實就是用來檢查返回的包是否會從相對應的到來的包使用相同的網絡接口出去。它們的區別只是層次不同
  2. `arp_ignore` 則是內核用來確定是否應該回覆從該端口收到的`ARP`請求的
  3. 這兩個判斷都在內核中的`net/ipv4/arp.c`中

故事是因爲有個同事在一個機器上用`libvirt`搭建了幾臺虛擬機之後使用橋接連接這幾臺機器，並且使用自動部署工具去部署這幾臺機器。結果發現其中有一臺機器的一個網卡沒有成功得到預期的地址。調查之後發現那臺機器在配置該地址之前使用了`arping -D`去檢查是否與其他機器已有的地址相互衝突了，剛巧該地址和宿主機之上的另外一個網卡地址相同，於是宿主機上橋接虛擬機網絡的網卡，便響應了那個`ARP`請求，導致配置失敗。

一開始我也很奇怪，爲什麼明明宿主機上橋接着虛擬機網絡的網卡並不擁有那個目標地址但卻會回覆，而更奇怪的是，虛擬機中使用`arping`不加`-D`參數，也收不到回覆。所以經過一番搜索與測試驗證，才大致搞明白了其中的原因。

首先，在內核文檔中有這樣一段對`arp_filter`參數的描述：

    0 - (default) The kernel can respond to arp requests with addresses
    from other interfaces. This may seem wrong but it usually makes
    sense, because it increases the chance of successful communication.
    IP addresses are owned by the complete host on Linux, not by
    particular interfaces. Only for more complex setups like load-
    balancing, does this behaviour cause problems.

內核認爲一個IP地址是屬於整個主機的，而非某個特定的端口，所以默認情況下，每個端口都會回覆目標是其他端口的IP地址的ARP請求。

查看了機器之後發現這一項爲默認值0，但是又一個疑問發生了，爲什麼使用`arping`不帶`-D`參數就無法收到返回呢？答案是`rp_filter`做了過濾，默認回應的包不從收到的端口出去，所以並不回覆。

那麼接下來又有一個問題，爲什麼帶了`-D`參數就收的到回覆呢？原因是，如果使用`arping -D`的話，發出的請求包的原地址是設置爲`0.0.0.0`的，這點可以參考[RFC2131](https://tools.ietf.org/html/rfc2131)中的`4.4.1`一段。然後來看代碼：

```
    /* Special case: IPv4 duplicate address detection packet (RFC2131) */
    if (sip == 0) {
            if (arp->ar_op == htons(ARPOP_REQUEST) &&
                inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL &&
                !arp_ignore(in_dev, sip, tip))
                    arp_send_dst(ARPOP_REPLY, ETH_P_ARP, sip, dev, tip,
                                    sha, dev->dev_addr, sha, reply_dst);
            goto out;
    }
```

當請求包的地址爲`0.0.0.0`的時候，內核只照顧`arp_ignore`這個選項，而不去管`arp_filter`選項的內容，也不去管`rp_filter`的內容。

下面看兩個場景，加深一下對這幾個參數的印象：

1. 主機上兩個接口，地址分別處於不同的子網之中
  * 這種情況不會引起混亂
  * 從哪個端口進來的`ARP`請求，一般來說請求的源地址也是那個子網之中的地址，因此回覆包也會從該端口出去，所以能通過`rp_filter`以及`arp_filter`的驗證
  * 這種情況下的回覆只需要考慮`arp_ignore`的影響
2. 主機上兩個接口，地址處於相同的子網之中
  * 容易引起混亂的情況
  * 兩個端口進來的`ARP`請求，基本上源地址是同一個子網，因此回覆的包只會從`ifindex`較小（猜測，未驗證）的端口發送出去，因此從其中一個端口進來的請求有可能無法通過`rp_filter`以及`arp_filter`的驗證
  * 同樣需要考慮`arp_ignore`的影響，對於上面說的收到無法通過`rp_filter`以及`arp_filter`驗證的請求包的端口，除非請求的源地址爲`0.0.0.0`，否則便根本不會回覆，而另一端口收到的則正常，而且對兩個接口上所設置的地址，都能夠通過上述的驗證
  * 要想讓兩個端口在同一子網中互不干擾，各自使用各自的地址並各自對請求包進行回覆，則需要使用策略路由，並且將`arp_filter`和`rp_filter`打開，`arp_ignore`正確設置

關於這些選項的具體設置值的意義，可以參考內核文檔，這裏就不複製粘貼進來加長篇幅了。總之，在能夠進行規劃的情況下，還是儘量避免容易引起混亂的情況爲好。
