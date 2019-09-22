---
layout: post
title: HDFS checkpoint
categories: [HDFS]
description: HDFS checkpoint
keywords: HDFS checkpoint
---

HDFS checkpoint

---

#### 实现

StandbyCheckpointer内部维护一个CheckpointerThread线程，该线程负责周期性检查checkpoint条件是否满足，如果满足就进行checkpoint。

检测周期(checkpointPeriod)：1000*Math.min(dfs.namenode.checkpoint.period, dfs.namenode.checkpoint.check.period)秒

条件1：最近一次合并到namespace的edit log的 txid 和最近一次做了checkpoint的txid的差值大于或者等于dfs.namenode.checkpoint.txns配置的数量(默认1000000)

条件2：当前时间距离最近一次checkpoint的时间间隔大于或者等于dfs.namenode.checkpoint.period (默认3600秒)

CheckpointerThread每隔checkpointPeriod 秒检查一次。优先检查条件1是否满足，如果满足就进行checkpoint，否则检查条件2，如果条件2满足就进行checkpoint，否则就等待下一个检查周期。

#### 几个重要参数

参数名称 |	 说明	| 默认值
dfs.namenode.checkpoint.period | checkpoint的时间间隔	| 3600（秒）
dfs.namenode.checkpoint.check.period | 检查周期 | 60（秒）
dfs.namenode.checkpoint.txns | 两次checkpoint间的txid数量，超过该值就应该checkpoint | 1000000
dfs.image.compress | 是否压缩生成的fsimage文件 | false
dfs.image.compression.codec | 压缩格式 | org.apache.hadoop.io.compress.DefaultCodec