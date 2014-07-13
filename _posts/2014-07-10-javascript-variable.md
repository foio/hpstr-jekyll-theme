---
layout: post
title: javascript的声明提前
description: "javascript"
modified: 2014-07-10
tags: [javascript]
image:
  background: triangular.png
---

>众所周知，javascript有全局作用域和函数局部作用域(没有c，c++，java等强类型语言中的块作用域)。那下面一段代码的结果是什么呢？

{% highlight javascript %}
var scope = "global";
function f(){
  alter(scope);
  
  alter(scope);
}
f();
{% highlight  %}

这段代码的执行结果为：
{% highlight javascript %}
undefined
local
{% highlight  %}

输出local很容易理解，应用局部作用域的变量覆盖了全局作用域。但是输出undefined就有点奇怪了？

这是由于函数作用域的特性导致的，局部变量在整个函数体中始终有定义，也就是说，在整个函数体中局部变量遮挡了全局变量，但是只有程序执行到var语句时局部变量才会被赋值。因此，上述过程等价于将函数内的变量的声明“提前”到函数体顶部，而变量的初始化仍然保留在原来位置，也就做“声明提前”。因为php也只有全局作用域和函数局部作用域，因此php的表现和javascript是一致的。