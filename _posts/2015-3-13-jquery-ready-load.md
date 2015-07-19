---
layout: post
title: jQuery的ready事件实现原理
description: "jQuery的ready事件实现原理"
modified: 2015-03-13
tags: [jquery javascript]
image:
  background: triangular.png
comments: true
---

大家在面试的过程中，经常会被问到一个问题：ready与load那一个先执行，那一个后执行？答案是ready先执行，load后执行。

DOM文档加载的步骤：
要想理解为什么ready先执行，load后执行就要先了解下DOM文档加载的步骤：

>(1) 解析HTML结构。
(2) 加载外部脚本和样式表文件。
(3) 解析并执行脚本代码。
(4) 构造HTML DOM模型。//ready
(5) 加载图片等外部文件。
(6) 页面加载完毕。//load

从上面的描述中大家应该已经理解了吧，ready在第（4）步完成之后就执行了，但是load要在第（6）步完成之后才执行。

所以有如下结论：
>ready与load的区别就在于资源文件的加载，ready构建了基本的DOM结构，所以对于代码来说应该越快加载越好。在一个高速浏览的时代，没人愿意等待答案。假如一个网站页面加载超过4秒，不好意思，你1/4的用户将面临着流失，所以对于框架来说用户体验是至关重要的，我们应该越早处理DOM越好，我们不需要等到图片资源都加载后才去处理框架的加载，图片资源过多load事件就会迟迟不会触发。


我们经常使如下jQuery语句,用于在页面dom结构刚刚加载完成时,执行代码.

{% highlight javascript %}
//普通方式
$(document).ready(function(){
	//your code here
});
//简写方式
$(function(){
	//your code here
});
{% endhighlight %}

那么jquery的ready函数是如何实现的呢? 它时如何保证ready在load之前执行的呢?

###1. 现代浏览器通过DOMContentLoaded事件实现

当HTML文档下载并解析完成以后，就会在document对象上触发DOMContentLoaded事件。这时，仅仅完成了HTML文档的解析（整张页面的DOM生成），所有外部资源（样式表、脚本、iframe等等）可能还没有下载结束。也就是说，这个事件比load事件，发生时间早得多。

因此可以通过如下方式实现jQuery的ready函数.

{% highlight javascript %}
document.addEventListener("DOMContentLoaded", function(event) {
  //your code here
});
{% endhighlight %}

但是对于IE8以及以下的浏览器,世界就没有那么友好了. 

###2. 低版本浏览器通过readystatechange事件实现?

写过AJAX原生代码的童鞋都知道, readystatechange用于表示XMLHttpRequest的状态发生改变,当readyState=2时表示AJAX执行成功. readystatechange也会在dom状态发生变化时触发.

因此可以通过如下方式实现jQuery的ready函数.

{% highlight  javascript %}
document.onreadystatechange = function () {
  if (document.readyState == "interactive" || document.readyState == "complete") {
    initApplication();
  }
}
{% endhighlight %}

但是readyState为interactive时, DOM结构并没有稳定, 此时依然会有脚本修改DOM元素,导致执行事件太早. readyState为complete时, 图片已经加载完毕, 所以complete虽然在window.onload前执行, 导致执行事件太晚.
让我们看看jQuery的源码时如何实现的.

###3. jQuery在低版本浏览器上是如何实现ready方法的
jQuery源码中使用了Diego Perini 在 2007 年的时候报告的一种检测 IE 是否加载完成的方式的方法. 该方法通过使用 doScroll 方法调用，可以看下这哥们的[博客](http://javascript.nwbox.com/IEContentLoaded/)。

原理就是对于 IE 在非 iframe 内时，只有不断地通过能否执行 doScroll 判断 DOM 是否加载完毕。在上述中间隔 50 毫秒尝试去执行 doScroll，注意，由于页面没有加载完成的时候，调用 doScroll 会导致异常，所以使用了 try -catch 来捕获异常。

{% highlight javascript %}
// 用于保证事件在onload之前执行
document.attachEvent( "onreadystatechange", completed );
// 如果是IE且不是iframe就通过不停的检查doScroll来判断dom结构是否ready
var top = false;
try {
    top = window.frameElement == null && document.documentElement;
} catch(e) {}
if ( top && top.doScroll ) {
    (function doScrollCheck() {
        if ( !jQuery.isReady ) {//ready方法没有执行过
            try {
                // 检查是否可以向左scroll滑动,当dom结构还没有解析完成时会抛出异常
                top.doScroll("left");
            } catch(e) {
	            //递归调用,直到当dom结构解析完成
                return setTimeout( doScrollCheck, 50 );
            }
            //没有发现异常,表示dom结构解析完成,删除之前绑定的onreadystatechange事件
            detach();

            //执行jQuery的ready方法
            jQuery.ready();
        }
    })();
}
{% endhighlight %}

当然,通过检查doScroll的方法也不是完美的, 这种实现的jQuery的ready方法执行时,readyState可能为interactive, 也可能为complete.  也即没法保证dom加载完成后立刻执行, 但是一定会在DOM结构稳定后, 且图片加载完毕前执行.

---

参考文档

[MDN里面关于Document.readyState的文档](https://developer.mozilla.org/en-US/docs/Web/API/Document/readyState), [一篇关于dom ready的博客](http://www.cnblogs.com/zhangziqiu/archive/2011/06/27/domready.html)
