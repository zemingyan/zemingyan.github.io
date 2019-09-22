---
layout: post
title: HDFS checkpoint
categories: [HDFS]
description: HDFS checkpoint
keywords: HDFS
---

Hadoop 2.0 HA 的 checkpoint 过程

---

#### 前言

HDFS 将文件系统的元数据信息存放在 fsimage 和一系列的 edits 文件中。

在启动 HDFS 集群时，系统会先加载 fsimage，然后逐个执行所有Edits文件中的每一条操作，来获取完整的文件系统元数据。

#### 文件

HDFS 的存储元数据是由 fsimage 和 edits 文件组成。fsimage 存放上次 checkpoint 生成的文件系统元数据，edits 存放文件系统操作日志。checkpoint的过程，就是合并 fsimage 和 edits 文件，然后生成最新的 fsimage 的过程。

fsimage文件: fsimage 里保存的是 HDFS 文件系统的元数据信息。每次 checkpoint 的时候生成一个新的 fsimage 文件，fsimage 文件同步保存在 active namenode 上和 standby namenode 上。是在 standby namenode 上生成并上传到 active namenode 上的。

edits文件: active namenode 会及时把 HDFS 的修改信息（创建，修改，删除等）写入到本地目录，和 journalnode 上的 edits 文件，每一个操作以一条数据的形式存放。edits文件默认每2分钟产生一个。正在写入的Edits文件以 edits_inprogress_* 格式存在。

![](/images/blog/2019-09-20-1.png){:height="80%" width="80%"}

#### checkpoint 过程

开启HA的HDFS，有 active 和 standby namenode 两个 namenode 节点。他们的内存中保存了一样的集群元数据信息。

因为 standby namenode 已经将集群状态存储在内存中了，所以创建检查点checkpoint的过程只需要从内存中生成新的fsimage。

这里standby namenode称为SbNN，activenamenode称为ANN

1. SbNN查看是否满足创建检查点的条件
   * 距离上次checkpoint的时间间隔 >= ${dfs.namenode.checkpoint.period}
   * edits中的事务条数达到 ${dfs.namenode.checkpoint.txns} 限制
2. SbNN将内存中当前的状态保存成一个新的文件，命名为fsimage.ckpt_txid。其中txid是最后一个edit中的最后一条事务的ID（transaction ID）。然后为该fsimage文件创建一个MD5文件，并将fsimage文件重命名为fsimage_txid。
       
3. SbNN向ANN发送一条HTTP GET请求。请求中包含了SbNN的域名，端口以及新fsimage的txid。

4. ANN收到请求后，用获取到的信息反过来向SbNN再发送一条HTTP GET请求，获取新的fsimage文件。这个新的fsimage文件传输到ANN上后，也是先命名为fsimage.ckpt_txid，并为它创建一个MD5文件。然后再改名为fsimage_txid。fsimage过程完成。

#### checkpoint 相关配置

**dfs.namenode.checkpoint.period**

两次检查点创建之间的固定时间间隔，默认3600，即1小时

**dfs.namenode.checkpoint.txns**

未检查的事务数量。若没检查事务数达到这个值，也触发一次checkpoint，1,000,000

**dfs.namenode.checkpoint.check.period**

standby namenode检查是否满足建立checkpoint的条件的检查周期。默认60，即每1min检查一次

**dfs.namenode.num.checkpoints.retained**

在namenode上保存的fsimage的数目，超出的会被删除。默认保存2个

**dfs.namenode.num.checkpoints.retained**

最多能保存的edits文件个数，默认为1,000,000. 官方解释是为防止standby namenode宕机导致edits文件堆积的情况，设置的限制

**dfs.ha.tail-edits.period**

standby namenode每隔多长时间去检测新的Edits文件。只检测完成了的Edits， 不检测inprogress的文件。(不是很明白)

---
参考链接
* [A Guide to Checkpointing in Hadoop](https://blog.cloudera.com/a-guide-to-checkpointing-in-hadoop/)
* [Hadoop2.0 HA的checkpoint过程](https://blog.csdn.net/Amber_amber/article/details/47003589)


