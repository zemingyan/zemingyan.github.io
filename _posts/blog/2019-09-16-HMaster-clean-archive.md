---
layout: post
title: HMaster 定期清理 archive
categories: [HBase]
description: HMaster 定期清理 archive
keywords: HBase
---

HFileCleaner 定期清理 archive 下的文件

---

#### 前言

表一旦删除，刚开始是可以在 archive 中看到删除表的数据文件，但是等待一段时间后 archive 中的数据就会被彻底删除，再也无法找回。

这是因为 master 上会启动一个定期清理 archive 中垃圾文件的线程（HFileCleaner），定期会对这些被删除的垃圾文件进行清理。

#### 分析

定时清理任务的插件设置会从 hbase.master.hfilecleaner.plugins 配置里加载所有 BaseHFileCleanerDelegate。 只有所有 delegate 都同意才能被删除。

![](/images/blog/2019-09-17-1.png){:height="80%" width="80%"}

就是通过 HFileLinkCleaner，SnapshotHFileCleaner，TimeToLiveHFileCleaner 这三种规则的约束来清理archive中的数据

##### HFileLinkCleaner

``` 
/**
 * HFileLink cleaner that determines if a hfile should be deleted.
 * HFiles can be deleted only if there're no links to them.
 *
 * When a HFileLink is created a back reference file is created in:
 *      /hbase/archive/table/region/cf/.links-hfile/ref-region.ref-table
 * To check if the hfile can be deleted the back references folder must be empty.
 */
```

如果对archive中的文件的引用不存在了，则可以删除

问题: /hbase/archive/table/region/cf/.links-hfile/ref-region.ref-table 从何而来？
回答: HBase 做 clone_snapshot 的时候。有兴趣可以自看[HBase原理 – 分布式系统中snapshot是怎么玩的？](http://hbasefly.com/2017/09/17/hbase-snapshot/)

##### SnapshotHFileCleaner

``` 
/**
 * Implementation of a file cleaner that checks if a hfile is still used by snapshots of HBase
 * tables.
 */
```

未被 snapshots 引用的文件，可以删除

问题: snapshot 为嘛会引用到 archive 中的文件？
回答: 表做了snapshot后，此时该表的元数据以及相关的link文件都存储在snapshot中, 该表发生 compact 操作前会将原始表移动到 archive 目录下再执行 compact。对于表删除操作，正常情况也会将删除表数据移动到archive目录下），这样snapshot对应的元数据就不会失去意义，只不过原始数据不再存在于数据目录下，而是移动到了archive目录下。

##### TimeToLiveHFileCleaner

``` 
/**
 * HFile cleaner that uses the timestamp of the hfile to determine if it should be deleted. By
 * default they are allowed to live for {@value #DEFAULT_TTL}
 */
```

默认清理时间超过5分钟的HFile

#### 实战

在查看公司 HBase 集群 archive 文件夹发现有 2018 年的数据未清理。。。

按所学知识正常的情况下超过5分钟的HFile就会被清理。

继续翻阅文件发现了 links-hfile 文件。因此推断是因为有 clone 的表还依赖着。

links-hfile 文件的消失是在 clone 表做 Compact 之后会消失。但是因为 clone 表已经一直没有写入，所以 region 没有做 majorCompat, 后台线程周期性检查也会因为 needsCompaction() 方法去判断没有足够多的文件触发了 Compaction

只能手动触发 majorCompact，一旦手动触发，HBase 会不做很多自动化检查，直接执行合并。

等待一段时间回看发现 archive 文件的2018年数据已清理

整个过程忘记留图了。。。导致只能后续口诉加回忆。

---
参考链接
* [HMaster 功能之定期清理archive](https://www.jianshu.com/p/f82aafd7b381)











