---
layout: post
title: kafka offset 机制
categories: [Kafka]
description: kafka offset 
keywords: kafka 
---

kafka offset 机制

---

#### 概念

什么是 offset？

offset 是 consumer position，Topic 的每个 Partition 都有各自的 offset

消费者需要自己保留一个 offset，从 kafka 获取消息时，只拉去当前 offset 以后的消息

Kafka 的 scala/java 版的 client 已经实现了这部分的逻辑

之前 offset 保存到 zookeeper 上，broker 存放 offset 是 kafka 从 0.9 版本开始，提供的新的消费方式
原因是zookeeper来存放，还是有许多弊端，不方便灵活控制，效率不高

#### offset 记录位置

kafka 消费者在会保存其消费的进度，也就是offset，存储的位置根据选用的 kafka api 不同而不同。具体可以参看
[kafka 消费者offset记录位置和方式](https://blog.csdn.net/u013063153/article/details/78122088)


#### offset 更新方式

- 自动提交，设置 enable.auto.commit=true，更新的频率根据参数【auto.commit.interval.ms】来定。这种方式也被称为【at most once】，
fetch 到消息后就可以更新offset，无论是否消费成功

- 手动提交，设置 enable.auto.commit=false，这种方式称为【at least once】。fetc h到消息后，等消费完成再调用方法【consumer.commitSync()】，
手动更新offset；如果消费失败，则 offset 也不会更新，此条消息会被重复消费一次

Kafka 提供 hight-level API 基本帮我们封装好管理和持久化 offset 了。所以不需要怎么管 offset。

Kafka默认实现了at least once语义

---
参考链接
* [Kafka的offset管理](https://blog.csdn.net/u012129558/article/details/80075270)
* [kafka 消费者offset记录位置和方式](https://blog.csdn.net/u013063153/article/details/78122088)
* [kafka系列-进阶篇之消息和offset存储](https://blog.csdn.net/camel84/article/details/82433075)
* [Kafka提交offset机制](https://www.cnblogs.com/FG123/p/10091599.html)










