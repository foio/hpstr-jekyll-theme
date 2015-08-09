---
layout: post
title: javascript 内存泄露研究
description: "javascript 内存泄露研究"
modified: 2015-05-12
tags: [jquery javascript]
image:
  background: triangular.png
comments: true
---

这篇文章我们主要研究下，几种常见的内存泄露模式以及预防方法，最后说一下jquery容易造成内存泄露的地方。

###1.IE8以下版本js和dom循环引用造成的内存泄露

---

使用简单基于引用记数技术的GC机制，都会因为循环引用而造成内存泄露，早期的php(<=5.2)也会因为循环引用造成内存泄露。理论上说，现代浏览器一般使用基于标记清除的机制来回收内存。但是由于IE的dom结构使用Com组件实现，导致js的垃圾回收器无法回收dom造成的内存泄露。

如下代码在IE<=8中会造成内存泄露：

{% highlight javascript %}
function setHandler() {
  var elem = document.getElementById('id')
  elem.onclick = function() { /* ... */ }
}
{% endhighlight %}

由于有作用域链的存在，以上代码即便function为空，也会存在循环引用，会在
IE8以下版本的浏览器中造成内存泄露。


###2.IE浏览器的跨页面内存泄露

---

我们在动态创建dom片段结构并添加到dom时也会出现内存泄露，这种内存泄露，即使关闭页面也不会回收，只有当IE进程终止时才会回收。比如如下代码（其实这些代码应该很少出现在项目中，因为不会有人直接将事件绑定写成javascript内联函数），就会出现内存泄露：

{% highlight javascript %}
//dom结构必须绑定javascript事件函数才会由内存泄露问题
var parentDiv =
    document.createElement("<div onClick='foo()'>");
var childDiv =
    document.createElement("<div onClick='foo()'>");

// 这里会造成内存泄露
parentDiv.appendChild(childDiv);
hostElement.appendChild(parentDiv);
hostElement.removeChild(parentDiv);
parentDiv.removeChild(childDiv);
parentDiv = null;
childDiv = null;
{% endhighlight %}

这是因为，为临时父节点添加子节点时会创建临时的作用域变量，而将节点片段添加到根dom结构时，他们都会从根dom结构继承作用域，这就会导致临时作用域变量无法回收。避免这种内存泄露的最简单方法
是改变dom插入顺序，防止创建临时作用域链。

{% highlight javascript %}
var parentDiv =
    document.createElement("<div onClick='foo()'>");
var childDiv =
    document.createElement("<div onClick='foo()'>");

//改变了dom插入顺序，避免了临时作用域变量的创建，从而防止了内存泄露
hostElement.appendChild(parentDiv);
parentDiv.appendChild(childDiv);
hostElement.removeChild(parentDiv);
parentDiv.removeChild(childDiv);
parentDiv = null;
childDiv = null;
{% endhighlight %}

###3.IE9以下浏览器的XMLHttpRequest造成的内存泄露

---

以下常用的代码，就会找出内存泄露。

{% highlight javascript %}
var xhr = new XMLHttpRequest() // or ActiveX in older IE
xhr.open('GET', '/server.url', true)
xhr.onreadystatechange = function() {
  if(xhr.readyState == 4 && xhr.status == 200) {            
    // ...
  }
}
xhr.send(null)
{% endhighlight %}

当请求完成时，xhr变量引用为0 了，应该会被垃圾回收啊，但是浏览器自己也需要持有一个XMLHttpRequest的引用用于管理请求的状态，而浏览器没有能正确处理。


###4.闭包造成的内存泄露

---

Javascript解析器并不知道内嵌函数需要外部函数中的哪些局部变量，因此它会把外部函数中的全部变量都放在作用域对象中，这就会造成外层函数中的变量即便不使用也不会得到释放。如下代码：

{% highlight javascript %}
function f() {
  var data = "Large piece of data, probably received from server"
  function inner() {
      //inner中并没有使用data
  }
  return inner
}
{% endhighlight %}

一种处理方式是在返回内嵌函数时将外部函数中不需要的便利设置为null，如下：

{% highlight javascript %}
function f() {
  var data = "Large piece of data, probably received from server"
  function inner() {
      //inner中并没有使用data
  }
  data = null
  return inner
}
{% endhighlight %}

###5.jQuery是如何处理的，带来了什么新的问题

---

我们经常使用`$.data()`来给dom结构绑定数据属性，jQuery的事件处理机制也在内部使用了`$.data()`API，其实jQuery的内部实现是为每个dom生产一个id，然后将数据存储在jQuyer.cache对象中。

{% highlight javascript %}
//生成唯一id
elem[ jQuery.expando ] = id = ++jQuery.uuid
//将数据存储在cache对象中
jQuery.cache[id]['prop'] = val
{% endhighlight %}

这种方式避免了dom结构指向js对象，从而有效的防止了循环引用造成的内存泄露。看似完美，其实又带来了新的问题。以下代码存在内存泄露：

{% highlight javascript %}
$('<div/>')
  .html(new Array(1000).join('text'))
  .click(function() { })
  .appendTo('#data')
document.getElementById('data').innerHTML = ''
{% endhighlight %}

Dom结构已经被删除了，为什么还是会泄露内存呢？因为`jQuery.cache`中还保存这指向`function`的引用，而`function`中存在指向`div`片段的引用，这就导致`function`和`div`片段形成的整个闭包都无法被内存回收。

嗯，就到这了，肚子饿了，觅食去。

---
参考文档
[http://javascript.info/tutorial/memory-leaks#ie-lt-8-dom-js-memory-leak](http://javascript.info/tutorial/memory-leaks#ie-lt-8-dom-js-memory-leak)

