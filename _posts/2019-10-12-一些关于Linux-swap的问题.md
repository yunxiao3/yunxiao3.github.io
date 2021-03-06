---
title: 一点关于Swap配置对性能影响的思考
description:
date:  Sat, 12 Oct 2019 19:32:26 
categories: 
- Linux
---

之前在学Linux的时候就了解过，Swap分区是在Linux系统的物理内存不够用的时候，把硬盘空间中的一部分空间释放出来，以供当前运行的程序使用的一种机制，类似于windows的虚拟内存的作用。但是一直停留在对这个概念的了解上面，并没有接触或者感触到Swap对系统性能的影响。

直到前段时间在用无服务器下载Linux内核的源码时出现了Out of memory的错误，因为之前使用阿里云相同内存的服务器并没有出现这种情况，所以在排查了一遍以后才发现自己的服务器没有设置Swap分区，导致1G的内存不够用。在将Swap调整到1G后运行正常。为什么不将内存设置的更大些呢？原因很简单因为Swap分区的内存是驻留在硬盘上的，而读写硬盘要比直接使用真实内存慢得多(要慢数千倍，即使是SSD也会慢很多后文会提到)设置过大的Swap内存可能会让系统性能变得异常糟糕。在用Swap解决了内存不足的问题后，我又发现自己的电脑在CPU利用率不高的情况下依旧很卡，之前一直以为是CPU性能跟不上，但仔细排查以后发现自己 **/proc/sys/vm/swappiness**中的值是默认的60(swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间，swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面。)，这导致在内存充足的情况下Swap分区的利用率比内存还要高。而这极大的拉低了我系统的流畅性。在将swappiness的值设置为5以后自己的电脑无论是在开机还是运行过程都比原来感觉快了许多。

查看并修改swappiness的命令：

~~~shell
sudo cat /proc/sys/vm/swappiness
sudo sysctl -w vm.swappiness=5
~~~

电脑重启以后设置失效，想永久修改swappiness可以在 /etc/sysctl.conf中添加vm.swappiness=5。

当然Swap还有很多东西需要去学习和了解，在面对不同的应用时和不同的场景下Swap大小和swappiness的设置都有着很多学问，在此只是记录一下自己对Swap分区和系统性能之间的一点点思考。