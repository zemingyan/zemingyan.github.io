---
layout: post
title: HBase Compaction
categories: [HBase]
description: HBase Compaction
keywords: Compaction, HBase
---

Compaction 是 buffer->flush->merge 的 LSM 树的关键操作

---

#### 介绍

![](/images/blog/2019-06-06-1.png){:height="80%" width="80%"}

一个表中的数据存储到 RS 上，RS 会管理实际存储表的数据的 region，每个 region上 每一个 ColumnFamily 
会有一个 MemStore。

写入的数据写入 MemStore 内存区，当内存缓冲区满了之后，会将数据刷新到磁盘，随时间推移，一个 Store 中的 StoreFile 数量将会增长

Compaction 是一个操作，通过merge，将会减少一个 Store 中的StoreFile的数量，从而提高读操作的性能。

Compaction 是资源密集型操作，将提高或者影响性能，取决于很多因素。

#### 类型

[官方Compaction介绍](http://hbase.apache.org/book.html#compaction)

合并有两种类型: 小合并（minor Compression）和大合并（major Compression）

Minor & Major Compaction的区别

1）Minor 操作只用来做部分文件的合并操作以及包括 minVersion = 0 并且设置 ttl 的过期版本清理，不做任何删除数据、多版本数据的清理工作。

2）Major 操作是对 Region 下的 HStore 下的所有 StoreFile 执行合并操作，最终的结果是整理合并出一个文件。

![](/images/blog/2019-06-06-2.png){:height="80%" width="80%"}

#### 作用

主要起到如下几个作用:

1）合并文件, 提高读写数据的效率

2）清除删除、过期、多余版本的数据

3）文件本地化

RegionServer 最终目的是要实现数据本地化，才能够快速查找数据，HDFS 客户端默认拷贝三份数据副本，
其中第一份副本写到本地节点上，第二和第三份则写在不同机器的节点上 (RegionServer)；

Region 的拆分会导致 RegionServer 需要读取非本地的 StoreFile，
此时，HDFS 将会自动通过网络拉取数据，但通过网络读写数据相对地比本地读写数据的效率要低，要提升效率，
必须尽可能采用数据本地性，这也是为什么 HBase 要不定时地进行大合并和刷新把数据聚合在本地磁盘上来实现数据本地化，提升查询效率。   
其次，在用户查询数据的时候，将会减少查询文件的数量，提高HBase查询效率，减少HDFS上保持寻址所有小文件的压力。压缩可能需要大量资源，并且可能会因许多因素而有助于或阻碍性能。

#### 触发时机

在什么情况下会发生 Compaction 呢？

HBase 中可以触发 compaction 的因素有很多，最常见的因素有这么三种: MemStore Flush、后台线程周期性检查、手动触发。

1. MemStore Flush: 应该说 compaction 操作的源头就来自flush操作，MemStore flush会产生 HFile 文件，文件越来越多就需要 compact。因此在每次执行完 Flush 操作之后，都会对当前 Store 中的文件数进行判断，符合条件就会触发 compaction。    
需要说明的是，compaction 都是以 Store 为单位进行的，而在 Flush 触发条件下，整个 Region 的所有 Store 都会执行compact，所以会在短时间内执行多次 compaction。

2. 后台线程周期性检查: CompactionChecker 是RS上的工作线程(Chore)。后台线程 CompactionChecker 定期触发检查是否需要执行 compaction，设置执行周期是通过 threadWakeFrequency 指定。  
   大小通过 hbase.server.thread.wakefrequency 配置(默认10000毫秒)，然后乘以默认倍数 hbase.server.compactchecker.interval.multiplier (1000), 
   毫秒时间转换为秒。因此，在不做参数修改的情况下，CompactionChecker大概是2hrs 46mins 40sec 执行一次。
   
3. 手动触发：一般来讲，手动触发 compaction 通常是为了执行 major compaction，原因有三，其一是因为很多业务担心自动 major compaction 影响读写性能，因此会选择低峰期手动触发；   
其二也有可能是用户在执行完 alter 操作之后希望立刻生效，执行手动触发 major compaction API；   
其三是 HBase 管理员发现硬盘容量不够的情况下手动触发 major compaction 删除大量过期数据；  
无论哪种触发动机，一旦手动触发，HBase会不做很多自动化检查，直接执行合并。

#### Compaction 诱发因子

参数名 | 配置项 | 默认值
-|-|-
minFilesToCompact | hbase.hstore.compactionThreshold | 3 |
maxFilesToCompact | hbase.hstore.compaction.max | 10 | 
minCompactSize | hbase.hstore.compaction.min.size | 128 MB 即(memstoreFlushSize) | 
maxCompactSize | hbase.hstore.compaction.max.size | Long.MAX_VALUE | 

说明: 在以前版本的HBase中，hbase.hstore.compaction.min 调用了该参数 hbase.hstore.compactionThreshold。

compactionChecker 实现类是 CompactionChecker, 其继承了ScheduledChore，在之前已经分析过，这个类通过实现 chore 方法而实现周期性的调用，其初始化是在initializeThreads中

直接看其Chore方法

![](/images/blog/2019-09-02-1.png){:height="80%" width="80%"}

方法比较简单，取出所有在线的region，遍历region上的所有store（HStore）判断是否需要compact

判断是 needsCompaction() 方法去判断是否足够多的文件触发了 Compaction 的条件。

![](/images/blog/2019-06-09-1.png){:height="80%" width="80%"}

条件为: HStore 中 StoreFiles 的个数 – 正在执行 Compacting 的文件个数 > minFilesToCompact

chore 方法中 needsCompaction判断的是minor compact 是否需要执行

chore 方法中可以直观的看到 majorCompact 是通过 isMajorCompaction() 方法判断的这是很多判断条件的合成

isMajorCompaction 判断上次进行majorCompaction到当前的时间间隔是否超过 hbase.hregion.majorcompaction 设置的值

#### 选择合适 HFile 合并

选择合适的文件进行合并是整个 compaction 的核心，因为合并文件的大小以及其当前承载的IO数直接决定了 compaction 的效果。

最理想的情况是，这些文件承载了大量IO请求但是大小很小，这样compaction本身不会消耗太多IO，而且合并完成之后对读的性能会有显著提升。
然而现实情况可能大部分都不会是这样，在0.96版本和0.98版本，分别提出了两种选择策略，在充分考虑整体情况的基础上选择最佳方案。

无论哪种选择策略都会首先对该 Store 中所有 HFile 进行一一排查，排除不满足条件的部分文件:

执行 RatioBasedCompactionPolicy 的 selectCompaction 算法

![](/images/blog/2019-09-02-2.png){:height="80%" width="80%"}

第一个红框: 排除当前正在执行compact的文件及其比这些文件更老的所有文件（SequenceId更大）

![](/images/blog/2019-09-02-3.png){:height="80%" width="80%"}

第二个红框: 排除某些过大的单个文件，如果文件大小大于hbase.hzstore.compaction.max.size（默认Long最大值），则被排除，否则会产生大量IO消耗

经过排除的文件称为候选文件，HBase接下来会再判断是否满足major compaction条件，如果满足，就会选择全部文件进行合并。

![](/images/blog/2019-09-02-4.png){:height="80%" width="80%"}

第三个红框: 判断是否需要进行 major Compaction，这是很多判断条件的合成。

> * 用户强制执行 major compaction
> * 长时间没有进行 compact。hbase.hregion.majorcompaction 设置的值，也就是判断上次进行 major Compaction 到当前的时间间隔，如果超过设置值，并且满足另外一个条件是 compactSelection.getFilesToCompact().size() < this.maxFilesToCompact (即候选文件数小于 hbase.hstore.compaction.max）。
> * Store中含有Reference文件，Reference文件是split region产生的临时文件，只是简单的引用文件，一般必须在compact过程中删除

    因此，通过设置 hbase.hregion.majorcompaction = 0 可以关闭 CompactionChecke 触发的 major compaction，但是无法关闭用户调用级别的 major Compaction。
 
如果不满足 major compaction 条件，就必然为 minor compaction，HBase 主要有两种 minor 策略: RatioBasedCompactionPolicy 和 ExploringCompactionPolicy

Ratio 策略是 0.94版本的默认策略，而 0.96 版本之后默认策略就换为了 Exploring 策略

深入阅读[HBase Compaction算法之ExploringCompactionPolicy](https://my.oschina.net/u/220934/blog/363270)

ExploringCompactionPolicy 跟 RatioBasedCompactionPolicy 的区别，简单的说就是 RatioBasedCompactionPolicy 是简单的从头到尾遍历StoreFile列表，遇到一个符合Ratio条件的序列就选定执行Compaction。而 ExploringCompactionPolicy 则是从头到尾遍历的同时记录下当前最优，然后从中选择一个全局最优列表。

ratio 默认值是1.2，但是打开了非高峰时间段的优化时，可以有不同的值，非高峰的ratio默认值是5.0，此优化目的是为了在业务低估时可以合并更多的数据，目前此优化只能是天的小说时间段，还不算灵活。其有如下四个参数控制

配置项 | 默认值 | 含义
-|-|-
hbase.hstore.compaction.ratio | 1.2F |  |
hbase.hstore.compaction.ratio.offpeak | 5.0F | 与下面两个参数联用 | 
hbase.offpeak.start.hour | -1 | 设置 HBase offpeak 开始时间[0,23] | 
hbase.offpeak.end.hour | -1 | 设置 HBase offpeak 结束时间 [0,23] | 

如果默认没有设置offpeak时间的话，那么完全按照hbase.hstore.compaction.ration来进行控制。如下图所示，如果filesSize[i]过大，超过后面8个文件总和*1.2，那么该文件被认为过大，而不纳入minor Compaction的范围。

![](/images/blog/2019-06-09-2.png){:height="80%" width="80%"}

这样做使得 Compaction 尽可能工作在最近刷入 HDFS 的小文件的合并，从而使得提高 Compaction 的执行效率。

**ExploringCompactionPolicy 继承 RatioBasedCompactionPolicy，重写了 applyCompactionPolicy 方法，applyCompactionPolicy 是对 minor compaction 的选择文件的策略算法。**

接着通过 selectCompaction 选出的文件，加入到 filesCompacting 队列中。创建compactionRequest，提交请求。

#### 挑选合适的线程池

HBase 实现中有一个专门的线程 CompactSplitThead 负责接收 compact 请求以及 split 请求，
而且为了能够独立处理这些请求，这个线程内部构造了多个线程池：largeCompactions、smallCompactions以及splits等，
其中splits线程池负责处理所有的split请求，
largeCompactions和smallCompaction负责接收处理CR(CompactionRequest)，
其中前者用来处理大规模compaction，后者处理小规模compaction。

哪些 compaction 应该分配给 largeCompactions 处理，哪些应该分配给 smallCompactions 处理？是不是 Major Compaction 就应该交给largeCompactions线程池处理？不对。  
而是根据一个配置hbase.regionserver.thread.compaction.throttle的设置值(一般在hbase-site.xml没有该值的设置)   
而是采用默认值2 * minFilesToCompact * memstoreFlushSize，如果 compactRequest 需要处理的 storefile 文件的大小总和，大于throttle的值，则会提交到largeCompactions线程池进行处理，反之亦然。

这里需要明白三点:

1 上述设计目的是为了能够将请求独立处理，提供系统的处理性能。

2 这两个线程池不是根据 CR 来自于 Major Compaction 和 Minor Compaction 来进行区分

3 largeCompactions 线程池和 smallCompactions 线程池默认都只有一个线程，用户可以通过参数 hbase.regionserver.thread.compaction.large 和 
hbase.regionserver.thread.compaction.small 进行配置

#### 执行 HFile 文件合并

上文一方面选出了待合并的HFile集合，一方面也选出来了合适的处理线程，万事俱备，只欠最后真正的合并。

合并流程说起来也简单，主要分为如下几步:

1. 分别读出待合并hfile文件的KV，并顺序写到位于./tmp目录下的临时文件中

2. 将临时文件移动到对应region的数据目录

3. 将compaction的输入文件路径和输出文件路径封装为KV写入WAL日志，并打上compaction标记，最后强制执行sync

4. 将对应region数据目录下的compaction输入文件全部删除

上述四个步骤看起来简单，但实际是很严谨的，具有很强的容错性和完美的幂等性：

1. 如果RS在步骤2之前发生异常，本次compaction会被认为失败，如果继续进行同样的compaction，上次异常对接下来的compaction不会有任何影响，也不会对读写有任何影响。唯一的影响就是多了一份多余的数据。

2. 如果RS在步骤2之后、步骤3之前发生异常，同样的，仅仅会多一份冗余数据。

3. 如果在步骤3之后、步骤4之前发生异常，RS在重新打开region之后首先会从WAL中看到标有compaction的日志，因为此时输入文件和输出文件已经持久化到HDFS，因此只需要根据WAL移除掉compaction输入文件即可

#### Compaction 对读写请求的影响

##### 存储上的写入放大

HBase Compaction会带来写入放大，特别是在写多读少的场景下，写入放大就会比较明显，下图简单示意了写入放大的效果。

![](/images/blog/2019-09-18-3.png){:height="80%" width="80%"}

（图片来源：https://mmbiz.qpic.cn/mmbiz_png/licvxR9ib9M6D6sDjXPZxHR1ic4LDKyicf2qfx417fJ8QHmfn82uBhSS1fC4mDqSB67JHzGs6kyqHiccrQEu2ryPKJA/640）

随着minor compaction以及major Compaction的发生，可以看到，这条数据被反复读取/写入了多次，这是导致写放大的一个关键原因，这里的写放大，涉及到网络IO与磁盘IO，因为数据在HDFS中默认有三个副本。

##### 读路径上的延时毛刺

HBase执行compaction操作结果会使文件数基本稳定，进而IO Seek次数相对稳定，延迟就会稳定在一定范围。然而，compaction操作会带来很大的带宽压力以及短时间IO压力。因此compaction就是使用短时间的IO消耗以及带宽消耗换取后续查询的低延迟。这种短时间的压力就会造成读请求在延时上会有比较大的毛刺。下图是一张示意图，可见读请求延时有很大毛刺，但是总体趋势基本稳定。

![](/images/blog/2019-09-18-4.png){:height="80%" width="80%"}

##### 写请求上的短暂阻塞

Compaction 对写请求也会有比较大的影响。主要体现在HFile比较多的场景下，HBase会限制写请求的速度。

当写请求非常多，导致不断生成 HFile，compact的速度远远跟不上HFile生成的速度，这样就会使HFile的数量会越来越多，
导致读性能急剧下降。
 
为了避免这种情况，在HFile的数量过多的时候会限制写请求的速度，每次 MemStore 执行 flush 的操作前，会查看 HFile 数量，
如果数量超过hbase.hstore.blockingStoreFiles 配置值，默认10，flush操作将会受到阻塞，阻塞时间为hbase.hstore.blockingWaitTime，默认90000，即1.5分钟，在这段时间内，如果compaction操作使得HFile下降到blockingStoreFiles配置值，则停止阻塞。另外阻塞超过时间后，也会恢复执行flush操作。这样做可以有效地控制大量写请求的速度，但同时这也是影响写请求速度的主要原因之一。

Compaction 执行合并操作生成的文件生效过程，需要对 Store 的写操作加锁，阻塞Store内的更新操作，直到更新 Store 的storeFiles完成为止。   
Memstore 无法刷新磁盘，此时如果 memstore 的内存耗尽，客户端就会导致阻塞或者是超时。

#### 总结

在大多数情况下，Major 是发生在 storefiles 和 filesToCompact 文件个数相同，并且满足各种条件的前提下执行。

触发 major compaction 的可能条件有: major_compact 命令、majorCompact() API、region server自动运行。
（相关参数：hbase.hregion.majoucompaction 默认为24 小时、hbase.hregion.majorcompaction.jetter 默认值为 0.5 防止region server 在同一时间进行major compaction）

hbase.hregion.majorcompaction.jetter 参数的作用是：对参数hbase.hregion.majoucompaction 规定的值起到浮动的作用

minor compaction的运行机制要复杂一些，它由一下几个参数共同决定:

hbase.hstore.compaction.min: 默认值为 3，表示至少需要三个满足条件的store file时，minor compaction才会启动。

hbase.hstore.compaction.max: 默认值为10，表示一次minor compaction中最多选取10个store file

hbase.hstore.compaction.min.size: 表示文件大小小于该值的store file 一定会加入到minor compaction的store file中

hbase.hstore.compaction.max.size: 表示文件大小大于该值的store file 一定会被minor compaction排除

hbase.hstore.compaction.ratio: 将store file 按照文件年龄排序（older to younger），minor compaction总是从older store file开始选择，如果该文件的size 小于它后面hbase.hstore.compaction.max 个store file size 之和乘以 该ratio，则该store file 也将加入到minor compaction 中。

另外，一般情况下，Major Compaction 时间会持续比较长，整个过程会消耗大量系统资源，对上层业务有比较大的影响。因此线上业务都会将关闭自动触发 Major Compaction 功能，改为手动在业务低峰期触发。

---
参考链接
* [HBase Compaction的前生今世－身世之旅](http://hbasefly.com/2016/07/13/hbase-compaction-1/?xiruzu=vpafs3)
* [HBase Compaction算法之ExploringCompactionPolicy](https://my.oschina.net/u/220934/blog/363270)
* [hbase compaction 简单介绍](https://blog.csdn.net/zhoudetiankong/article/details/68924319)
* [深入分析HBase Compaction机制](https://blog.csdn.net/hljlzc2007/article/details/10980949#commentsedit)
* [HBase什么时候作minor major compact](https://www.cnblogs.com/cxzdy/p/5368715.html)
* [HBase自动大合并脚本](https://blog.csdn.net/Ancony_/article/details/84789142)
* [HBase在线系统性能优化](https://blog.csdn.net/wodatoucai/article/details/71171348)
* [HBase Region操作实战分析之Store Compaction](https://blog.csdn.net/zjumath/article/details/6955082)
* [hbase 定时进行compact CompactionChecker类](https://www.iteye.com/blog/blackproof-2192143)
* [Hbase Region Split compaction 过程分析以及调优](https://cloud.tencent.com/developer/article/1005586)
* [深入理解 HBase Compaction 机制](https://blog.csdn.net/u011598442/article/details/90632702)

