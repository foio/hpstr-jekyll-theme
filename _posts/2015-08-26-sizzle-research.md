---
layout: post
title: sizzle引擎研究
description: "sizzle引擎研究"
modified: 2015-08-26
tags: [jquery javascript]
image:
  background: triangular.png
comments: true
---

什么是sizzle？下面时官方的一段解释。

```
A pure-javascript CSS selector engine
× Standalone(no dependencies)
× Competitive performance
× Only 4kB with gzipped
× Easy to use
× Css3 support
× Bla bla ……
```

其实说白了，sizzle就是一个很给力的选择器解析引擎。

我们为什么需要sizzle呢？其实对现代浏览器来说，document.querySelectorAll就可以解决一切。比如zeptoJs就是用querySelectorAll进行选择器解析的，因为移动端所有浏览器都支撑querySelectorAll。但是对于低版本的IE(<=8)浏览器，不仅不支持querySelectorAll，连getElementById都有bug，因此自己用浏览器原生API解析选择器简直难上加难。好在sizzle引擎帮我们处理了一切。知其然，更要知其所以然。下面让我们看看sizzle引擎内部时如何实现的。

sizzle解析器的主要有以下几个工作步骤。

<figure>
		<img src="/images/sizzle-step.jpg"/>
</figure>

接下来我们就一次解析。为了简单，我们在接下来的文章中都使用选择器`div input[name=ttt]`作为例子。

###1.词法分析

---

词法分析是指我们将文本代码解析为一个个记号(token)，以便后续语法分析使用。
####(1) sizzle的token种类

```
分组(,):/^[\x20\t\r\n\f]*,[\x20\t\r\n\f]*/

层级关系( >+~):/^[\x20\t\r\n\f]*([>+~]|[\x20\t\r\n\f])[\x20\t\r\n\f]*/

单个元素处理:
		var characterEncoding = "(?:\\\\.|[\\w-]|[^\\x00-\\xa0])+"
        var ID = new RegExp("^#(" + characterEncoding + ")")
        var TAG = new RegExp( "^(" + characterEncoding.replace( "w", "w*" ) + ")" )
        var Class = new RegExp( "^\\.(" + characterEncoding + ")" )
```


####(2)从左到右扫描生产token集合

{% highlight javascript %}
//分组
  var rcomma = /^[\x20\t\r\n\f]*,[\x20\t\r\n\f]*/;
  //层级
  var rcombinators =           
 /^[\x20\t\r\n\f]*([>+~]|[\x20\t\r\n\f])[\x20\t\r\n\f]*/
  //选择器
  var TAG = /^((?:\\.|[\w*-]|[^\x00-\xa0])+)/;
  var matchExpr = {
      CLASS: /^\.((?:\\.|[\w-]|[^\x00-\xa0])+)/,
      TAG: /^((?:\\.|[\w*-]|[^\x00-\xa0])+)/
  };
  //扫描
  while (selector) {
      //分组
      if (match = rcomma.exec(selector)) {
          selector = selector.slice(match[0].length)
          groups.push((tokens = []));
      }
      //层级关系
      if ((match = rcombinators.exec(selector))) {
          matched = match.shift();
          tokens.push({
              value: matched,
              type: match[0].replace(rtrim, " ")
          });
          selector = selector.slice(matched.length);
      }
      //选择器
      for (type in matchExpr) {
          if ((match = matchExpr[type].exec(selector))) {
              matched = match.shift();
              tokens.push({
                  value: matched,
                  type: type,
                  matches: match
              });
              selector = selector.slice(matched.length);
          }
      }
  }
{% endhighlight %}

最终生成的token集合如下:

{% highlight javascript %}
   {matches: ["div"],type: "TAG",value: "div“ }, 
   {match:[“”], type: " ", value: " "},
   {matches: ["input"], type: "TAG", value: "input"}, 
   {matches: ["name"], type: "ATTR", value: "[name=ttt]"}
{% endhighlight %}

###2.过滤函数

---

sizzle针对每一种token都实现一个过滤函数，如下代码所示：
{% highlight javascript %}
//各种类型的token的过滤器，全部返回闭包函数
Expr.filter = {
    ATTR   : function (name, operator, check) {return closure}
    CHILD  : function (type, what, argument, first, last) {return closure}
    CLASS  : function (className) {return closure}
    ID     : function (id) {return closure}
    PSEUDO : function (pseudo, argument) {return closure}
    TAG    : function (nodeNameSelector) { return function(elem) {
	return elem.nodeName && elem.nodeName.toLowerCase() === nodeNameSelector;
              };
     }
}
{% endhighlight %}

通过过滤函数我们可以.....


tired，休息去，明天继续

---
[为了便于理解本文，请下载本文对应的ppt](/download/sizzle-presentation.pptx)

