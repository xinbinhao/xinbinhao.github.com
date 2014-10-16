---
title: "TCP系列二:主动关闭连接"
layout: post-my
guid: urn:uuid:iw2e47f0q407ch4wu1kdd07jjjnn252s
description: "tcp连接时一端主动断开连接原理解析"
tags:
  - tcp shutdown
---

在编写网络通信模块时，经常会越到主动断开连接和被动断开连接的情况，针对主动断开连接时，要执行4次握手过程。而非正常断开连接有多种，比如说 连接的一端执行kill -9时，另一端需要针对 -1 状态码做处理，然后调用close主动断开连接。还有一种情况是链接的一端宕机(拔网线、掉电)，这种情况请参考  [TCP系列一:宕机](http://polimo.github.io/2014/08/25/tcp-poweroff.html “TCP系列一:宕机”)

今天主要说的是主动关闭连接情况。关闭4次握手过程如下图：

![tcp shutdown](/media/files/2014/09/23/tcp-shutdown.jpg)

假如连接的两端分别为客户端(C)和服务端(S)。

一、C/S 执行close 
客户端(C)或者服务端(S)一端执行关闭时，另一端会捕获到 -1 状态码，当获取到 -1 状态码时应该主动关闭，否则当前连接为坏连接。

看到这点的时候很多人认为很简单，但在工作中发现有业务线没有针对-1做处理，很容易引发别的问题。

二、执行close 底层发生了什么

按照上图中主动关闭状态图看，如果一端主动执行close时应该发送 fin 包，但真的是这样吗？

在做网络框架开发时发现一条tcp链接在close时，通过抓包看close在系统调用的时候，发送端发出的是rst报文, 而不是正常的fin而另一端会收到econnrest,而不是正常的fin包。如下：


> 17:26:09.521258 IP localhost.13600 > x.x.58.86.http: R 19:19(0) ack 523 win 9509


可以清楚的看到 R 19:19(0) ack 523 win 9509,发的是rst包,为什么呢？

看看代码实现了什么：

{% highlight java %}

net/ipv4/tcp.c

/* As outlined in RFC 2525, section 2.17, we send a RST here because
      * data was lost. To witness the awful effects of the old behavior of
      * always doing a FIN, run an older 2.1.x kernel or 2.0.x, start a bulk
      * GET in an FTP client, suspend the process, wait for the client to
      * advertise a zero window, then kill -9 the FTP client, wheee...
      * Note: timeout is always zero in such a case.
      */
     if (data_was_unread) {
          /* Unread data was tossed, zap the connection. */
          NET_INC_STATS_USER(sock_net(sk), LINUX_MIB_TCPABORTONCLOSE);
          tcp_set_state(sk, TCP_CLOSE);
          tcp_send_active_reset(sk, sk->sk_allocation);
     }

{% endhighlight%}

从代码可以看出，当接收缓冲区还有数据，协议栈就会发rst代替fin，所以上面为什么会发rst而不是fin了。


在我们程序中可以在调用close时把缓存区的数据手动拉取处理完，这样就可以发送fin包了。


