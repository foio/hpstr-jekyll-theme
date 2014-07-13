---
layout: post
title: javascript事件委托机制对比
description: " javascript事件委托机制"
modified: 2014-07-13
tags: [javascript]
image:
  background: triangular.png
---

随着DOM结构的复杂化和Ajax等动态脚本技术的运用，事件委托自然浮出了水面。jQuery为绑定和委托事件提供了.bind()、.live()和.delegate()方法。本文在讨论这几个方法内部实现的基础上，展示它们的优劣势及适用场合。

-----

##两个问题
####1.元素过多时的性能问题
假设有一个多行多列的表格，我们想让用户单击每个单元格都能看到与其中内容相关的更多信息（比如，通过提示条）。为此，可以为每个单元格都绑定click事件：

{% highlight javascript %}
$(“info_table td”).bind(“click”, function(){//TODO});
{% highlight  %}

问题是，如果表格中要绑定单击事件的有10列500行，那么查找和遍历5000个单元格会导致脚本执行速度明显变慢，而保存5000个td元素和相应的事件处理程序也会占用大量内存。


####2.动态添加元素的无法直接绑定事件
假设一个动态翻页的表格，每一次翻页都是重新插入的表格。通过给第一页添加事件，对后续通过js异步加载的页面没有作用。

----

##基本解决方法
事件委托可以解决上述两个问题。具体到代码上，只要用jQuery1.3新增的.live()方法代替.bind()方法即可：

{% highlight javascript %}
$(“#info_table td”).live(“click”,function(){//TODO});
{% highlight  %}

这里的.live()方法会把click事件绑定到$(document)对象，而且只需要给$(document)绑定一次，就能够处理后续动态加载表格。在接收到任何事件时，$(document)对象都会检查事件类型和事件目标，如果是click事件且事件目标是td，那么就执行委托给它的处理程序。

-----

##基本方法的问题

到目前为止，一切似乎很完美。可惜，事实并非如此。因为.live()方法并不完美，它有如下几个主要缺点：

```
1. $()函数会找到当前页面中的所有td元素并创建jQuery对象，但在确认事件目标时却不用这个td元素集合，而是使用选择符表达式与event.target或其祖先元素进行比较，因而生成这个jQuery对象会造成不必要的开销；收集td元素并创建jQuery对象，但实际操作的却是$(document)对象，令人费解。
2. 默认把事件绑定到$(document)元素，如果DOM嵌套结构很深，事件冒泡通过大量祖先元素会导致性能损失；
3. 只能放在直接选择的元素后面，不能在连缀的DOM遍历方法后面使用，即$(“#info_table td”).live…可以，但$(“#info_table”).find(“td”).live…不行；
```

jQuery从1.4开始支持在使用.live()方法时配合使用一个上下文参数：

{% highlight javascript %}
$(“td”,$(“#info_table”)[0]).live(“click”,function(){//TODO});
{% highlight  %}

这样，“受托方”就从默认的$(document)变成了$(“#info_table”)[0]，节省了冒泡的旅程。不过，与.live()共同使用的上下文参数必须是一个单独的DOM元素，所以这里指定上下文对象时使用的是$(“#info_table”)[0]，即使用数组的索引操作符来取得的一个DOM元素。

如前所述，为了突破单一.bind()方法的局限性，实现事件委托，jQuery1.3引入了.live()方法。后来，为解决“事件传播链”过长的问题，jQuery 1.4又支持为.live()方法指定上下文对象。而为了解决无谓生成元素集合的问题，jQuery 1.4.2干脆直接引入了一个新方法.delegate()。

-----

##更好的解决办法
使用.delegate()，前面的例子可以这样写：

{% highlight javascript %}
$(“#info_table”).delegate(“td”,”click”,function(){//TODO});
{% highlight  %}

使用.delegate()有如下优点：

```
1. 直接将目标元素选择符（”td”）、事件（”click”）及处理程序与“受拖方”$(“#info_table”)绑定，不额外收集元素、事件传播路径缩短、语义明确；
2. 支持在连缀的DOM遍历方法后面调用，即支持$(“table”).find(“#info”).delegate…，支持精确控制；
```

可见，.delegate()方法是一个相对完美的解决方案。但在DOM结构简单的情况下，也可以使用.live()。
>提示：使用事件委托时，如果注册到目标元素上的其他事件处理程序使用.stopPropagation()阻止了事件传播，那么事件委托就会失效。
