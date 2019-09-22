---
layout: post
title: 平均负载
categories: [Linux]
description: 到底应该怎么理解“平均负载”？
keywords: Linux, load average
---

到底应该怎么理解“平均负载”？

---

#### 概念

什么是平均负载？

正确定义: 单位时间内，系统中处于可运行状态和不可中断状态的平均进程数。  
错误定义: 单位时间内的 CPU 使用率。

可运行状态的进程: 正在使用 CPU 或者正在等待 CPU 的进程,即 ps aux 命令下STAT处于R状态的进程  
不可中断状态的进程: 处于内核态关键流程中的进程，且不可被打断，如等待硬件设备IO响应，ps命令D状态的进程

可以简单理解为，平均负载其实就是平均活跃进程数。

理想状态: 每个cpu上都有一个活跃进程，即平均负载数等于 cpu 数   
过载经验值: 平均负载高于 cpu 数量70%的时候

为什么是 CPU 数量的70%呢？ 请阅读[Linux系统平均负载3个数字的含义](https://blog.csdn.net/zwldx/article/details/82812704)

#### 相关命令

```
CPU 核数: lscpu、 grep 'model name' /proc/cpuinfo | wc -l
```

显示平均负载: uptime、top，显示的顺序是最近1分钟、5分钟、15分钟，从此可以看出平均负载的趋势

``` 
[root@massive-dataset-new-002 ~]# uptime
 20:36:29 up 202 days,  1:32,  1 user,  load average: 0.00, 0.00, 0.00
```

分别是当前时间、系统运行时间以及正在登录用户数、依次则是过去 1 分钟、5 分钟、15 分钟的平均负载（Load Average）

查看 cpu 负载变化的命令

```
watch -d uptime  # -d 会高亮显示变化的区域
```

strees: 压测命令，--cpu cpu压测选项，-i io压测选项，-c 进程数压测选项，--timeout 执行时间

mpstat: 多核cpu性能分析工具，-P ALL监视所有cpu

pidstat: 进程性能分析工具，-u 显示cpu利用率

#### 平均负载与cpu使用率的区别
    
CPU使用率: 单位时间内cpu繁忙情况的统计

- CPU密集型进程，CPU使用率和平均负载基本一致

- IO密集型进程，平均负载升高，CPU使用率不一定升高

- 大量等待CPU的进程调度，平均负载升高，CPU使用率也升高                      

#### 调优

平均负载过高时，如何调优

工具：stress、sysstat，yum 即可安装

1. CPU密集型进程   
mpstat -P ALL 5: -P ALL表示监控所有CPU，5表示每5秒刷新一次数据，观察是否有某个cpu的%usr会很高，但iowait应很低  
pidstat -u 5 1：每5秒输出一组数据，观察哪个进程%cpu很高，但是%wait很低，极有可能就是这个进程导致cpu飚高

2. IO密集型进程   
mpstat -P ALL 5: 观察是否有某个cpu的%iowait很高，同时%usr也较高  
pidstat -u 5 1：观察哪个进程%wait较高，同时%CPU也较高

3. 大量进程   
pidstat -u 5 1：观察那些%wait较高的进程是否有很多  




