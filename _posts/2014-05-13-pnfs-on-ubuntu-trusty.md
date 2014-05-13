---
layout: post
title: 在 Ubuntu 14.04 上搭建 pNFS
---

遍地是坑！

关于 pNFS 的介绍我就不赘述了，可以参考挖坑人的[这篇](http://mathslinux.org/?p=498)介绍。

- - -
我先是走了很多弯路。

据[这篇](http://wiki.linux-nfs.org/wiki/index.php/PNFS_server_projects) wiki 的介绍：

> any work done on gfs2 should also apply to ocfs2 with minimal effort (since they share userland infrastructure).

因为能查到的关于 gfs2 的文档似乎都以 cman 为例，Ubuntu 14.04 上没有 cman 了，于是先尝试了一下 ocfs2，装完之后一切倒都是正常的，就是 pNFS 的 GETDEVINFO 返回的 r_addr 总是 MDS 的地址，不知道为什么，也没有找到相关的文档，于是这个尝试就这样以失败告终。

后来我又打算尝试一下 spNFS，pNFS [开发内核树](http://wiki.linux-nfs.org/wiki/index.php/PNFS_Development_Git_tree)里的最新版本没有[这篇](http://wiki.linux-nfs.org/wiki/index.php/Configuring_pNFS/spnfsd) wiki 里面的内容，最后在 3.1 的分支里面找到，于是切换过去，照常编译内核，重启，进了 initramfs 的提示符，又一次失败告终。

- - -
> That which does not kill us makes us stronger.

没办法了，硬着头皮看文档吧，已经做了最差的准备可能需要在 Ubuntu 上手动编译 cman 什么的东西了，于是先翻了一遍 [Clusters from Scratch](http://clusterlabs.org/doc/en-US/Pacemaker/1.1-crmsh/html/Clusters_from_Scratch/index.html)，然后把 corosync 什么的配置完成了，在之后看了下 gfs2 的 manpages，发现其实什么 cman 啊，pacemaker 啊其实统统都不需要，就只需要把 corosync 配置好集群，然后 gfs2 就能够正常工作了。

于是就照着 manpages 里面的介绍搭建完成，然后尝试把 gfs2 挂载起来，在 MDS 和 DS 上启动 NFS server，然后 client 上挂载，wireshark 走起。这次，终于看到 GETDEVINFO 返回的 r_addr 是 DS 的地址了，读取的操作也确实在 DS 上发生，元数据操作在 MDS 上发生，so far so good！不过还有个问题，写操作的时候 LAYOUTGET 操作返回了 NFS4ERR_BADIOMODE，不知道究竟哪里出问题了，还是等有空再研究吧...测试也不做了，因为 iscsi 的目标是用 iet 虚拟出来的假货，等有环境了继续测试吧。

- - -
以上是大概的流程：

1.  在 MDS 和 DS 的机器上都需要 pNFS 的内核，关于在 Ubuntu 上编译内核，当然推荐 make-kpkg，简直神器，参考自[这里](https://help.ubuntu.com/community/Kernel/Compile)，编译出来 deb 然后 dpkg 装上就好了。内核的配置不说什么了，文件系统选上 pNFS 相关的内容，不知道该不该选就看帮助吧，记得也选上 gfs2：

    ```
    $ fakeroot make-kpkg  --initrd  kernel_image kernel_headers
    ```

2.  同样 MDS 和 DS 都需要 pNFS 的 nfs-utils，这没什么好说的，编译安装就是了，不过要注意的是 Ubuntu 的 nfs-common 软件包已经有相关的命令二进制文件了，先装 nfs-common 然后用[这里](http://wiki.linux-nfs.org/wiki/index.php/PNFS_Setup_Instructions)介绍的方法用 pnfs-nfs-utils 编译出来的二进制覆盖之，注意不要再装 nfs-common 又覆盖回去就好了。关于这个的依赖就慢慢跟着 configure 的报错解决吧，还有个 patch 解决 krb5 的查找的，原因是 Ubuntu(deb 系都这样么？) 上库的位置有些不同：

    ```
    diff --git a/aclocal/kerberos5.m4 b/aclocal/kerberos5.m4
    index dfa5738..bc55d6b 100644
    --- a/aclocal/kerberos5.m4
    +++ b/aclocal/kerberos5.m4
    @@ -36,8 +36,8 @@ AC_DEFUN([AC_KERBEROS_V5],[
        AC_DEFINE_UNQUOTED(KRB5_VERSION, $K5VERS, [Define this as the Kerberos version number])
        if test -f $dir/include/gssapi/gssapi_krb5.h -a \
                    \( -f $dir/lib/libgssapi_krb5.a -o \
    -                   -f $dir/lib64/libgssapi_krb5.a -o \
    -                   -f $dir/lib64/libgssapi_krb5.so -o \
    +                   -f $dir/lib/x86_64-linux-gnu/libgssapi_krb5.a -o \
    +                   -f $dir/lib/x86_64-linux-gnu/libgssapi_krb5.so -o \
                        -f $dir/lib/libgssapi_krb5.so \) ; then
            AC_DEFINE(HAVE_KRB5, 1, [Define this if you have MIT Kerberos libraries])
            KRBDIR="$dir"
    ```

3.  然后在 MDS 和 DS 上一起登录同一个 iSCSI target，这个和本篇主旨没什么关系，参考相关的 iSCSI 文档吧。关于虚拟一个假的 iSCSI target，可以参考[这里](http://www.qyjohn.net/?p=3104)。
4.  在 MDS 和 DS 机器上配置 corosync，把它们放到同一个集群，这里就照着 gfs2 的 manpages 里面写的内容就好了，不贴配置文件了，参考 `man 5 gfs2` 里面的 SETUP 一节中第一步的配置文件，改 IP 地址这种事情就不说了。然后启动 corosync 并查看状态是否正常，同样，是在 MDS 和 DS 上都要完成：

    ```
    $ sudo /etc/init.d/corosync start
    $ sudo /etc/init.d/corosync status
    $ sudo corosync-quorumtool -l
    ```

5.  接下来是启动 dlm，我禁用了 dlm 的 fencing，在 /etc/dlm/dlm.conf 里面填 `enable_fencing=0`。另外 Ubuntu 上的 dlm 脚本没有改好，/etc/init.d/dlm 根本就不能用，于是手动解决，dlm_controld 的参数根据需要设置，不懂请看帮助：

    ```
    $ sudo modprobe dlm
    $ sudo mount -t configfs none /sys/kernel/config/
    $ sudo dlm_controld -f 0 -s 0 -q 0 -K
    $ sudo dlm_tool ls
    ```

6.  格式化一个 gfs2 分区，这个只在一台机器上做就可以了，我直接把 iSCSI target(我这里是 /dev/sdb) 抛出来的整个 LUN 格式化了，集群名是 corosync 配置文件里写的，锁名自己起个：

    ```
    $ sudo mkfs.gfs2 -p lock_dlm -j 2 -t 集群名:锁名 /dev/sdb
    ```

7.  挂载那个 gfs2 分区到 nfs 要抛出的目录。
8.  MDS 和 DS 都启动 nfs server，在 Ubuntu 上，关于 pnfs 的 export 的参数参考[这里](http://wiki.linux-nfs.org/wiki/index.php/PNFS_Setup_Instructions)的 Step 1。

    ```
    $ sudo /etc/init.d/nfs-kernel-server start
    ```

9.  在 MDS 上，运行以下命令，里面的 /dev/sdb（参考文档中说要直接用 sdb，但我的测试好像两者都可以） 是本地 gfs2 分区的位置，DS_IP 是 DS 的 ip，可以一个或者多个，多个的话中间逗号分隔：

    ```
    $ echo "/dev/sdb:DS_IP1[,DS_IP2...]" |sudo tee /proc/fs/nfsd/pnfs_dlm_device
    ```

10.  client 上挂载，client 的内核必须已经有 pNFS client 支持，查看 nfs_layout_nfsv41_files 模块是否可用就行，然后运行：

    ```
    $ sudo mount -t nfs4 -o minorversion=1 MDS_IP:/ 挂载目录
    ```

11.  wireshark/tcpdump 走起，然后分析 pcap 文件，就不贴图了：

    ```
    $ sudo tcpdump -ni eth0 -s0 -w /tmp/capture.pcap "port 2049"
    ```

- - -
以下是参考文档，很多文档都是马马虎虎，和我这篇一样其实，不懂靠猜，猜的准确度靠人品：

* 一篇介绍 pNFS 的 slides: http://snia.org/sites/default/files/Part4-Using_pNFS%20Feb._2013.pdf
* pNFS 的开发 wiki: http://wiki.linux-nfs.org/wiki/index.php/PNFS_Development
* pNFS 配置的过程: http://wiki.linux-nfs.org/wiki/index.php/PNFS_Setup_Instructions
* pNFS 开发的代码 Git 仓库介绍: http://wiki.linux-nfs.org/wiki/index.php/PNFS_Development_Git_tree
* pNFS 的服务器项目，客户端已经被合并到 Linux 内核了: http://wiki.linux-nfs.org/wiki/index.php/PNFS_server_projects
* pNFS + gfs2 的一个例子: http://blog.csdn.net/ycnian/article/details/8523193
* Ubuntu 上编译内核: https://help.ubuntu.com/community/Kernel/Compile
* 在 Ubuntu 上虚拟一个 iSCSI target 用于测试: http://www.qyjohn.net/?p=3104
* cluster from scratch，有些 corosync 之类的介绍: http://clusterlabs.org/doc/en-US/Pacemaker/1.1-crmsh/html/Clusters_from_Scratch/index.html
