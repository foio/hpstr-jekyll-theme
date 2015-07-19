---
layout: post
title: hive自定义函数
description: "hive自定义函数"
modified: 2014-12-10
tags: [hive hadoop]
image:
  background: triangular.png
comments: true
---

>Hive可以将类sql查询语句转换成Hadoop的map reduce任务，让熟悉关系型数据库的人也可以利用hadoop的强大并行计算能力。Hive提供了强大的内置函数支持，但是总有一些特殊情况，内置函数无法覆盖，这就要求我们对定义自己的函数。接下来我们通过一个例子看一下如何自定义hive函数。

###1. 自定义函数的实现

假设我们的关系型数据库中user表有一个status字段，代表着用户的活跃等级，取值为1~10，活跃度一次递增。现在我们要根据status字段将用户分为3个活跃度等级。Hive显然没有这种与业务逻辑强耦合的内置函数，但这不应该成为阻碍我们使用Hive的理由。下面的扩展函数就可以满足需求。

{% highlight java %}
package com.test.example;
import org.apache.hadoop.hive.ql.exec.Description;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;
public class UserStatus extends UDF {
   public Text evaluate(Text input) {
     if(input == null) return null;
     int status= Integer.parseInt(input.toString());
     if(status>= 1 && status<= 3){
         return new Text(String.valueOf(1));
     }else if(status>=4 && status<=7){
         return new Text(String.valueOf(2));
     }else if(status>=7 && status<=10){
         return new Text(String.valueOf(3));
     }
     return null;
   }
}
{% endhighlight %}

从上面的例子可以看出实现自定义的hive函数还是相当简单的。就是继承org.apache.hadoop.hive.ql.exec.UDF 并实现execute函数。

###2. 自定义函数的使用

定义为自定义函数后该如何使用呢？其实也是相关简单的。假设包含自定义函数的jar包为mydf.jar。

####(1).在hive shell中加载

首先加载jar包，并创建临时函数\

{% highlight java %}
%> hive
hive> ADD JAR /path/to/mydf.jar;
hive> create temporary function userStatus as 'com.test.example.UserStatus';
{% endhighlight %}

然后就可以直接使用了

{% highlight java %}
hive> select userStatus(4);
{% endhighlight %}

但是每次使用都要加载一次，太费劲了。有没有别的方法呢。


####(2).在.hiverc中加载

编辑home目录下的.hiverc文件，如果没有这个文件就新建一个。将加载jar包的命令写入.hiverc文件，启动hive shell时会自动执行.hiverc文件，不需要每个shell都load一遍。


---

参考资料: [hive内置函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF), [hive自定义函数demo](https://github.com/foio/hive-extension-examples)
