---
layout: post
title: Hadoop API
categories: [Hadoop]
description: Hadoop API
keywords: Hadoop
---

记录自己工作中遇到的 Hadoop API。 可以当作字典来查找

---

#### 前言

记录自己工作中遇到的 Hadoop API。 可以当作字典来查找

#### HDSF

##### HDFS 获取目录大小

获取文件大小，在命令行上，使用 hadoop fs -du 命令可以，但是通过 Java API 怎么获取呢

最开始想到的是递归的方法，这个方法很慢，后来发现 FileSystem.getContentSummary 的方法。

参考自[java api获取hdfs目录大小](https://superlxw1234.iteye.com/blog/1514586)

``` 
fs.getContentSummary(new Path(PathSrc)).getLength();
```

##### 获取 HDFS 存储信息

FsStatus 对象提供了获取 hadoop 总空间、已使用空间、剩余空间 的方法。使用方式如下

``` 
FileSystem fileSystem = FileSystem.get(new URI(hdfsURL), configuration);
FsStatus fsStatus = fileSystem.getStatus();
long capacity = fsStatus.getCapacity(); //Configured Capacity
long remaining = fsStatus.getRemaining(); //DFS Remaining
long used = fsStatus.getUsed(); // DFS Used
```

##### 获取 NN 的 active 节点

是使用 Hadoop 源码提供的 HAUtil 工具类来做的

```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.hdfs.HAUtil;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.URI;

/**
 * @author lihm
 * @date 2019-07-26 09:48
 * @description TODO
 */

public class DemoApplication {

    public void getNameNodeAdress() throws Exception {
        Configuration conf = new Configuration();
        String nameservices = "testcluster";
        String[] namenodesAddr = {"massive-dataset-new-001:8020", "massive-dataset-new-008:8020"};
        String[] namenodes = {"nn1", "nn2"};
        conf.set("fs.defaultFS", "hdfs://" + nameservices);
        conf.set("dfs.nameservices", nameservices);
        conf.set("dfs.ha.namenodes." + nameservices, namenodes[0] + "," + namenodes[1]);
        conf.set("dfs.namenode.rpc-address." + nameservices + "." + namenodes[0], namenodesAddr[0]);
        conf.set("dfs.namenode.rpc-address." + nameservices + "." + namenodes[1], namenodesAddr[1]);
        conf.set("dfs.client.failover.proxy.provider." + nameservices, "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider");
        String hdfsRPCUrl = "hdfs://" + nameservices + ":" + 8020;

        FileSystem fs = null;
        try {
            fs = FileSystem.get(new URI(hdfsRPCUrl), conf);
            InetSocketAddress active = HAUtil.getAddressOfActive(fs);
            System.out.println(active.getHostString());
            // System.out.println("hdfs port:" + active.getPort());
            // InetAddress address = active.getAddress();
            // System.out.println("hdfs://" + address.getHostAddress() + ":"+ active.getPort()); 
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (fs != null) {
                    fs.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    public static void main(String[] args) {
        DemoApplication nn = new DemoApplication() ;
        try {
            nn.getNameNodeAdress();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

参考自[通过api获取NN的active节点](https://blog.csdn.net/xiaoyutongxue6/article/details/81120589)

