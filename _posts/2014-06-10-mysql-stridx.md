---
layout: post
title: mysql字符串索引问题
description: "mysql字符串索引"
modified: 2014-06-10
tags: [mysql]
image:
  background: triangular.png
comments: true
---

事情的起因是线上日志发现的mysql慢查询。100万数据量的标准，联合查询全部走索引的情况下，尽然要600多毫秒。很不解，但是将索引列由varchar(50)型改为bigint型后，数据提升了30倍。究其原因就索引树上搜索时要进行大量的比较操作，而字符串的比较比整数的比较耗时的多。

所以建议一般情况下不要在字符串列建立索引，如果非要使用字符串索引，可以采用以下两种方法：

>1.只是用字符串的最左边n个字符建立索引，推荐n<=10;比如index left(address,8),但是需要知道前缀索引不能在order by中使用，也不能用在索引覆盖上。

>2.对字符串使用hash方法将字符串转化为整数，address_key=hashToInt(address)，对address_key建立索引，查询时可以用如下查询where address_key = hashToInt(‘beijing,china’) and address = ‘beijing,china’;
