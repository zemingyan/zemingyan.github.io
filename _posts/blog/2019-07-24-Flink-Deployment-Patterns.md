---
layout: post
title: Flink 部署模式
categories: [Flink]
description: 大数据计算引擎 Flink 的部署模式
keywords: Flink
---

大数据计算引擎 Flink 部署模式

---

#### 前言

大数据发展到今天，整个生态已经比较成熟比较丰富了。

在 Apache Flink China Meetup 上，把大数据的计算引擎分三代  

第一代无疑是 Hadoop 的 MapReduce，第二代是Spark，而对流处理和批处理统一性上做得很好的 Flink 被归作最新一代的计算引擎，也就是第三代大数据计算引擎。

![](/images/blog/2019-07-24-1.png){:height="80%" width="80%"} 

#### Local 本地部署

Flink 可以运行在 Linux、Mac OS X 和 Windows 上。

本地模式的安装唯一需要的只是 Java 1.8.x 及以上。   
本地运行会启动Single JVM，主要用于测试调试代码。

#### Standalone Cluster 集群部署

Flink 自带了集群模式 Standalone，对软件有些要求

1.安装Java1.8或者更高版本

2.集群各个节点需要ssh免密登录

#### Flink On Yarn

Apache Hadoop YARN是一个集群资源管理框架。它允许在群集之上运行各种分布式应用程序。

使用以下命令查看相关参数

`./bin/yarn-session.sh -h`

此命令将显示以下概述

``` 
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <property=value>             use value for given property
     -d,--detached                   If present, runs the job in detached mode
     -h,--help                       Help for the Yarn session CLI.
     -id,--applicationId <arg>       Attach to running YARN session
     -j,--jar <arg>                  Path to Flink jar file
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -m,--jobmanager <arg>           Address of the JobManager (master) to which to connect. Use this flag to connect to a different JobManager than the one specified in the configuration.
     -n,--container <arg>            Number of YARN container to allocate (=Number of Task Managers)
     -nl,--nodeLabel <arg>           Specify YARN node label for the YARN application
     -nm,--name <arg>                Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -sae,--shutdownOnAttachedExit   If the job is submitted in attached mode, perform a best-effort cluster shutdown when the CLI is terminated abruptly, e.g., in response to a user interrupt, such
                                     as typing Ctrl + C.
     -st,--streaming                 Start Flink in streaming mode
     -t,--ship <arg>                 Ship files in the specified directory (t for transfer)
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -yd,--yarndetached              If present, runs the job in detached mode (deprecated; use non-YARN specific option instead)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for high availability mode
```

通过命令 yarn-session.sh 来实现，本质上是在 Yarn 集群上启动一个 Flink 集群。

由 Yarn 预先给 Flink 集群分配若干个 container 给 flink 使用，在yarn的界面上只能看到一个 Flink session cluster 的任务。

![](/images/blog/2019-07-25-5.png){:height="80%" width="80%"}

只有一个 Flink 界面，可以从 Yarn 的  ApplicationMaster链接进入。

![](/images/blog/2019-07-25-6.png){:height="80%" width="80%"

这里需要注意一下如果没有提交 job, 只启动 yarn-session 的话, 打开 Flink 的 ui 界面是看不到上面箭头所指的资源数的。

![](/images/blog/2019-07-25-7.png){:height="80%" width="80%"}

只有提交一个job后, 才会显示。上图是提交了一个job, 当然还可以提交多个job到这个容器中运行。

**Flink 页面中能看到目前只启动了一个 TaskMananger（一个JVM进程），并且有 FreeSlot，新启动的 Flink Job 会在这些 slots 中启动，直到没有更多 FreeSlots 了才会分配新的 TaskMananger。**

##### 工作流程

![](/images/blog/2019-07-25-1.png){:height="80%" width="80%"}

Flink ON YARN工作流程如下所示

首先提交 job 给 YARN，就需要有一个 Flink YARN Client 

1. Client 首先检查要请求的资源(containers 和 memory)是否可用。然后将 Flink 相关 jar 包和配置文件上传到 HDFS。

2. Client 向 ResourceManager 申请一个 yarn container 用以启动 ApplicationMaster。由于客户端已经将配置文件和 jar 包注册为了 container 的资源，所以 NodeManager 会直接使用这些资源准备好 container（例如，下载文件等）。一旦该过程结束，AM就被启动了。

3. JobManager 和 ApplicationMaster 运行于同一个 Container。一旦创建成功，AM 就知道了 JobManager 的地址。它会生成一个新的 Flink 配置文件，这个配置文件是给将要启动的 TaskManager 用的，该配置文件也会上传到 HDFS。

4. ApplicationMaster 为 Flink 的 TaskManagers 分配 Container 并启动 TaskManager，TaskManager 内部会划分很多个Slot，它会自动从 HDFS 下载jar文件和修改后的配置，然后运行相应的 task。   
TaskManager也会与APPMaster中的JobManager进行交互，维持心跳等。

ApplicationMaster 的 Container 也提供了 Flink 的 web 接口。Yarn 代码申请的端口都是临时端口，目的是为了让用户并行启动多个 Flink YARN Session。

用 yarn session 在启动集群时，有2种方式可以进行集群启动分别是:

- 客户端模式
- 分离式模式

##### 客户端模式

默认可以直接执行`bin/yarn-session.sh`

默认启动的配置是`{masterMemoryMB=1024, taskManagerMemoryMB=1024, numberTaskManagers=1, slotsPerTaskManager=1}`

要自己自定义配置的话，自己可以根据参数来启动

示例: 发出以下命令以启动Yarn会话集群。
```
./bin/yarn-session.sh -n 2 -jm 1024 -tm 4096 -s 6
```

其中 JobManager 内存1GB，2个 TaskManager，每个 TaskManager 4GB内存和6个处理插槽启动。

-n: TaskManager的数量，相当于executor的数量

-s: 每个JobManager的core的数量，executor-cores。建议将slot的数量设置每台机器的处理器数量

-tm: 每个TaskManager的内存大小，executor-memory

-jm: JobManager的内存大小，driver-memory

对于客户端模式而言，可以启动多个 yarn session。   
一个 yarn session 模式对应一个 JobManager,并按照需求提交作业。同一个 Session 中可以提交多个Flink作业。

![](/images/blog/2019-07-25-4.png){:height="80%" width="80%"}

根据图中 Flink JobManager is now running 关键字可以知道 JobManager 位置

可以通过停止 unix 进程（使用CTRL + C）或在客户端输入“stop”来停止YARN会话。还可以通过yarn application -kill 命令来停止。

对于客户端模式进程如下:

在启动的 yarn-session 服务器上查看 Java 进程

![](/images/blog/2019-07-25-2.png){:height="80%" width="80%"}

在 JobManager 所在服务器查看 Java 进程

![](/images/blog/2019-07-25-3.png){:height="80%" width="80%"}

##### 分离模式

如果不希望Flink YARN客户端始终保持运行，则还可以启动分离的 YARN 会话。

对于分离式模式，并不像客户端那样可以启动多个 yarn session，如果启动多个，会出现后新建的 session一直处在等待状态。

JobManager的个数只能是一个，同一个 Session 中可以提交多个 Flink 作业。

客户端在启动 Flink Yarn Session 后，就不再属于 Yarn 集群的一部分。

分离模式启动命令上比客户端多一个参数。该参数称为 -d 或 --detached。-nm 给 yarn-session 起一个 test3 名字。

```
./bin/yarn-session.sh -n 2 -jm 1024 -tm 4096 -s 6 -d -nm test3
```

在这种情况下，Flink YARN 客户端将仅向群集提交 Flink，然后自行关闭。

请注意，在这种情况下，无法使用 Flink 停止YARN会话。使用YARN实用程序（yarn application -kill <appId>）来停止YARN会话。

![](/images/blog/2019-07-25-10.png){:height="80%" width="80%"}

分离模式 Java 进程只会有 YarnSessionClusterEntrypoint，而没有 FlinkYarnSessionCli，可以看到客户端模式和分离式模式的区别，除了进程外，其他都一样。

##### 附加到现有会话

假设现在在其他机器上启动了一个会话，但是这边又有需要附加到那个会话，这时就有作用了

``` 
Usage:
   Required
     -id,--applicationId <yarnAppId> YARN application Id
```

示例: 发出以下命令以附加到正在运行的Flink YARN会话application_1463870264508_0029：

``` 
./bin/yarn-session.sh -id application_1463870264508_0029
```

附加到正在运行的会话使用 YARN ResourceManager 来确定作业管理器RPC端口。

通过停止 unix 进程（使用CTRL + C）或在客户端输入“stop”来停止YARN会话。

#### 其他

系统默认使用 con/flink-conf.yaml 里的配置。

Flink on yarn将会覆盖掉几个参数:
- jobmanager.rpc.address 因为jobmanager的在集群的运行位置并不是实现确定的，前面也说到了就是am的地址
- taskmanager.tmp.dirs 使用yarn给定的临时目录
- parallelism.default也会被覆盖掉，如果在命令行里指定了slot数。

如果不想更改 conf/flink-conf.yaml 配置文件以设置配置参数，则可以选择通过-D标志传递动态属性。所以可以这样传递参数：-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624。

yarn-session 的 参数介绍
  -n: 指定TaskManager的数量;
  -d: 以分离模式运行;
  -id: 指定yarn的任务ID;
  -j: Flink jar文件的路径;
  -jm: JobManager容器的内存（默认值：MB）;
  -nl: 为YARN应用程序指定YARN节点标签;
  -nm: 在YARN上为应用程序设置自定义名称;
  -q: 显示可用的YARN资源（内存，内核）;
  -qu: 指定YARN队列;
  -s: 指定TaskManager中slot的数量;
  -st: 以流模式启动Flink;
  -tm: 每个TaskManager容器的内存（默认值：MB）;
  -z: 命名空间，用于为高可用性模式创建Zookeeper子路径;

**客户端模式与分离模式选用**

简单来说就是如果用分离式模式 ，那么在启动的时候会在yarn中常驻一个进程，并且已经确定了之后提交的job的内存等资源的大小，比如8G内存，如果某一个job把8G内存全部占完了，只能是第一个job执行完成把资源释放了，第二个job才能继续执行。  
如果是客户端模式，那么提交后，资源的大小是由yarn的队列所决定的，多个job提交，资源的占用和竞争都是由yarn所控制。

---
参考链接
* [大数据框架Flink的三大部署模式](https://baijiahao.baidu.com/s?id=1626634215130992812&wfr=spider&for=pc)
* [Flink on Yarn 有两种模式 分离模式 和 客户端模式](https://developer.aliyun.com/ask/133627?spm=a2c6h.13159736)
* [Flink on yarn应用部署](https://mp.weixin.qq.com/s/VCmeuzz-euCmu9ojZJ3uaA)
* [Fliin YARN设置](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/deployment/yarn_setup.html#flink-yarn-session)
* [Flink on yarn部署模式](https://www.jianshu.com/p/1b05202c4fb6)
