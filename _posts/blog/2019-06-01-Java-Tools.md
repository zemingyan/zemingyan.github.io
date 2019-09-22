---
layout: post
title: Java 板斧
categories: [Linux]
description: Java 板斧
keywords: Linux
---

Java 排查问题的命令和工具

---

#### 前言



#### jps

``` 
jps -mlvV
```

![](/images/blog/2019-06-05-1.png){:height="80%" width="80%"} 

如果遇到 process information unavailable 情况。到 jdk 的安装路径下运行命令 bin/jps

#### jstack

查看当前java进程的堆栈状态

```
jstack 2815
```

jstack 命令生成的 thread dump 信息包含了JVM中所有存活的线程

为了分析指定线程，必须找出对应线程的调用栈。

在top命令中，已经获取到了占用cpu资源较高的线程pid，将该pid转成16进制的值，在thread dump中每个线程都有一个nid，找到对应的nid即可；

隔段时间再执行一次stack命令获取thread dump，区分两份dump是否有差别，在nid=0x246c的线程调用栈中，发现该线程一直在执行JstackCase类第33行的calculate方法，
得到这个信息，就可以检查对应的代码是否有问题。

##### jinfo

可看系统启动的参数，如下

``` 
jinfo 30670
jinfo -flags  30670
```

##### jmap

[jvm 性能调优工具之 jmap](https://www.jianshu.com/p/a4ad53179df3)

##### jstat

jstat参数众多，但是使用一个就够了

``` 
jstat -gcutil 2815 1000 
```

2815 为 PID；1000 为间隔时间，单位为毫秒；后面还可以跟查询次数

![](/images/blog/2019-09-03-2.png){:height="80%" width="80%"}

S0 幸存1区当前使用比例  
S1 幸存2区当前使用比例  
E 伊甸园区使用比例  
O 老年代使用比例  
M 元数据区使用比例  
CCS 压缩使用比例  
YGC 年轻代垃圾回收次数  
FGC 老年代垃圾回收次数  
FGCT 老年代垃圾回收消耗时间  
GCT 垃圾回收消耗总时间  

#### VM options

有时候可以配置下 java 启动参数选项

你的类到底是从哪个文件加载进来的

```
-XX:+TraceClassLoading
结果形如[Loaded java.lang.invoke.MethodHandleImpl$Lazy from D:programmejdkjdk8U74jrelib
t.jar]
```

应用挂了输出dump文件

``` 
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/admin/logs/java.hprof
```


---
参考链接
* [阿里员工排查问题的工具清单，总有一款适合你！](https://mp.weixin.qq.com/s/eDL4-XJZO4Y8Hs4U0bCcnQ)
* [运维技巧之逼格高又实用的Linux命令](https://mp.weixin.qq.com/s/f_yEBXHjC_v-PlcrFi9ryQ)
* [java命令--jstack 工具](https://www.cnblogs.com/kongzhongqijing/articles/3630264.html)


