---
layout: post
title: mysql explain详解
description: "mysql explain"
modified: 2014-05-1
tags: [Photography]
image:
  background: triangular.png
---

>一个常见的理解错误：mysql在执行explain时不会执行sql语句，事实上如果查询的from字段有子查询，explain会执行子查询。

explain只能解释select查询，对update，delete，insert需要重写为select。常见的explain输入如下：

| id| select_type | table | type | possible_keys| key |key_len |ref|rows|Extra
| :------| --------:| --: |--: |--: |--: |--: |--: |--: |--: |
| 1  | SIMPLE |  articles   |ref|url_md5_idx|url_md5_idx|96|const|1|Using where

下面就explain的各个字段分别解释。

**1.id**
>当sql语句中有子查询和关联查询时会显示多列，id用于标志多列数据。
    
**2.select_type**
>用于表示是简单还是复杂的查询，不包括子查询和union的查询为简单查询。如果查询中有任何复杂的部分，外层查询标记为primary。复杂查询分为四大类(SUBQUERY,DERIVED,UNION,UNION RESULT)
```
(1)SUBQUERY:包含在select列表中的子查询中的select，不在from子句中的select
(2)DERIVED:表示包含在from子句中的select。mysql会递归的执行并将结果放在一个临时表中，服务器内部称其为“派生表”
(3)UNION:在union中第二个和随后的select被标记为union。
(4)UNION RESULT:用来在UNION产生的匿名临时表检索结果的select被标记为union result 综上,select_type共有SIMPLE,PRIMARY,SUBQUERY,DERIVED,UNION,UNION RESULT 六种常见情况。
```
**3.table**
>一般情况下为表名,当from子句中有子查询或者union时，table列会变得复杂的多，在这种情况下，mysql会创建匿名的临时表，这种情况下，table列为**derived N**的形式，其中N时子查询的id。当有UNION时，UNION RESULT的table列包含一个参与UNION的id列表，形为**union1,3**

**4.type**
>访问类型mysql决定如何查找表中的行，从最差到最优依次如下：
```
(1)ALL:全表扫描，通常意味着mysql必须扫描整张表，从头到尾去找到所需要的行。
(2):index：这和全表扫描一样，只是mysql在扫描表示按索引次序进行而不是行，他的主要优点是避免了排序，最大的缺点是承担按索引次数读取整张表的开销。如果Extra字段看到Using
(3):index，说明Mysql正在使用覆盖索引，他比按索引次序全表扫描开销要少得多。
(4)range：范围扫描就是一个有限制的索引扫描，它开始于索引的某一点，返回匹配这个值域的行。这比全索引扫描要好一些，因为它用不着遍历全部索引。 显而易见的范围扫描时带有between或者where >,当mysql使用索引去查找in()和or时也会显示range。但是这两者在性能上有很重要的差异。
(5)ref：这是一种索引访问（索引查找），它返回所有匹配某个单个值得行，然而它可能找到多个符合条件的行，因此它是查找和扫描的混合体。此类索引的扫描只有在使用非唯一索引或者唯一索引的非唯一前缀时才发生。
(6)eq_ref：使用这种索引查找，mysql最多只返回一条记录。这种访问方法在使用mysql主键或者唯一索引查找时看到。它会将他们与某个参考值作比较。
(7)const,system 当mysql能够从某部分进行优化将其转换为一个常量时，它就会使用这些访问类型。比如如下查询：explain select id from mis_audit_comment where id = 1\G;
(8)NULL 这种访问方式意味着Mysql能在优化阶段分解查询语句，在执行阶段甚至用不着访问表和索引。
```

**5.possible_key**
>显示查询可以使用的索引。

**6.key
>显示mysql决定使用哪个索引来优化对表的访问。

**7. key_len**
>mysql在索引里使用的字节数，可以根据key_len计算出该索引正在使用哪些列。可以根据key_len查看sql语句使用联合索引的情况。当有多列索引(audit_status,status,create_time)时，key_len为2时，表示只用了第一个为small int的索引。

**8.ref**
>显示table在key中选取的索引中查找值所用的列或者常量。

**9.row**
>mysql估计为了找到所需的行而要读取的行数。是mysql认为它要检查的行数，而不是结果集里的行数。

**10.Extra**
>记录了不适合在其他列中显示的额外信息
```
(1)Using index:mysql将使用覆盖索引，以避免访问表。
(2)Using where:mysql服务器将在存储引擎检索行后再进行过滤。
(3)Using temporary:mysql在对结果排序时使用了临时表
(4)Using filesort:表示mysql使用一个外部索引排序，而不是按索引的顺序读取表。
```