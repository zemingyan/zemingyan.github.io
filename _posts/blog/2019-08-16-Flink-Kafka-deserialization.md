---
layout: post
title: Flink-Kafka 内置 Schemas
categories: [Flink]
description: 内置的常用消息格式的Schemas
keywords: Flink, Kafka
---

Apache Flink 内部提供了内置的常用消息格式的 Schemas

---

#### 前言

Flink 消费 Kafka 时，要对消息进行格式化

#### Schemas

Flink 有提供内置的 Schemas

先看 Flink 中初始化 Kafka 数据源代码，其中传入服务器名和 Topic 名就可以了

``` 
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
FlinkKafkaConsumer010<String> consumer = new FlinkKafkaConsumer010<>(
    "flink_test", new SimpleStringSchema(), props);
DataStream<String> stream = env.addSource(consumer);
```

上方 SimpleStringSchema 即为 Schemas。如果需要使用其他的，替换掉该参数，并把相应数据类型进行修改

如果想自己实现 Schema ，可以参看[Apache-Flink深度解析-DataStream-Connectors之Kafka](https://juejin.im/post/5c8fd4dd5188252d92095995#heading-8) Simple ETL 部分，
与 SimpleStringSchema 是一样的效果，只是自己实现的 Schema

如果类找不到，请添加依赖

``` 
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>1.8.0</version>
</dependency>
```

##### SimpleStringSchema

SimpleStringSchema 把 message 反序列化为字符串。如果 message 有键, 则忽略键

##### JSONDeserializationSchema

JSONDeserializationSchema 使用 jackson 将 message 反序列化为 json 格式的消息并返回 com.fasterxml.jackson.databind.node.ObjectNode 对象流。你可以使用 .get("property") 方法访问字段。再一次, 键被忽略。

##### JSONKeyValueDeserializationSchema

JSONKeyValueDeserializationSchema 与前一个非常类似，但处理带有json编码的键和值的消息

返回的 ObjectNode 包含如下字段:

key: 键中存在的所有字段
value: 所有的 message 字段
metadata（可选）: 暴露消息的 offset, partition 和 topic（将 true 传递给构造函数以获取元数据）

例如:

``` 
kafka-console-producer --broker-list localhost:9092 --topic json-topic \
    --property parse.key=true \
    --property key.separator=|
{"keyField1": 1, "keyField2": 2} | {"valueField1": 1, "valueField2" : {"foo": "bar"}
```

会被解码为

``` 
{
    "key":{"keyField1":1,"keyField2":2},
    "value":{"valueField1":1,"valueField2":{"foo":"bar"}},
    "metadata":{
        "offset":43,
        "topic":"json-topic",
        "partition":0
    }
}
```

如果消息本身就没有 key、metadata。JSONKeyValueDeserializationSchema 解析出来也是没有key、value

##### AvroDeserializationSchema

使用静态提供的模式读取使用 Avro 格式序列化的数据

可以从 Avro 生成的类（AvroDeserializationSchema.forSpecific（...））推断出模式，或者它可以与 GenericRecords 一起使用手动提供的模式（使用AvroDeserializationSchema.forGeneric（...））

##### TypeInformationSerializationSchema

TypeInformationSerializationSchema (and TypeInformationKeyValueSerializationSchema) 它基于Flink的TypeInformation创建模式。 如果数据由Flink写入和读取，这将非常有用


#### 总结

AvroDeserializationSchema 与 TypeInformationSerializationSchema 

还没接触过，这里就暂时记录下，如果下次有用到再回来记录

---
参考链接
* [Apache-Flink深度解析-DataStream-Connectors之Kafka](https://juejin.im/post/5c8fd4dd5188252d92095995#heading-5)
* [Flink-Kafka 内置的反序列化 schemas](https://ohmycloud.github.io/2019/05/15/apache-flink-built-in-deserialization-schemas-md/)
* [apache-flink内置反序列化模式](https://riptutorial.com/zh-CN/apache-flink/example/27995/%E5%86%85%E7%BD%AE%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%A8%A1%E5%BC%8F)
* [Flink + Kafka + JSON - java example](https://stackoverflow.com/questions/39300183/flink-kafka-json-java-example)

