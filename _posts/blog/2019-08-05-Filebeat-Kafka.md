---
layout: post
title: Filebeat 采集日志到 Kafka 配置及使用
categories: [Hadoop]
description: Filebeat、Kakfa
keywords: Filebeat, Kafka
---

Filebeat 采集日志到 Kafka 配置及使用

---

#### 前言

想把 Hadoop 集群中的日志收集起来进行流式分析

#### Filebeat

日志采集器选择了 Filebeat 而不是Logstash，是由于 Logstash 是跑在 JVM 上面，资源消耗比较大，
后来作者用 GO 写了一个功能较少但是资源消耗也小的轻量级的 Agent 叫 Logstash-forwarder，
后来改名为FileBeat。

官网下载对应环境的[Filebeat](https://www.elastic.co/cn/downloads/beats/filebeat)

我下载的是 7.3.0 版本 LINUX 64-BIT

##### filebeat.yml 配置

最核心的部分在于FileBeat配置文件的配置，需要指定paths（日志文件路径）、hosts（kafka主机ip和端口）、
topic（kafka主题）、name（本机IP）、logging.level（filebeat日志级别）。更多参考官网说明[配置Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-howto-filebeat.html)

``` 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/hadoop-hdfs/lihm.log
  multiline:
    pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
    negate: true
    match: after

output.kafka:
  enabled: true
  hosts: ["127.0.0.1:9092"]
  topic: test

logging.level: error
name: 100.73.12.124
```

##### 常用运维指令

终端启动（退出终端或ctrl+c会退出运行）  
`./filebeat -e -c filebeat.yml`
      
以后台守护进程启动启动filebeats
`nohup ./filebeat -e -c filebeat.yml &`    

停止运行FileBeat进程
`ps -ef | grep filebeat`  
`Kill -9 线程号`

#### Kafka

一个典型的 Kafka 集群包含若干 Producer，若干 broker、若干 Consumer Group，以及一个 Zookeeper 集群。

官网下载 [kafka_2.12-2.3.0.tgz](http://kafka.apache.org/downloads)

参考自[安装kafka_2.12-2.0.0](https://blog.csdn.net/shubingzhuoxue/article/details/82868956)文章自行安装

因是个人测试，只部署了一台kafka机器。

#### 调试

首先肯定得在 Kafka 创建 filebeat.yml 配置的 Kakfa topic

`bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test`

创建完成 topic 后，FileBeat 就可以往 Kafka 传输日志。

通过两个步骤验证 Filebeat 的采集输送是否正常

##### 采集验证

终端执行命令，查看控制台输出，如果服务有异常会直接打印出来并自动停止服务。

`./filebeat -e -c filebeat.yml`

如果以后台守护进程启动则查看 nohup.out 文件有没有异常

##### 接收验证

Kafka集群控制台直接消费消息，验证接收到的日志信息

`bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning`

--from-beginning 表示从 Kafka 保存的最开头的 message 消费。如果想以当前为准就去掉该参数即可

#### 总结

这过程算踩坑过程，尤其是 filebeat.yml 配置踩了很多坑。

原理和思想方面的知识就自行搜索学习。这里是记录一个实操的过程

---
参考链接
* [基于Kafka+ELK搭建海量日志平台](https://mp.weixin.qq.com/s/RM5wLt8qtJjHX6lwWCIhnA)
* [filebeat合并多行日志示例](https://my.oschina.net/openplus/blog/1589846)
* [filebeat采集日志到kafka配置及使用](https://www.jianshu.com/p/229c01447e54)
* [Filebeat的下载（图文讲解）](https://www.cnblogs.com/zlslch/p/6621834.html)
* [filebeat配置日志记录（等级）](https://www.cnblogs.com/qinwengang/p/10982424.html)
* [Filebeat+ELK部署文档](https://www.cnblogs.com/liangyou666/p/9185274.html)

