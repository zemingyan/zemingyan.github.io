---
layout: post
title: Kafka server 配置参数
categories: [Kafka]
description: Kafka server 配置参数
keywords: Kafka
---

Kafka server 配置参数

---

#### 前言

Kafka 在不修改 server 配置参数的情况下，可以以默认的参数配置启动

但是在生产环境下需要修改部分参数以更好的提供服务

#### properties

``` 
# broker的名字
broker.id=1
num.network.threads=3
num.io.threads=8
# kafka存放数据的路径
log.dirs=/data/kafka/kafka-logs
# Zookeeper 连接
zookeeper.connect=
host.name=
advertised.host.name=
listeners=PLAINTEXT://:9092
# 客户端连接的端口
port=9092
# 可以接收的消息最大尺寸
message.max.bytes=10240000
socket.send.buffer.bytes=20480000
socket.receive.buffer.bytes=20480000
socket.request.max.bytes=1048576000
num.partitions=3
num.recovery.threads.per.data.dir=1
log.retention.hours=168
log.segment.bytes=1073741824
log.flush.interval.messages=10000
log.flush.interval.ms=60000
log.retention.check.interval.ms=300000
log.cleaner.enable=true
log.cleanup.policy=delete
zookeeper.connection.timeout.ms=6000
auto.create.topics.enable=false
delete.topic.enable=true
# server用来处理网络请求的网络线程数目
num.network.threads=32
# server用来处理请求的I/O线程的数目
num.io.threads=64
num.replica.fetchers=8
auto.leader.rebalance.enable=true
# 在网络线程停止读取新请求之前，可以排队等待I/O线程处理的最大请求个数
queued.max.requests=5000
replica.fetch.max.bytes=10240000
controller.socket.timeout.ms=30000
controlled.shutdown.enable=true
```

---
参考链接
* [kafka配置参数详解](https://www.cnblogs.com/gxc2015/p/9835837.html)