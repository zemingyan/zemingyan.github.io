---
layout: post
title: HMaster 由于 MasterProcWals 过多导致启动选主超时
categories: [HBase]
description: HMaster 由于 MasterProcWals 过多导致启动选主超时
keywords: HBase
---

由 MasterProcWals 状态日志过多导致的HBase Master重启失败问题

---

#### 前言

由于 Kylin 业务迁移后， Master 日志中仍然报出 Kylin 相关表的合并日志，因此对 HBase 进行滚动重启，发现重启主 Master 后发生选主超时

#### 分析

CM 页面显示未发现有活动的 Master 并有红色告警提示

但在 zookeeper 中 get /hbase/master 发现 100.106.37.5 的 master 选主已经成功

查看对应日志发现一直刷如下信息

![](/images/blog/2019-09-17-6.png){:height="80%" width="80%"}

去 HDFS 查看该路径下有接近 20000 个这样的日志文件，且单副本达到了300余G的大小

同时根据已知 Master 进入活动状态需要读取并实例化所有正在运行的程序当前记录在 /hbase/MasterProcWALs/ 目录下对应的文件。

``` 
MasterProcWAL: HMaster记录管理操作，比如解决冲突的服务器，表创建和其它DDLs等操作到它的WAL文件中，
这个WALs存储在MasterProcWALs目录下，它不像RegionServer的WALs，HMaster的WAL也支持弹性操作，
就是如果Master服务器挂了，其它的Master接管的时候继续操作这个文件。
```

因此推断是由于 MasterProcWA 日志文件太大太多，回放时间超过了 Master 选主时间

等2万个日志文件刷完以后，Master 报如下错误

![](/images/blog/2019-09-18-1.png){:height="80%" width="80%"}

再过 900000ms 这么长时间继续报该错误

#### 解决

根据日志和[由MasterProcWals状态日志过多导致的HBase Master重启失败问题](https://cloud.tencent.com/developer/article/1349438)猜测
可能是一个Bug

move /hbase/MasterProcWALs 目录下的所有文件，重启 HBase Master 解决问题。

重启 HBase 观察日志发现已经正常刷 Region 信息日志

![](/images/blog/2019-09-18-2.png){:height="80%" width="80%"}

去 HBase Shell 中操作，报错ERROR：master is initializing，please wait（在这之前，没有移动 MasterProcWALs 日志 选主超时的时候，此时报错为ERRAOR：master server not running yet），证明已经重启成功，大概五分钟之后，Master 选主成功，CM页面告消失，HBase Shell 能够正常操作

#### 总结

从这次事故以后 MasterProcWals 状态日志就没有过多的情况了。

据 Fayson 大神说该问题主要和HBase某个分支的实现方式有关，据说已经重新设计了该实现方式，新的实现方式能够避免该问题，将在CDH 6中应用。

---
参考链接
* [由MasterProcWals状态日志过多导致的HBase Master重启失败问题](https://cloud.tencent.com/developer/article/1349438)





