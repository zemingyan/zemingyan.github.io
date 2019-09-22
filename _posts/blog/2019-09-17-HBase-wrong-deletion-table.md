---
layout: post
title: HBase 误删表恢复
categories: [HBase]
description: HBase 误删表恢复
keywords: HBase
---

HBase 误删表

---

#### 前言

HBase 的数据主要存储在分布式文件系统 HFile 和 HLog 两类文件中。Compaction操作会将合并完的不用的小 HFile 移动到 archive 文件夹。WAL 文件在数据完全 flush 到 HFile 中时便会过期，
被移动到oldWALs文件夹中。

HMaster 上的定时线程 HFileCleaner/LogCleaner 周期性扫描 archive 目录和 oldWALs 目录, 判断目录下的HFile或者WAL是否可以被删除，如果可以,就直接删除文件。

#### 分析

关于 HFile 文件和 HLog 文件的过期时间，其中涉及到两个参数

**hbase.master.logcleaner.ttl**

HLog在 oldWAL 目录中生存的最长时间，过期则被 Master 的线程清理，默认是600000（ms）

![](/images/blog/2019-09-17-2.png){:height="80%" width="80%"}

**hbase.master.hfilecleaner.plugins**

HFile的清理插件列表，逗号分隔，被HFileService调用，可以自定义，默认org.apache.hadoop.hbase.master.cleaner.TimeToLiveHFileCleaner。

![](/images/blog/2019-09-17-3.png){:height="80%" width="80%"}

官方文档解释

![](/images/blog/2019-09-17-4.png){:height="80%" width="80%"}

在类org.apache.hadoop.hbase.master.cleaner.TimeToLiveHFileCleaner中，可以看到如下的设置

默认 HFile 的失效时间是5分钟。由于一般的hadoop平台默认都没有对该参数的设置，可以在配置选项中添加对hbase.master.hfilecleaner.ttl的设置。

![](/images/blog/2019-09-17-5.png){:height="80%" width="80%"}

实际在测试的过程中，删除一个hbase表，在archive文件夹中，会立即发现删除表的所有region数据（不包含regioninfo、tabledesc等元数据文件），等待不到6分钟所有数据消失，说明所有数据生命周期结束，被删除。

#### 恢复

删除表步骤 distable + drop。当然 truncate 清除表数据也可以通过这种方式恢复

##### 抢救数据

保证在删除表之后的5分钟之内将 HDFS 目录 /hbase/archive/ 文件夹下的表 region 数据拷贝到 /tmp 下。

``` 
hadoop fs -cp /hbase/archive/data/default/{tableName}  /tmp
```

##### 建表

新建同名和同列族的表

注意: **请提供表结构，如果表结构未提供，将很难恢复**

##### 拷贝

将抢救下来的 region 数据拷贝到 hbase 表对应的目录下

``` 
hadoop fs -cp /tmp/data/default/test/* /hbase/data/default/test
```

##### 元数据修复

HDFS 里有对应的 Region 数据了。想当然是想到用-fixMeta修复

-fixMeta: 主要修复.regioninfo文件和hbase:meta元数据表的不一致。修复的原则是以HDFS文件为准：如果region在HDFS上存在，但在hbase.meta表中不存在，就会在hbase:meta表中添加一条记录。反之如果在HDFS上不存在，而在hbase:meta表中存在，就会将hbase:meta表中对应的记录删除。

但是因为缺少 regioninfo 信息，不能直接用 -fixMeta 修复。所以得先修复下 regioninfo、tableinfo

-fixHdfsOrphans: 尝试修复hdfs中没有.regioninfo文件的region目录

-fixTableOrphans: 尝试修复hdfs中没有.tableinfo文件的table目录（只支持在线模式）

尝试先用fixHdfsOrphans,fixTableOrphans,fixMeta的顺序进行修复，失败。

最后用-repair修复，但是内部的执行顺序可能不对，执行一遍失败，多执行几遍，成功。

``` 
sudo -u hbase hbase hbck -repair {tableName}
```

##### 验证

通过 HBase Shell 验证

#### 总结

-repair 属于高危修复命令。要慎用，我指定 tableName 是避免 repair 影响到其他表

这里是说删表恢复。误删表数据也有恢复方法。[误删HBase数据如何抢救？](https://mp.weixin.qq.com/s/XBwan9ZOCrOM565Cdw6Z0A)

感兴趣自己研读

---
参考链接
* [HDFS和Hbase误删数据恢复](https://blog.csdn.net/chaolovejia/article/details/48265335)
* [HBase之disable+drop删除表疑点解惑](https://www.cnblogs.com/yingjie2222/p/6377359.html)



