---
layout: post
title: HBase Region 数量&大小
categories: [HBase]
description: HBase Region 数量&大小
keywords: HBase, Region
---

HBase Region 数量&大小

---

#### 前言

准备迁移表，评估下目标HBase集群是否能容纳下该表。表在原集群有192个 Region，因此我想评估下该目标集群是否足够容纳

#### Region 数量

通常较少的 Region 数量可使群集运行的更加平稳，官方指出每个RegionServer大约 100 个regions的时候效果最好

- HBase的一个特性MSLAB，它有助于防止堆内存的碎片化，减轻垃圾回收Full GC的问题，默认是开启的。但是每个MemStore需要2MB（一个列簇对应一个写缓存MemStore）。
所以如果每个Region有2个family列簇，总有1000个region，就算不存储数据也要3.95G内存空间

- 如果很多region，它们中MemStore也过多，内存大小触发RegionServer级别限制导致flush，就会对用户请求产生较大的影响，可能阻塞该Region Server上的更新操作

- HMaster 要花大量的时间来分配和移动 Region，且过多 Region 会增加 ZooKeeper 的负担

- 从 HBase 读入数据进行处理的 mapreduce 程序，过多 Region 会产生太多 Map 任务数量，默认情况下由涉及的 Region 数量决定

如果一个 HRegion 中 Memstore 过多，而且大部分都频繁写入数据，每次flush的开销必然会很大，因此也建议在进行表设计的时候尽量减少ColumnFamily的个数。每个region都有自己的MemStore，当大小达到了上限(hbase.hregion.memstore.flush.size，默认128MB)，会触发Memstore刷新

**计算集群 Region 数量的公式**

`((RS Xmx) * hbase.regionserver.global.memstore.size) / (hbase.hregion.memstore.flush.size * (# column families))`

假设一个RS有16GB内存，那么16384*0.4/128m 等于51个活跃的region。

如果写很重的场景下，可以适当调高 hbase.regionserver.global.memstore.size，这样可以容纳更多的 Region 数量

建议分配合理的 Region 数量，根据写请求量的情况，一般 20-200 个之间，可以提高集群稳定性，排除很多不确定的因素，提升读写性能。

监控Region Server中所有 MemStore 的大小总和是否达到了上限(hbase.regionserver.global.memstore.upperLimit ＊ hbase_heapsize，默认 40%的JVM内存使用量)，超过可能会导致不良后果，如服务器反应迟钝或compact风暴

#### Region 大小

HBase中数据一开始会写入memstore，满128MB（看配置）以后，会flush到disk上而成为storefile。当storefile数量超过触发因子时（可以配置），会启动compaction过程将它们合并为一个storefile。对集群的性能有一定影响。而当合并后的storefile大于max.filesize，会触发分割动作，将它切分成两个region。

- 当hbase.hregion.max.filesize比较小时，触发split的机率更大，系统的整体访问服务会出现不稳定现象。

- 当hbase.hregion.max.filesize比较大时，由于长期得不到split，因此同一个region内发生多次compaction的机会增加了。这样会降低系统的性能、稳定性，因此平均吞吐量会受到一些影响而下降。

hbase.hregion.max.filesize 不宜过大或过小，经过实战，生产高并发运行下，关闭某些重要场景的HBase表的major_compact！在非高峰期的时候再去调用major_compact，这样可以减少split的同时，显著提供集群的性能，吞吐量、非常有用。

---
参考链接
* [HBase最佳实践之Region数量&大小](https://mp.weixin.qq.com/s/0tGNpmBRHbI673TwxIC2NA)



