---
layout: post
title: Cloudera Manager 官方 API
categories: [CDH]
description: Cloudera Manager 官方 API
keywords: CDH
---

Cloudera Manager API

---

#### 前言

很多时候，想通过 CM 获取些集群信息。可以利用到 Cloudera 提供的官方 [API](http://cloudera.github.io/cm_api/)

#### 介绍

Cloudera Manager的 REST API 允许您使用现有工具，并以编程方式管理您的Hadoop集群。该API在 Cloudera Express 和 Cloudera Enterprise 中均可用，并附带开源客户端库。

官方支持 Python 和 Java

![](/images/blog/2019-07-20-1.png){:height="80%" width="80%"} 

![](/images/blog/2019-07-20-2.png){:height="80%" width="80%"}

客户端 有 api 版本说明，cm5.15.0 对应 api19，cm6.2.0 对应 api32 版本。

根据 https://community.cloudera.com/t5/Cloudera-Manager-Installation/CM-Python-API/td-p/60338 提示

集群为5.7.1 ，使用 http://host:port/api/version 查阅之后为 v12。

还可以到 https://cloudera.github.io/cm_api/docs/releases/ 查询

```
# Python
api = ApiResource(cm_host, cm_port, version=12, username="admin", password="***")

# Java
public class ClouderaManagerUtil {
    public static void main(String[] args) {
        RootResourceV12 apiRoot = new ClouderaManagerClientBuilder()
                .withHost("100.73.12.13")
                .withUsernamePassword("admin", "admin")
                .build()
                .getRootV12();

        // 获取所有主机
        ApiHostList hosts = apiRoot.getHostsResource().readHosts(DataView.SUMMARY);
        Map<String, String> cmHost = new HashMap<String, String>();
        for (ApiHost apiHost : hosts.getHosts()) {
            cmHost.put(apiHost.getHostId(), apiHost.getIpAddress());
            System.out.println(apiHost.getHostId() + " " + apiHost.getIpAddress());
        }

        // 获取所有的集群
        ApiClusterList clusters = apiRoot.getClustersResource().readClusters(DataView.SUMMARY);
        for (
                ApiCluster apiCluster : clusters.getClusters()) {
            System.out.println(apiCluster.getName());
        }
    }
}
```

---
参考链接
* [cm_api.api_client.ApiException: (error 404)](https://blog.csdn.net/Leon_liuqinburen/article/details/82145355)







