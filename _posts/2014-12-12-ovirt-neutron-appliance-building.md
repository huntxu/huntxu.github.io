---
layout: post
title: 從 oVirt 的 Neutron appliance 開始(3) - Neutron appliance 的構建
---

{% capture series %}{% include ovirt-neutron-appliance-and-more.md %}{% endcapture %}
{{ series | markdownify }}

前面的文章中我介紹了 oVirt 的 Neutron appliance 使用的準備工作和在 oVirt 中如何集成。這篇文章中我將介紹構建這個 Neutron appliance 的過程，以及如何進行一定的修改定制使得其在環境中更加合適，減少部署之後額外的修改配置步驟。

這篇文章的內容和本系列主題的關聯性比較小，只不過既然我們是從 oVirt Neutron appliance 開始的，還是需要了解一下這樣的一個 appliance 是如何來的，作為背景知識在這裏介紹就好了。多了解一些總是不會很差的。

相對這之前以及之後的內容，這部分是我掌握得比較薄弱的部分，所以只能夠大概說明操作的過程，不會對使用的工具有很深入的介紹。好在，在我看來，相關工具的使用還是比較簡單的。

大體上講，要構建一個 appliance，主要分成兩步。第一是制作一個鏡像，裏面包含有相應的服務程序，并且對這些服務進行了初始的配置。第二是進行封裝，即是去掉制作過程中與本地環境相關的內容，使得其它另外的環境中能夠重新進行配置然後工作。

關于如何制作 Neutron appliance，請看[這篇上游說明](http://gerrit.ovirt.org/gitweb?p=ovirt-appliance.git;a=blob;f=neutron-appliance/README.md;h=260e6987762a0b4baf97710ad08e6f79bbe081e2;hb=HEAD)。源代碼倉庫在[這裏](http://gerrit.ovirt.org/gitweb?p=ovirt-appliance.git)。

制作過程中的兩個步驟，分別對應使用了 [imagefactory](http://imgfac.org/) 和 [virt-sysprep](http://libguestfs.org/virt-sysprep.1.html)，有需要的話，請參閱這兩個工具的網站了解相關信息。

### 制作鏡像
```
imagefactory target_image --template PATH/TO/rdo-icehouse-centos-7-ml2-plugin.tdl APPLIANCE_NAME
```

`imagefactory` 是命令，我們看後面的內容。

根據[這個用戶手冊](http://imgfac.org/documentation/imagefactory.html)，最後這部分 `APPLIANCE_NAME` 是最終目標的名字，`target_image` 是 `imagefactory` 的一個子命令，用以生成一個鏡像，根據 `--template` 參數指定的模板文件。這個模板文件遵循[這個描述](http://imgfac.org/documentation/tdl/TDL.html)。

也就是說，所有的關鍵內容都在我們上面舉例的這個 `rdo-icehouse-centos-7-ml2-plugin.tdl` 裏面了。對于懶得去 git clone 源代碼下來看的人，點[這裏](http://gerrit.ovirt.org/gitweb?p=ovirt-appliance.git;a=blob;f=neutron-appliance/rdo-icehouse-centos-7-ml2-plugin.tdl;h=8a2ff255df8c249d98d109952711f903f6b3af47;hb=master)。

這個文件裏面也幾乎沒什麽特別的內容，定義了一些鏡像的基本信息，然後執行相關的命令，基本沒什麽好說的。裏面用到了 [packstack](https://wiki.openstack.org/wiki/Packstack)，無非就是生成一個應答文件，然後根據這個應答文件進行 OpenStack 的自動化部署。所以裏面配置的部分，主要就是對 answerfile 進行處理，對需要的服務進行配置，不需要的服務則選擇不部署在這個鏡像之中。注意其中設置 ml2 插件的幾個項目(`CONFIG_NEUTRON_ML2_`)，其實內容就和我們[上一篇文章](./2014-12-03-ovirt-neutron-appliance-usage.html)中在 Neutron appliance 中進行修改的部分的原文是一致的，也就是其實我們就只是要對應的在這個地方添加或者刪除相應的設置項即可。至于具體是哪些項目需要修改，參考下 Packstack 相關的文檔好了，不在這裏贅述。

當然，我們這裏改的只是 appliance 之中的設置，主機的設置一樣是需要安裝之後進行一定的修改的。當然，還可以考慮使用 puppet 這類配置管理工具，但是不在本系列主題的內容之內了，有興趣者可以自行研究。

### 進行封裝
```
virt-sysprep --add PATH_TO_CREATED_IMAGE --enable net-hwaddr,dhcp-client-state,ssh-hostkeys,ssh-userdir,udev-persistent-net
```

virt-sysprep 根據其[官網](http://libguestfs.org/virt-sysprep.1.html)上的解釋，是一個用于重置/刪除一個虛擬機的相關配置使其回復初始的樣子以使得其能夠被克隆并使用的工具。這個和 Windows 上的 Sysprep 功能應該是差不多的，可以看[這篇文章](http://technet.microsoft.com/zh-cn/library/cc721940%28v%3Dws.10%29.aspx)，以及[這篇文章](http://support.microsoft.com/kb/302577/zh-cn)。

這個更不需要怎麽解釋了，`--enable` 參數後面帶的是選擇的[操作](http://libguestfs.org/virt-sysprep.1.html#operations)，在這裏是 `net-hwaddr` 刪除掉網絡配置文件中的 MAC 地址信息，`dhcp-client-state` 是刪除 DHCP 客戶端的租約（假設部署過程中使用到了 DHCP 的話），`ssh-hostkeys` 是刪掉鏡像中的 SSH host key，`ssh-userdir` 是刪掉鏡像中所有用戶目錄下的 `.ssh` 目錄（如果有的話），`udev-persistent-net` 刪除將某個 MAC 地址的網卡映射到特定設備名稱的 udev 規則（否則 MAC 地址在鏡像被重新使用之後很可能更改，然後虛擬機裏面的設備名稱就可能發生變化）。其它更多內容，則請參考文檔。

經過這樣一個步驟之後，鏡像就封裝完成。然後我們就可以像使用從 oVirt 的 glance 上導入的 appliance 一樣，使用我們自己制作的這個 appliance 了。當然，要導入到 oVirt 之中，你就需要先有一個 glance 服務，把我們制作的這個鏡像上傳到那個 glance 服務中，然後和我們之前的操作一樣，在 oVirt 界面上從這個 glance 服務裏面導入就可以了。

### 結束語
本篇的內容由于沒怎麽涉及底層，因此只是簡單的介紹，所以相對來講比較簡單。但是其實涉及的相關的 Linux 知識也是蠻多的，主要是系統的安裝及配置部分的內容。

對于所用到的兩個工具，有興趣的同學可以更深入的進行研究了解其細節。我覺得還是有很多值得研究的地方的，比如說 imagefactory，可以研究如何編寫 tdl 文件，如何很快的創建所需要的鏡像，如何將常見的軟件 appliance 化等等，都是值得探討的問題。

最後做一下預告，此系列從下一篇文章開始，將進入 Neutron 的介紹部分。首先會是 Neutron 中核心的網絡和路由功能的簡單介紹及說明。
