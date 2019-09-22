---
layout: post
title: 测试故障模拟
categories: [Linux]
description: 测试故障模拟
keywords: Linux
---

针对一定场景实现故障现场进行模拟测试

---

#### 前言

与测试同事沟通时，看到一篇测试同学写的故障模拟文档。本着 SRE 精神，有时候自己也需要模拟下故障场景。
尤其是模拟应急故障演练时特别需要

#### 打满 CPU

``` 
#!/bin/bash
 
cat << EOF > /tmp/infiniteburn.sh
#!/bin/bash
while true;
    do openssl speed;
done
EOF
 
for i in {1..32}
do
    nohup /bin/bash /tmp/infiniteburn.sh &
done
```

`openssl speed`是用来测试加密算法性能的，是一种CPU密集型计算，运行一个脚本只会打满一个CPU

`for i in {1..32}`用来执行32次/tmp/infiniteburn.sh脚本，32是假设当前机器的内核个数不会超过32，若超过修改一下这个数值即可


#### 打满 IO

``` 
#!/bin/bash
 
cat << EOF > /tmp/loopburnio.sh
#!/bin/bash
while true;
do
    dd if=/dev/urandom of=/burn bs=1M count=1024 iflag=fullblock
done
EOF
 
nohup /bin/bash /tmp/loopburnio.sh &
```

dd命令用于读取、转换并输出数据   
dd可以从标准输入或文件中读取数据，根据指定的格式来转换数据再输出到文件、设备或标准输出

`dd if=/dev/urandom of=/burn bs=1M count=1024 iflag=fullblock`
这条命令的意思是采用dd工具模拟读写。if指定输入的文件名，of指定输出的文件名，bs同时设置读写块的大小为1M，count是指仅拷贝1024个块，块大小等于bs指定的字节数。iflag=fullblock表示堆积满block

##### paping工具

通过该工具可以查看模拟网络故障的效果如何

paping 可以在 Linux 平台上测试网络的连通性以及网络延迟等

``` 
wget http://www.updateweb.cn/softwares/paping_1.5.5_x86-64_linux.tar.gz
tar -zvxf paping_1.5.5_x86_linux.tar.gz
./paping -p 80 -c 5000 www.baidu.com
```

如遇到`./paping: error while loading shared libraries: libstdc++.so.6: cannot open shared object file: No such file or directory`
错误，安装对应库来解决`yum -y install libstdc++6` `yum -y install lib32stdc++6`

使用方式

``` 
Syntax: paping [options] destination
 
Options:
 -?, --help     display usage
 -p, --port N   set TCP port N (required)                       // 指定被测试服务的TCP端口（必须）
     --nocolor  Disable color output                            // 屏蔽彩色输出
 -t, --timeout  timeout in milliseconds (default 1000)          // 指定超时时长，单位为毫秒，默认为1000
 -c, --count N  set number of checks to N                       // 指定测试次
```

示例

``` 
[root@100.73.12.12 miaold]#  ./paping -p 80 -c 5 www.baidu.com  
paping v1.5.5 - Copyright (c) 2011 Mike Lovell
 
Connecting to www.a.shifen.com [180.97.33.107] on TCP 80:
 
Connected to 180.97.33.107: time=0.23ms protocol=TCP port=80
Connected to 180.97.33.107: time=0.24ms protocol=TCP port=80
Connected to 180.97.33.107: time=0.26ms protocol=TCP port=80
Connected to 180.97.33.107: time=0.24ms protocol=TCP port=80
Connected to 180.97.33.107: time=0.19ms protocol=TCP port=80
 
Connection statistics:
        Attempted = 5, Connected = 5, Failed = 0 (0.00%)
Approximate connection times:
        Minimum = 0.19ms, Maximum = 0.26ms, Average = 0.23ms
```

可以看到平均连接时间为0.23ms

#### 网络延迟

该命令将网卡eth0的传输设置为延迟300ms发送

```
tc qdisc add dev eth0 root netem delay 300ms
```

该命令将 eth0 网卡的传输设置为延迟 300ms ± 50ms (250 ~ 350 ms 之间的任意值)发送 

``` 
tc qdisc add dev eth0 root netem delay 300ms 50ms
```

对应的删除命令为`tc qdisc del dev eth0 root netem xxxxxx`

#### 网络中断

该命令将 eth0 网卡的传输设置为随机产生 10% 的损坏的数据包

``` 
tc qdisc add dev eth0 root netem corrupt 10%
```

#### 网络丢包

该命令将 eth0 网卡的传输设置为随机丢掉7%的数据包, 成功率为25%；如果不加上后面的25%，那么一次就是随机丢掉7%的数据包

``` 
tc qdisc add dev eth0 root netem loss 7% 25%
```

机器假死、进程假死和线程假死时，其上游调用所接受到的状态分别为:

机器假死 - timeout  
进程假死 - connection refused  
线程假死 - timeout  

#### 模拟 timeout 

``` 
iptables -A OUTPUT -p tcp --dport 6666 -j DROP
```

#### 模拟 connection refused

``` 
iptables -A OUTPUT -p tcp --dport 7777 -j REJECT
```

对应的删除命令为

iptables -D xxxxxx

---
参考链接
* [使用cat和EOF添加多行数据](https://blog.csdn.net/lym152898/article/details/83306993)
