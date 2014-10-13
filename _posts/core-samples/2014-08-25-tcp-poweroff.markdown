---
title: "TCP系列一:宕机"
layout: post-my
guid: urn:uuid:866af2a0-3fc7-4ced-839d-1839115b7176
description: "tcp 宕机"
tags:
  - tcp
  - poweroff
  - 宕机
---


**场景**：
客户端与服务端通过tcp长连接的方式进行数据交互，当服务端机器突然宕机(掉电、拔网线等)客户端会发生什么。

**过程**：
> 正常情况下当服务端执行kill -9/-12/-15 客户端会收到-1的数据内容，当客户端收到-1数据信息时，执行关闭socket既可以正常关闭socket。（关闭socket执行4次握手过程）

回到问题本身，为了复现该问题，我们找两台机器进行测试，客户端(x.x.156.57)服务端(x.x.36.14)通过tcp长连接进行数据交互，客户端先发送10条数据到服务端，然后sleep 3分钟，客户端sleep的过程中，关闭服务端(服务端机器不关进程直接执行poweroff模拟宕机情况).客户端sleep 3分钟后，在给服务端发送一条数据。

1、当服务端宕机，在客户端机器执行命令：netstat/ss |grep 服务端端口
会发现当前的tcp连接还是正常状态(为什么？因为tcp长连接正常关闭需要4次握手过程，而针对服务端突然宕机情况是没有正常执行4次握手过程)

> **ss -na|grep x.x.36.14**

> ESTAB      0      0       ::ffff:x.x.156.57:42218   ::ffff:x.x.36.14:16019 

2、服务端关机后一直不重启，客户端过一段时间后会自动断开连接(为什么？时间是否可以控制？)


**tcpdump**

```javascript

17:31:16.236188 IP (tos 0x0, ttl 63, id 38272, offset 0, flags [DF], proto TCP (6), length 142)
x.x.36.14.16019 > x.x.156.57.42215: Flags [P.], cksum 0xcf84 (correct), seq 721:811, ack 919, win 114, options [nop,nop,TS val 665838 ecr 27489167], length 90

17:31:16.275492 IP (tos 0x0, ttl 64, id 41102, offset 0, flags [DF], proto TCP (6), length 52)
x.x.156.57.42215 > x.x.36.14.16019: Flags [.], cksum 0xf944 (correct), ack 811, win 115, options [nop,nop,TS val 27489209 ecr 665838], length 0

17:34:16.239767 IP (tos 0x0, ttl 64, id 41103, offset 0, flags [DF], proto TCP (6), length 154)
x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x51f7), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27669172 ecr 665838], length 102

17:34:16.440506 IP (tos 0x0, ttl 64, id 41104, offset 0, flags [DF], proto TCP (6), length 154)
x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x512d), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27669374 ecr 665838], length 102

17:34:16.844514 IP (tos 0x0, ttl 64, id 41105, offset 0, flags [DF], proto TCP (6), length 154)
x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x4f99), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27669778 ecr 665838], length 102

17:34:17.652631 IP (tos 0x0, ttl 64, id 41106, offset 0, flags [DF], proto TCP (6), length 154)
    x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x4c71), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27670586 ecr 665838], length 102

17:34:19.268528 IP (tos 0x0, ttl 64, id 41107, offset 0, flags [DF], proto TCP (6), length 154)
    x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x4621), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27672202 ecr 665838], length 102

17:34:22.500519 IP (tos 0x0, ttl 64, id 41108, offset 0, flags [DF], proto TCP (6), length 154)
x.x.156.57.42215 > x.x.36.14.16019: Flags [P.], cksum 0xd547 (incorrect -> 0x3981), seq 919:1021, ack 811, win 115, options [nop,nop,TS val 27675434 ecr 665838], length 102

```

> 第1行正常数据发送，第2行返回ack，第三行发送数据，后面5行数据重发。

从tcp底层看当服务端宕机后，客户端看当前连接是正常的，所以客户端还会通过该连接进行数据交互(但其实是传递不到服务端的)，如果发送的数据量很大可以通过
netstat/ss 中得send-q 参数看出(send-q会很大，这里还有个问题如果服务端没有宕，tcp连接也正常，但send-q堆积很大时会发生什么？这个通过另一文章阐述)，

从tcpdump看客户端发送数据后没有返回ack，客户端会重发，重发5次后断开了连接(为什么是5次？)。
这个就要从操作系统tcp参数说起了。

Centos (不同的系统对应参数也不同)
more /proc/sys/net/ipv4/tcp_retries2(默认15) 

tcp_retries2 ：INTEGER
默认值为15
在丢弃激活(已建立通讯状况)的TCP连接之前﹐需要进行多少次重试。默认值为15，根据RTO的值来决定，相当于13-30分钟(RFC1122规定，必须大于100秒).(这个值根据目前的网络设置,可以适当地改小,我的网络内修改为了5)


所以为什么会是重发5次后客户端主动断开连接，客户端主动断开连接其实是会发一个rst的。


ps 

> 至于重发时间间隔可以参考下 **tcp/ip卷 1**



---

