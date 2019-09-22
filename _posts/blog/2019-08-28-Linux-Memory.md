---
layout: post
title: Linux 内存
categories: [Linux]
description: Linux 内存
keywords: Linux
---

内存是评判服务器的一个非常重要的指标。内存的多少，可能会直接影响着服务器的整体性能。

---

#### 前言

内存是评判服务器的一个非常重要的指标。内存的多少，可能会直接影响着服务器的整体性能。阅读[Linux性能监测：内存篇](https://www.jellythink.com/archives/462)

#### 实操

free命令可以显示Linux系统中空闲的、已用的物理内存及swap内存，及被内核使用的buffer。

free命令习惯上有以下几种形式

``` 
free -k # 以KB为单位显示内存使用情况
free -m # 以MB为单位显示内存使用情况
free -g # 以GB为单位显示内存使用情况
free -h # 以人类友好的方式显示内存使用情况
```

输入free -m时，系统就会输出以下内容

``` 
[root@massive-dataset-new-002 ~]# free -g
             total       used       free     shared    buffers     cached
Mem:            61         19         41          0          0         10
-/+ buffers/cache:          8         52
Swap:            4          0          4
```

输出解释  
total: 内存总数，物理内存总数  
used: 已经使用的内存数  
free: 空闲的内存数  
shared: 多个进程共享的内存总额  
buffers: 缓冲内存数  
cached: 缓存内存数  
\- buffers/cached: 应用使用内存数  
\+ buffers/cached: 应用可用内存数  
Swap: 交换分区，虚拟内存  

通过free命令查看机器空闲内存时，会发现free的值很小。  
这主要是因为，在Linux系统中有这么一种思想，内存不用白不用，因此它尽可能的cache和buffer一些数据，以方便下次使用。
但实际上这些内存也是可以立刻拿来使用的。

buffers: 作为 buffer cache 的内存，用来缓存磁盘数据。

cached: 作为 page cache 的内存, 用来缓存文件数据。

如果 cache 的值很大，说明cache住的文件数很多。如果频繁访问到的文件都能被cache住，那么磁盘的读IO 必会非常小

在使用free命令时，都是需要重点关注 `- buffers/cached`和 + `buffers/cached`。

\- buffers/cached，即used - buffers/cached，表示应用程序实际使用的内存    
\+ buffers/cached，即free + buffers/cached，表示理论上都可以被使用的内存    

可见-buffers/cache反映的是被程序实实在在吃掉的内存，而+buffers/cache反映的是可以挪用的内存总数。








