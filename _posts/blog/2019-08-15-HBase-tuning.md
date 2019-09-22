---
layout: post
title: HBase 调优
categories: [HBase]
description: HBase 调优
keywords: HBase
---

HBase 调优

---

#### 前言

HBase 配置参数极其繁多，参数配置可能会影响到 HBase 性能问题，因此得好好总结下。

HBase 调优是个技术活。得结合多年生产经验加测试环境下性能测试得出。

1. JVM垃圾回收优化

2. 本地 memstore 分配缓存优化

3. Region 拆分优化

4. Region 合并优化

5. Region 预先加载优化

6. 负载均衡优化

7. 启用压缩，推荐snappy

8. 进行预分区，从而避免自动 split，提高 HBase 响应速度

9. 避免出现 region 热点现象，启动按照 table 级别进行 balance

#### RS 参数

**hbase.server.thread.wakefrequency**

该值默认是 10 秒，它影响着 Flush 和 Compaction

FlushHandler 从队列 flushQueue 取出需要刷新的请求，从队列里取请求超时时间是该参数

Compaction 后台定期线程检查也是该参数 

#### flush 参数

**hbase.hregion.memstore.block.multiplier**

写请求达到 HRegion 后，HRegion 首先会加行锁，然后进行 checkResource 操作，在 checkResource 操作里主要检查 memstoreSize 是否大于 blockingMemStoreSize，其中 blockingMemStoreSize 由等于 memstoreFlushSize * 
hbase.hregion.memstore.block.multiplier，该参数默认值是 4

hbase.hregion.memstore.block.multiplier 设置的太大在写入量大的时候很可能会导致机器内存耗尽而引发 OutofMem 错误

**hbase.hregion.memstore.flush.size**

上文提到的 memstoreFlushSize 大小由参数 hbase.hregion.memstore.flush.size 指定，该值默认值是 128M

**hbase.hstore.flusher.count**

MemStoreFlusher 的职责是接受 HRegion 的刷新请求并调度该请求，在 MemStoreFlusher 内部有一个 flushQueue 队列，消费此队列的是 FlushHandler，FlushHandler 的个数由参数 hbase.hstore.flusher.count 设定, 默认值是 2

如果线程个数较少，MemStore 刷新将排队。对于更多的线程个数，刷新将并行执行，从而增加 HDFS 负载。这也会导致更多的 compact。

**hbase.hstore.blockingStoreFiles**

在HFile的数量过多的时候会限制写请求的速度，如果过多就会阻塞 flush 而进行 compact 操作并阻塞一定时间后才进行 flush 操作

阻塞文件参数由参数 hbase.hstore.blockingStoreFiles 控制该值默认为 7，阻塞时间由参数 hbase.hstore.blockingWaitTime 控制，该值默认为 90000 毫秒

**hbase.hstore.blockingWaitTime**

值默认为 90000 毫秒，上文有提参数作用

#### Split 参数

**hbase.regionserver.region.split.policy**

HBase 的分裂策略可以通过表的属性 SPLIT_POLICY 指定，也可以通过 hbase-site.xml 全局指派，参数为 hbase.regionserver.region.split.policy，默认为IncreasingToUpperBoundRegionSplitPolicy。

**hbase.regionserver.thread.split**

执行 split 的线程数，默认为 1

**hbase.regionserver.regionSplitLimit**

当前 regionserver 的 region 个数最大值，如果当前 regionserver 的 region 个数超过该值，那么将不会在进行 split 操作。默认值 1000

#### Compaction 参数

Compaction 的主要目的

- 将多个HFile 合并为较大HFile，从而提高查询性能
- 减少HFile 数量，减少小文件对 HDFS 影响
- 提高 Region 初始化速度

**hbase.hstore.compaction.min**

当某个列族下的 HFile 文件数量超过这个值，则会触发 minor compaction 操作 默认是3，比较小，建议设置 10-15   
设置过小会导致合并文件太频繁，特别是频繁 bulkload 或者数据量比较大的情况下 设置过大又会导致一个列族下面的 HFile 数量比较多，影响查询效率

**hbase.hstore.compaction.max**

一次最多可以合并多少个HFile，默认为 10 限制某个列族下面选择最多可选择多少个文件来进行合并   
注意需要满足条件`hbase.hstore.compaction.max` > `hbase.hstore.compaction.min`

**hbase.hstore.compaction.max.size**

默认 Long 最大值，minor_compact 时 HFile 大小超过这个值则不会被选中合并   
用来限制防止过大的 HFile 被选中合并，减少写放大以及提高合并速度

**hbase.hstore.compaction.min.size**

默认 memstore 大小，minor_compact 时 HFile 小于这个值，则一定会被选中   
可用来优化尽量多的选择合并小的文件

**hbase.regionserver.thread.compaction.small**

默认1，每个RS的 minor compaction线程数，其实不是很准确，这个线程主要是看参与合并的 HFile 数据量 有可能 minor compaction 数据量较大会使用 compaction.large 提高线程可提高 HFile 合并效率

**hbase.regionserver.thread.compaction.large**

默认1，每个RS的 major compaction线程数，其实不是很准确，这个线程主要是看参与合并的 HFile 数据量 有可能 minor compaction 数据量较大会使用 compaction.large 提高线程可提高 HFile 合并效率

**hbase.hregion.majorcompaction**

默认604800000, 单位是毫秒，即7天。major compaction 间隔

设置为0即关闭hbase major compaction，改为业务低谷手动执行。[HBase 自动大合并脚本](https://lihuimintu.github.io/2019/06/09/HBase-timing-major-compaction/)

**hbase.hregion.majorcompaction.jetter**

Major compaction 抖动参数，默认值0.5。这个参数是为了避免major compaction同时在各个regionserver上同时发生，避免此操作给集群带来很大压力。 这样节点major compaction就会在 + 或 - 两者乘积的时间范围内随机发生。

---
参考链接
* [HBase调优 HBase Compaction参数调优](https://mp.weixin.qq.com/s/uMDoSnsbcqznCvSQCW5_yA)
* [Hbase Region Split compaction 过程分析以及调优](https://cloud.tencent.com/developer/article/1005586)
* [深入理解 HBase Compaction 机制](https://blog.csdn.net/u011598442/article/details/90632702)



