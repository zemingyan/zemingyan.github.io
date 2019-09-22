---
layout: post
title: Flink Watermarks
categories: [Flink]
description: Flink Watermarks
keywords: Flink, Watermarks
---

初学Flink，对Watermarks的一些理解和感悟

----

#### 重要的概念

Window: Window是处理无界流的关键，Windows将流拆分为一个个有限大小的buckets，可以可以在每一个buckets中进行计算

start_time,end_time: 当Window时时间窗口的时候，每个window都会有一个开始时间和结束时间（前开后闭），这个时间是系统时间

event-time: 事件发生时间，是事件发生所在设备的当地时间，比如一个点击事件的时间发生时间，是用户点击操作所在的手机或电脑的时间

Watermarks: 可以把他理解为一个水位线，这个Watermarks在不断的变化，一旦Watermarks大于了某个window的end_time，就会触发此window的计算，Watermarks就是用来触发window计算的

#### 处理乱序的数据流

什么是乱序呢？可以理解为数据到达的顺序和他的event-time排序不一致。导致这的原因有很多，比如延迟，消息积压，重试等等

因为Watermarks是用来触发window窗口计算的，我们可以根据事件的event-time，计算出Watermarks，并且设置一些延迟，给迟到的数据一些机会。

可以阅读[Flink事件时间处理和水印](https://blog.csdn.net/a6822342/article/details/78064815)有个认识

#### 生成 Timestamp 和 Watermark

请仔细阅读[Flink生成Timestamp和Watermark](https://www.jianshu.com/p/8c4a1861e49f)对应[官网](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/event_timestamps_watermarks.html)

#### Kafka consumer 时间戳提取/水位生成

自定义时间戳提取器/水位生成器，具体方法参见这里，然后按照下面的方式传递给consumer

``` 
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
 
FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());
 
DataStream<String> stream = env
    .addSource(myConsumer)
    .print();
```

CustomWatermarkEmitter 为自定义的时间戳提取器/水位生成器, 具体方法参见[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/event_timestamp_extractors.html)

在内部，Flink 会为每个Kafka分区都执行一个对应的assigner实例。一旦指定了这样的assigner，对于每条Kafka中的消息，extractTimestamp(T element, long previousElementTimestamp)方法会被调用来给消息分配时间戳，而getCurrentWatermark()方法（定时生成水位）或checkAndGetNextWatermark(T lastElement, long extractedTimestamp)方法(基于特定条件)会被调用以确定是否发送新的水位值。

请仔细阅读[Flink生成Timestamp和Watermark](https://www.jianshu.com/p/8c4a1861e49f)对应[官网](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/event_timestamps_watermarks.html)


---
参考链接
* [Apache-Flink深度解析-DataStream-Connectors之Kafka](https://cloud.tencent.com/developer/article/1406134)




