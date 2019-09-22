---
layout: post
title: ulimit 未生效
categories: [Linux]
description: ulimit 未生效
keywords: Linux, ulimit
---

ulimit 未生效

---

#### 前言

在安装启动 elasticsearch 时遇些问题。

`max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]`

根据搜索到的资料[elasticsearch启动常见的几个报错](https://www.jianshu.com/p/2285f1f8ec21)，我尝试切换到root用户，编辑limits.conf添加如下内容

``` 
vim /etc/security/limits.conf
* soft nofile 128000
* hard nofile 128000
```

然后又切回普通账号来启动ES，发现还是报错。。(ES6.x之后为了安全只能普通账号启动)

Orz，又开启了排查问题之路

#### 排查

在 root 账号下执行 `ulimit -n` 输出 128000

但是我在 普通账号下执行 `ulimit -n` 输出 65535

额。说明普通账号未生效，百思不解。

尝试依靠搜索引擎寻求帮助，找到[limits.conf不生效问题](https://blog.csdn.net/weixin_34331102/article/details/92789331)也没解决问题

寻求 Sa 同学帮忙之后了解到这个`/etc/security/limits.d/90-nofile.conf`文件设置会覆盖全局设置。

将该文件进行修改

``` 
vim /etc/security/limits.d/90-nofile.conf
* soft nofile 128000
* hard nofile 128000
```

`ulimit -n` 再查看发现已修改

完结、撒花






