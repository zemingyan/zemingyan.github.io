---
layout: post
title: Flink WordCount 程序
categories: [Flink]
description: Flink WordCount 简单程序入门
keywords: keyword1, keyword2
---

Mac 上搭建 Flink 1.8.0 环境并构建运行简单 WordCount 简单程序入门

---

#### 前言

18年关注到的 Flink。那是还懵懵懂懂参加了 Flink Forward in China 2018。

因岗位为 SRE 和公司暂无需要。一直没有认真入门过。

这次打算作为 19 年下半年计划正式入坑学习。

开始前建议先阅读下云邪[入门教程 5分钟从零构建第一个 Flink 应用](https://mp.weixin.qq.com/s/8jEMQuU1PsuZOYj0mTtztQ)

#### 准备工作

Flink 可以运行在 Linux、Mac 和 Windows 上，唯一的要求就是必须安装 Java 8 或以上版本。

可以通过发出以下命令来检查Java的正确安装

```
java -version
```

有 Java 8，输出将如下所示

``` 
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
```

Java 安装后去[官网](https://flink.apache.org/zh/downloads.html)下载 Flink，然后解压即可运行。这里以 flink-1.8.0 为例。

``` 
cd ~/Downloads/
tar xzf flink-1.8.0-bin-scala_2.12.tgz
cd flink-1.8.0
```

配置全局变量。FLINK_HOME 写你自己的 Flink 路径

``` 
vim ~/.bash_profile

# Flink HOME
FLINK_HOME=/Users/tu/Public/SoftWare/flink-1.8.0 
export PATH=$FLINK_HOME/bin:$PATH
```

对于 MacOS 用户，可以选择通过 homebrew 安装 Flink。一键安装就可以用，不需要配置全局变量
``` 
brew install apache-flink
```

检查安装
``` 
$ flink --version
Version: 1.8.0, Commit ID: 4caec0d  
```

在 Flink 目录下运行以下命令即可启动

```
[16:29:35] tu flink-1.8.0 $ ./bin/start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host lihuimindeMacBook-Pro.local.
Starting taskexecutor daemon on host lihuimindeMacBook-Pro.local.
```

接着可以打开浏览器访问 [http://localhost:8081/](http://localhost:8081/) 查看

![](/images/blog/2019-07-23-1.png){:height="80%" width="80%"} 

通过 jps 可以看到多出来两个JVM进程，运行的主类

``` 
[16:29:45] tu bin $ jps -l
9169 org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
9702 sun.tools.jps.Jps
9658 org.apache.flink.runtime.taskexecutor.TaskManagerRunner
```

#### 开发

集群启动后就可以开发 Flink 程序

使用 IDEA 新建一个 maven 项目

![](/images/blog/2019-07-23-2.png){:height="80%" width="80%"} 

如果没有看到 flink-quickstart-java 的话通过右上角的 Add Archetype 按钮来添加。

![](/images/blog/2019-07-23-3.png){:height="80%" width="80%"} 

创建一个 SocketTextStreamWordCount Java 文件，加入以下代码

```java
package cn.lihm.examples.streaming;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

/**
 * @author lihm
 * @date 2019-07-23 16:18
 * @description TODO
 */

public class SocketTextStreamWordCount {
    public static void main(String[] args) throws Exception {
        // the port to connect to
        final String hostname;
        final int port;
        try {
            final ParameterTool params = ParameterTool.fromArgs(args);
            port = params.getInt("port");
            hostname = params.get("hostname");
        } catch (Exception e) {
            System.err.println("USAGE: Please run 'SocketWindowWordCount --hostname <hostname> --port <port>'");
            return;
        }
        // set up the streaming execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 获取数据
        DataStreamSource<String> stream = env.socketTextStream(hostname, port);

        // 计数
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = stream.flatMap(new LineSplitter())
                .keyBy(0)
                .sum(1);

        sum.print();

        env.execute("Java WordCount from SocketTextStream Example");
    }

    public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) {
            String[] tokens = s.toLowerCase().split("\\W+");

            for (String token: tokens) {
                if (token.length() > 0) {
                    collector.collect(new Tuple2<String, Integer>(token, 1));
                }
            }
        }
    }
}

```

官网还有 Scala 版本的。自己学习尝试即可。这里不介绍了


#### 运行

接着进入工程目录，使用以下命令打包。

`mvn clean package -Dmaven.test.skip=true`

![](/images/blog/2019-07-23-4.png){:height="80%" width="80%"} 

然后开启监听 9000 端口

`nc -l 9000`

![](/images/blog/2019-07-23-5.png){:height="80%" width="80%"} 

最后进入 flink 安装目录 bin 下执行以下命令提交job任务。注意换成你自己项目的路径

``` 
flink run -c cn.lihm.examples.streaming.SocketTextStreamWordCount flink-learning-examples-1.0-SNAPSHOT.jar --port 9000 --hostname 127.0.0.1
```

![](/images/blog/2019-07-23-6.png){:height="80%" width="80%"} 

执行完上述命令后，可以在 webUI 中看到正在运行的程序

![](/images/blog/2019-07-23-7.png){:height="80%" width="80%"} 

可以在 nc 监听端口中输入 text

```
$ nc -l 9000
lorem ipsum
ipsum ipsum ipsum
bye
```

![](/images/blog/2019-07-23-8.png){:height="80%" width="80%"} 

该任务的输出在 flink 家目录下 log 下以 .out 结尾的文件下。   
通过 tail 命令看一下输出的文件，来观察统计结果。

![](/images/blog/2019-07-23-9.png){:height="80%" width="80%"} 

最后测试完可以关闭集群

``` 
./bin/stop-cluster.sh
```

#### 总结

整个过程下来还是有些成就感的。提高了自己动手能力。

---
参考链接
* [Flink 从 0 到 1 学习 —— Mac 上搭建 Flink 1.6.0 环境并构建运行简单程序入门](http://www.54tianzhisheng.cn/2018/09/18/flink-install/)
* [flink WordCount初体验](https://www.jianshu.com/p/c9b1081215e7)
* [本地安装教程](https://ci.apache.org/projects/flink/flink-docs-release-1.8/tutorials/local_setup.html)
* [入门教程 5分钟从零构建第一个 Flink 应用](https://mp.weixin.qq.com/s/8jEMQuU1PsuZOYj0mTtztQ)





