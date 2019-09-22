---
layout: post
title: Impala 操作 Kudu 表报错影响使用
categories: [Impala]
description: Impala 操作 Kudu 表报错影响使用
keywords: Impala, Kudu
---

Impala 操作 Kudu 表报 Not authorized: unencrypted connections from publicly routable IPs are prohibited.

---

#### 前言

又到晚饭点时间。一如以往的在去吃晚饭的路上，此时企业微信弹窗闪了一下。

数据平台部的同事说他们内部的 CDH 集群 Impala 操作 Kudu 表报错了，寻求解决帮助。

那还能说，吃饭的速度都快了很多，就为了早点回来解决。

#### 问题解决

问题现场，执行 SQL 复现报错。

``` 
WARNINGS: TransmitData() to 100.106.38.12:27000 failed: Not authorized: Client connection negotiation failed: client connection to 100.106.38.12:27000: FATAL_UNAUTHORIZED: Not authorized: unencrypted connections from publicly routable IPs are prohibited. See --trusted_subnets flag for more information.: 100.106.40.23:32811
```

![](/images/blog/2019-07-24-2.png){:height="80%" width="80%"} 

定眼一看，这种报错没见过，翻译过来就是`致命的未授权：未授权：禁止来自公共可路由IP的未加密连接。`

没见过这种报错信息。看来只能寻求搜索引擎了。某度搜索没有找到想要的，还是谷歌给力，直接找到 cloudera 问题帖子。

[KUDU trusted subnets does not work ](https://community.cloudera.com/t5/Interactive-Short-cycle-SQL/KUDU-trusted-subnets-does-not-work/td-p/69176)

问题相似度极高。而且还是已经解决的帖子。看来有成功的希望啊。

![](/images/blog/2019-07-24-3.png){:height="80%" width="80%"} 

翻译过来就是 `Impala已经从5.15开始将krpc用于transmitdata（）rpc。因此，您也需要在IMPALA配置中以相同的方式配置“trusted_subnets”标志（它不会从kudu配置中获取）。`

这套集群是 CHD 6.1.0 的，看来很可能符合这个问题，且也说了是不会从 Kudu 配置中获取 trusted_subnets 标志。

这里就有个问题了。trusted_subnets 是啥？ 碰巧的是原提问者也在问题中说到这个

![](/images/blog/2019-07-24-8.png){:height="80%" width="80%"} 

我也不是很清楚，但是感觉这个很关键，去 Impala 配置和 Kudu 配置搜索了下这个关键字。

只在 Kudu 中搜索到了，Impala 未找到这个，也符合原文说过 IMPALA 需要以相同的方式配置 

![](/images/blog/2019-07-24-4.png){:height="80%" width="80%"} 

这时候脑海中有个困惑，trusted_subnets 在 Impala 中怎么配置呢？

我对比了下 Kudu 的配置位置，Kudu 是在代码段位置配置，我猜测 Impala 应该也是在这配置。

在 Impala 配置中搜索`代码块`出现好几个地方可以添加。

这时候我只能凭经验猜测 `Impala 命令行参数高级配置代码段`和`Impala 服务环境高级配置代码段`可能性大，不太可能是其他服务。

抱着试一试心态在这两个地方添加下方配置。

```
--trusted_subnets=0.0.0.0/0
--rpc-authentication=disabled
--rpc-encryption=disabled
```

保存时CM报`Impala 服务环境高级配置代码段`配置错误

![](/images/blog/2019-07-24-6.png){:height="80%" width="80%"} 

看来只能先去掉，去掉后没有报错。

![](/images/blog/2019-07-24-5.png){:height="80%" width="80%"} 

重启 Impala 后再执行发现问题解决。

![](/images/blog/2019-07-24-7.png){:height="80%" width="80%"} 

#### 总结

谷歌大法好啊，感谢前人栽树，后人乘凉，没有他们的，我也不能把问题解决。

这里中的为什么，我就没过多研究了，因为本人对 Kudu 了解不多，暂时放着。

---
参考链接
* [ KUDU trusted subnets does not work ](https://community.cloudera.com/t5/Interactive-Short-cycle-SQL/KUDU-trusted-subnets-does-not-work/td-p/69176)




