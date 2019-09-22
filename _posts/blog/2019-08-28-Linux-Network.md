---
layout: post
title: Linux 网络
categories: [Linux]
description: Linux 网络
keywords: Linux
---

Linux 网络

---


#### 前言

网络的监测是所有Linux子系统里面最复杂的，有太多的因素在里面，比如：延迟、阻塞、冲突、丢包等，更糟的是与Linux主机相连的路由器、交换机、无线信号都会影响到整体网络并且很难判断是因为Linux网络子系统的问题还是别的设备的问题，增加了监测和判断的复杂度。

网络问题是什么，是不通，还是慢？

1. 如果是网络不通，要定位具体的问题，一般是不断尝试排除不可能故障的地方，最终定位问题根源。一般需要查看

　　　　是否接入到链路

　　　　是否启用了相应的网卡

　　　　本地网络是否连接

　　　　DNS故障

　　　　能否路由到目标主机

　　　　远程端口是否开放

2. 如果是网络速度慢，一般有以下几个方式定位问题源：

　　　　DNS是否是问题的源头

　　　　查看路由过程中哪些节点是瓶颈

　　　　查看带宽的使用情况

#### 实操

##### 是否接入到链路

使用 ethtool 查看 em1 的物理连接

```
[root@hahaha ~]# ethtool em1
Settings for em1:
        Supported ports: [ FIBRE ]
        Supported link modes:   1000baseT/Full 
                                10000baseT/Full 
        Supported pause frame use: No
        Supports auto-negotiation: Yes
        Advertised link modes:  1000baseT/Full 
                                10000baseT/Full 
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Speed: Unknown!
        Duplex: Unknown! (255)
        Port: FIBRE
        PHYAD: 0
        Transceiver: external
        Auto-negotiation: on
        Supports Wake-on: umbg
        Wake-on: g
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: no
```

Speed 显示当前网卡的速度。但是不知道为啥，我司机器是Unknown。。

##### 是否启用了相应的网卡

`ip addr show` 显示网卡及配置的地址信息

``` 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 06:50:32:00:02:18 brd ff:ff:ff:ff:ff:ff
    inet 100.73.12.12/24 brd 100.73.12.255 scope global eth0
```

首先这个系统有两个接口：lo和eth0，lo是环回接口，而我们重点关注的则是eth0这个普通网络接口

state UP: 网络接口已启用。具体详细的请阅读[Linux ip命令详解](https://www.jellythink.com/archives/469)

##### 是否正确设置路由

`route  -n`查看路由状态

``` 
[root@massive-dataset-new-002 ~]# route  -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
100.73.12.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
0.0.0.0         100.73.12.254   0.0.0.0         UG    0      0        0 eth0
```

通过route命令查看内核路由，检验具体的网卡是否连接到目标网路的路由，之后就可以尝试ping 网关，排查与网关之间的连接。

如果无法ping通网关，可能是网关限制了ICMP数据包，或者交换机设置的问题。

##### DNS工作状况

通常很多网络问题是DNS故障或配置不当造成的

使用nslookup命令查看DNS解析

``` 
[root@massive-dataset-new-002 ~]# nslookup baidu.com
Server:         100.73.18.5
Address:        100.73.18.5#53

Non-authoritative answer:
Name:   baidu.com
Address: 39.156.69.79
Name:   baidu.com
Address: 220.181.38.148
```

这里的DNS服务器 100.73.18.5 位于当前局域网内，nslookup的结果显示DNS工作正常。

如果这里nslookup命令无法解析目标域名，则很有可能是DNS配置不当

##### 是否可以正常路由到远程主机

互谅网是通过大量路由器中继连接起来的，网络的访问就是在这些节点间一跳一跳最终到达目的地，想要查看网络连接，最直接最常用的命令是ping，ping得通，说明路由工作正常，但是如果ping不通，traceroute命令可以查看从当前主机到目标主机的全部“跳”的过程。traceroute和ping命令都是使用ICMP协议包。

``` 
[root@massive-dataset-new-002 ~]#  traceroute www.baidu.com
traceroute to www.baidu.com (103.235.46.39), 30 hops max, 60 byte packets
 1  100.73.12.254 (100.73.12.254)  2.124 ms  2.670 ms  2.388 ms
 2  100.73.255.252 (100.73.255.252)  0.107 ms  0.092 ms  0.089 ms
 3  114.113.234.233 (114.113.234.233)  7.929 ms  8.298 ms  8.290 ms
 4  172.16.0.4 (172.16.0.4)  13.154 ms  14.051 ms  13.626 ms
 5  192.168.77.9 (192.168.77.9)  9.923 ms  9.330 ms  9.901 ms
 6  61.49.40.97 (61.49.40.97)  2.216 ms  2.319 ms  1.595 ms
 7  * * *
 8  219.232.11.73 (219.232.11.73)  2.156 ms 219.232.11.153 (219.232.11.153)  3.377 ms 219.232.11.233 (219.232.11.233)  2.410 ms
 9  202.96.12.21 (202.96.12.21)  3.088 ms 123.126.0.101 (123.126.0.101)  3.232 ms 125.33.186.85 (125.33.186.85)  3.251 ms
10  219.158.5.146 (219.158.5.146)  9.310 ms 219.158.4.170 (219.158.4.170)  10.142 ms 219.158.5.158 (219.158.5.158)  8.519 ms
11  219.158.3.138 (219.158.3.138)  10.748 ms  11.408 ms 219.158.16.66 (219.158.16.66)  8.184 ms
12  219.158.96.26 (219.158.96.26)  152.988 ms 219.158.98.18 (219.158.98.18)  154.549 ms  154.532 ms
13  219.158.40.190 (219.158.40.190)  170.641 ms 219.158.33.58 (219.158.33.58)  190.226 ms 219.158.40.190 (219.158.40.190)  170.384 ms
14  if-ae-8-2.tcore1.sv1-santa-clara.as6453.net (66.110.59.9)  190.528 ms  192.138 ms  189.755 ms
15  if-ae-0-2.tcore2.sv1-santa-clara.as6453.net (63.243.251.2)  170.883 ms  168.308 ms  168.036 ms
16  209.58.86.30 (209.58.86.30)  168.931 ms *  192.311 ms
17  * * *
18  103.235.45.0 (103.235.45.0)  308.516 ms  329.374 ms  331.152 ms
19  * * *
20  * * *
21  * * *^C
```

traceroute可以查看网络中继在哪里中断或者网络延时情况，“*”是因为网络不通或者某个网关限制了ICMP协议包。

##### 远程主机是否开放端口

telnet 命令是检查端口开放情况的利器

telnet IP PORT，可以查看指定远程主机是否开放目标端口

但是telnet 命令的功能非常有限，当防火墙存在时，就不能很好地显示结果，所以telnet无法连接包含两种可能：1是端口确实没有开放，2是防火墙过滤了连接。

##### 本机查看监听端口

如果要在本地查看某个端口是否开放，可以使用如下命令

`netstat -lnp | grep PORT` 查看本地指定端口的监听情况

``` 
[root@massive-dataset-new-002 ~]# netstat -anlp | grep 13307
tcp        0      0 0.0.0.0:13307               0.0.0.0:*                   LISTEN      18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:46370         ESTABLISHED 18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:51380         ESTABLISHED 18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:33466         ESTABLISHED 18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:48615         ESTABLISHED 18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:44921         ESTABLISHED 18368/mysqld        
tcp        0      0 100.73.12.12:13307          100.73.53.250:35016         ESTABLISHED 18368/mysqld 
```

其中第一列是套接字通信协议，第2列和第3列显示的是接收和发送队列，第4列是主机监听的本地地址，反映了该套接字监听的网络；第6列显示当前套接字的状态，最后一列显示打开端口的进程。

查看当前活动端口监听的网络，如果netstat找不到指定的端口，说明没有进程在监听指定端口。

##### 网络较慢的排查

iftop命令类似于top命令，查看哪些网络连接占用的带宽较多

``` 
# CentOS上安装所需依赖包
yum install flex byacc  libpcap ncurses ncurses-devel libpcap-devel
yum install iftop
```

该命令按照带宽占用高低排序，可以确定那些占用带宽的网络连接

![](/images/blog/2019-08-28-2.png){:height="80%" width="80%"}

最上方的一行刻度是整个网络的带宽比例，下面第1列是源IP，第2列是目标IP，箭头表示了二者之间是否在传输数据，以及传输的方向。最后三列分别是2s、10s、40s时两个主机之间的数据传输速率。

最下方的TX、RX分别代表发送、接收数据的统计，TOTAL则是数据传输总量。

使用 -n 选项直接显示连接的IP，否则看到的则是解析成域名后的结果。  
-i 选项可以指定要查看的网卡，默认情况下，iftop会显示自己找到的第一个网卡   
在进入iftop的非交互界面后，按 p 键可以打开或关闭显示端口，按 s 键可以显示或隐藏源主机，而按 d 键则可以显示或隐藏目标主机。

虽然iftop报告每个连接所使用的带宽，但它无法报告参与某个套按字连接的进程名称/编号（ID）。

**tcpdump**

当一切排查手段都无济于事时仍然不能找到网络速度慢、丢包严重等原因时，往往祭出杀手锏——抓包。抓包的最佳手段是在通信的双方同时抓取，这样可以同时检验发出的数据包和收到的数据包，tcpdump是常用的抓包工具。

具体自行查阅[Linux tcpdump命令详解](https://www.jellythink.com/archives/478)


---
参考链接
* [Linux系统排查4——网络篇](https://www.cnblogs.com/Security-Darren/p/4700387.html)


