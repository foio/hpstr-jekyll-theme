---
layout: post
title: mysql读写分离机制
description: "mysql r/w splitting"
modified: 2014-07-24
tags: [mysql]
image:
  background: triangular.png
---

Mysql是这个世界上最通用的数据库了，很多优点集于一身：开源、高效、免费等。但是对于生成实践中的高并发、高可用性、安全性等要求，单台mysql往往无法满足。

#master-slave
一般可以通过主从读写分离来提升数据库的并发能力。

常见的部署如下：

![enter image description here][1]

这种架构满足了生产环境的需要，但是却大大增加了开发复杂度，作为一名研发人员要在业务代码中考虑读写分离以及由此引发的更加复杂的事务操作，不得不说很糟糕。为了解决这种问题，可以引入中间层proxy。

-----

#mysql proxy

mysql proxy实现了mysql client和mysql server的标准通信协议。因此对研发人员来说使用mysql  proxy和使用单机的msyql没有区别，他们甚至感觉不到proxy的存在。mysql proxy实现了读写分离、连接池、负载均衡。
下图是一个简单的proxy示意图，其中关键点在于：


```
将写和事务性语句分发到master，将读分发的slave
```


![enter image description here][2]


  [1]: http://heylinux.com/wp-content/uploads/2011/06/mysql-master-salve-proxy.jpg
  [2]: http://jan.kneschke.de/assets/projects/mysql/mysql-proxy-types-trx-splitting.png