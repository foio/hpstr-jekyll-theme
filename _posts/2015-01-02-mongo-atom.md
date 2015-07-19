---
layout: post
title: 关于mongo原子操作的探讨
description: "关于mongo原子操作的探讨"
modified: 2014-01-02
tags: [mongo database]
image:
  background: triangular.png
comments: true
---

>众所周知，Redis采用的是异步I/O非阻塞的单进程模型，每一条Redis命令都是原子性的。那么mongoDB呢？ mongo有哪些原子操作呢？有哪些实现事务性操作的技巧呢？

###1.对单个文档的原子性修改

mongoDB保证了对单个document的多个filed的原子性修改。如果需要对单个文档进行原子性的CAS操作(check and set)，可以使用findAndModify操作。

比如下面就是一条原子性的CAS操作，首先选择_id为123的文档（注意这里只选择了一个文档），然后对计数器count加1，将status字段变为true，并返回修改后的结果。

{% highlight javascript%}
db.colleciton.findAndModify({query:{_id:'123'},$inc:{count:1},$update:{status:true}},new:true);
{% endhighlight %}

###2.对多个文档使用$isolate操作符

`$isolate`操作符可以对多个文档的修改提供隔离性。针对其他线程的并发写操作，`$isolate`保证了提交前其他线程无法修改对应的文档。针对其他线程的读操作，`$isolate`保证了其他线程读取不到未提交的数据。

但是`$isolate`有验证的性能问题，因为这种情况下线程持有锁的时间较长，严重的影响mongo的并发性。另外，`$isolate`也无法保证多个文档修改的一致性(all-or-nothing)，$isolate失败是可能只修改了部分文档。


###3.从语意层面实现事务性操作

mongoDB官方提供了一种做法，即两阶段提交(two-phase commit)，基本的原理就是利用了写操作的幂等性。具体实现可以看官网的详细讲解。但是利用幂等性来实现事务性有一个重要的前置条件：业务不在乎中间态的不一致。幂等性可以保证最终的一致性，但是会出现中间的不一致状态。

---

参考资料: (1) [mongoDB原子性与实务](http://docs.mongodb.org/manual/core/write-operations-atomicity/)
