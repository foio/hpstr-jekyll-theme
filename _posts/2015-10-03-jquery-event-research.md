---
layout: post
title: jQuery事件系统研究
description: "jQuery事件系统研究"
modified: 2015-10-03
tags: [jquery javascript]
image:
  background: triangular.png
comments: true
---

我们习惯理所当然的使用jQuery提供的高级事件API，jQuery帮我们处理了事件注册机制在各个浏览器间的兼容性，帮我们实现了高级的事件委托机制，大大的提高了我们的工作效率；然而我从来都不是一个知其然不知其所以然的工程师，你是否一样？如果你同样对其实现细节感兴趣，请仔细阅读本文。本文不涉及jQuery提供的自定义事件。

我将本文组织为如下结构：

```
1.使用原生API有什么问题
2.大牛时如何解决的
3.jQuery的实现
```

---

###1. 使用原生API有什么问题


(1) 最原始的方法是直接对dom元素的onclick、onmouseover等属性赋值一个函数:


{% highlight javascript %}
Element.onclick = function(){
   //event handler logic
};
{% endhighlight %}


然而这种方式一次只能添加一个事件处理函数，不具有通用性。

(2) 对于符合最新W3C标准的浏览器(IE>8,chrome,firefox,android)可以使用element.addEventListener，将事件类型作为一个参数传递。

{% highlight javascript %}
element.addEventListener(‘click’,function(){
   //event handler logic
},false);
{% endhighlight %}

element.addEventListener是支持添加多个事件的，调用顺序是事件添加顺序。并且我们可以通过传递第三个参数来控制事件的触发时机是捕获阶段还是冒泡阶段，默认为冒泡阶段。然而低版本的IE(<=8)不支持element.addEventListener。

(3)对于低版本的IE(<=8)相应的提供了element.attachEvent:

{% highlight javascript %}
element.attachEvent(‘onclick’,function(){
        //event handler
});
{% endhighlight %}

element.attachEvent的事件类型参数格式必须是on+事件类型，但是监听函数内部的this指针指向的却不是触发事件的element。

由此可见，以上三种方法直接使用原生API的方法都是有问题的或者说不完美的。

---

###2.Dean Edwards大牛的实现
Dean Edwards在2005年写的了addevent的库，用于处理对浏览器的兼容性问题。理解了这个addEvent库，我们可以更好的理解jQuery的事件系统。

{% highlight javascript %}
//事件添加方法
addEvent.guid = 1;
function addEvent(element, type, handler) {
    // 为传入的每个事件初始化一个唯一的id
    if (!handler.$$guid) handler.$$guid = addEvent.guid++; //下文：addEvent.guid = 1;

    // 给element维护一个events属性，初始化为一个空对象。  
    // element.events的结构类似于 { "click": {...}, "dbclick": {...}, "change": {...} }  
    // 即element.events是一个对象，其中每个事件类型又会对应一个对象
    if (!element.events) element.events = {};

    // 试图取出element.events中当前事件类型type对应的对象，赋值给handlers
    var handlers = element.events[type];
    if (!handlers) {
    	handlers = element.events[type] = {};
        //如果handlers是undefined，则初始化为空对象
        // 如果这个element已经有了一个方法，例如已经有了onclick方法
        // 就把element的onclick方法赋值给handlers的0元素，此时handlers的结构就是：
        // { 0: function(e){...} }
        // 此时element.events的结构就是： { "click": { 0: function(e){...} },  /*省略其他事件类型*/ } 
        if (element["on" + type]) {
        	handlers[0] = element["on" + type];
        }
    }
    // 把当前的事件handler存放到handlers中，handler.$$guid = addEvent.guid++; addEvent.guid = 1; 肯定是从1开始累加的
    // 因此，这是handlers的结构就是 { 0: function(e){...}, 1: function(){}, 2: function(){} 等等... }
    handlers[handler.$$guid] = handler;
    // 下文定义了一个handleEvent(event)函数
    // 将这个函数，绑定到element的type事件上。  说明：在element进行click时，将会触发handleEvent函数，handleEvent函数将会查找element.events，并调用相应的函数。可以把handleEvent称为“主监听函数”
    element["on" + type] = handleEvent;
};


function handleEvent(event) {
    // 在IE中，event需要通过window.event获取
    event = event || window.event;
    // 根据事件类型在events中获取事件集合（events的数据结构，参考addEvent方法的注释）
    var handlers = this.events[event.type];
    // 注意！注意！  这里的this不是window，而是element对象，因为上文 element["on" + type] = handleEvent;
    // 所以在程序执行时，handleEvent已经作为了element的一个属性，它的作用域是element，即this === element

    // 循环执行handlers集合里的所有函数    另外，这里执行事件时传递的event，无论在什么浏览器下，都是正确的
    for (var i in handlers) {
        //此处为何要把handlers[i]赋值给this.$$handleEvent，然后在执行呢？
        this.$$handleEvent = handlers[i];
        this.$$handleEvent(event);
    }
};

{% endhighlight %}

Dean Edwards的这段代码还是比较好理解的，他首先将事件组织成一个如下结构，即每一种事件类型维持一个绑定事件处理函数列表，当某一类事件触发时，通过一个wrapper函数(通过onEvent API绑定的handleEvent)轮询对应列表并依次处理。事件列表结构组织如下：


{% highlight javascript %}
element: {
            onclick: handleEvent(event),   /*下文定义的函数*/
            events: {
                click:{
                    0: function(){...},    /*element已有的click事件*/
                    1: function(){...},
                    2: function(){...}
                    /*.......其他事件......*/
                },
                change:{
                    /*省略*/
                },
                dbclick:{
                    /*省略*/
                }
            }
}
{% endhighlight %}


如果理解了以上代码，那么jQuery的事件处理系统就容易懂了。


---

###3. jQuery的实现
其实用Dean Edwards的addEvent库就可以很好的实现了事件处理程序的健壮性：

```
(1).跨浏览器兼容性
(2).this变量的正确指向
(3).支持添加多个事件处理函数
```

但是jQuery的真正强大之处在于它对事件委托的实现，这也是我们接下来分析的要点，关于事件委托的介绍请参考我的[另一篇文章](http://foio.github.io/javascript-delegate/)。

对于事件绑定，jQuery提供了一个万能函数on，`$(selector).on(event,childSelector,data,function,map)`，基本上jQuery的各种事件函数变体都是对on函数的封装：
比如：`$(selector).click()，`

{% highlight javascript %}
jQuery.fn[ 'click' ] = function( data, fn ) {
    return arguments.length > 0 ?
         this.on( name, null, data, fn ) :
         this.trigger( name );
};
{% endhighlight %}

`$(selector).bind ()`


{% highlight javascript %}
bind: function( types, data, fn ) {
    return this.on( types, null, data, fn )
}
{% endhighlight %}


还有`$(selector).live()`，`$(selector).delegate()`，具体各个api的区别请参考jQuery文档。

####(1) 解析on函数

on函数内部调用了`jQuery.event.add`的add函数。


{% highlight javascript %}
$(selector).on(event,childSelector,data,function,map)
				||
			   \||/
			    \/
jQuery.event.add( this, types, fn, data, selector );

{% endhighlight %}

我们来分析以下add函数的代码：

{% highlight javascript %}
add: function( elem, types, handler, data, selector ) {
    //从jQuery私有缓存中获取该dom节点对应的事件元数据，这里需要简单了解以下jQuery的数据缓存系统，我们常常用$(selector).data()来从缓存中获取数据，相应的jQuery也有自己的私有缓存变量
    elemData = data_priv.get( elem );
    //该dom节点没有绑定过events类型的事件，则建立一个该类型事件的对象
    if ( !(events = elemData.events) ) {
       events = elemData.events = {};
   }
   //该节点没有绑定过事件，则绑定一个事件处理wrapper函数
   if ( !(eventHandle = elemData.handle) ) {
       //事件处理wrapper函数其实就简单调用了jQuery.event.dispatch.apply
        eventHandle = elemData.handle = function( e ) {
            jQuery.event.dispatch.apply( elem, arguments )
        }
    }
    //当参数中有多个选择器描述符时，循环遍历各个选择器，添加到对应的事件处理列表中
    types = ( types || "" ).match( rnotwhite ) || [ "" ];
    t = types.length;
    while ( t-- ) {
           //拼装事件处理元数据，dispatch就是通过这个元数据调用相应的handler函数的
            handleObj = jQuery.extend({
	            type: type,
	            origType: origType,
	            data: data,
	            handler: handler,
	            guid: handler.guid,
	            selector: selector,
	            needsContext: selector && jQuery.expr.match.needsContext.test( selector ),
	            namespace: namespaces.join(".")
            }, handleObjIn );
        //重点：如果selector参数不为空则表示事件委托，将事件委托添加到列表的前面，以保证优先处理。jquery事件处理的顺序时先委托事件其次正常事件
        if ( selector ) {
            handlers.splice( handlers.delegateCount++, 0, handleObj );
        } else {
            handlers.push( handleObj );
        }
    }
}
{% endhighlight %}

on函数主要建立事件元数据，并将元数据放到对应的列表里。最终生成如下事件数据结构：

{% highlight javascript %}
jQuery事件的数据结构：
elemData：{
      handle：function(){  //事件处理包装函数  }
      events:  {
            click:{
                delegateCount: n, //需要委托的事件数，需要委托的事件放在数据的最前面，通过delegateCount来标记
                [ //事件元数据，dispatch函数就是通过遍历这个数据结构来依次调用事件处理函数的
                    {
                      type: type,
                      origType: origType,
                      data: data,
                      handler: handler,
                      guid: handler.guid,
                      selector: selector,
                    }
                    ......
                    {}
                ]
              },
            mouseon:[],
            }
}
{% endhighlight %}

我们也看到了，on函数通过绑定一个wrapper函数来调用真实的事件处理函数，类似大牛Dean Edwards的实现。而wrapper函数中调用了dispatch函数，下面我们分析一下dispatch。


####(2) 解析dispatch函数

dispatch函数首先通过handlers函数对事件元数据队列进行预处理：委托元数据生成以及事件最终排序

```
jQuery.event.dispatch.apply( elem, arguments )
				||
			   \||/
			    \/
jQuery.event.handlers.call( this, event, handlers )
```


我们先看一下handlers函数的详细解析，handler函数主要是实现了委托的功能。首先从事件触发节点开始到事件委托节点结束，模拟冒泡过程，在冒泡的过程中遍历elemData中的委托事件元数据，选择匹配的节点，生成handlerQueue以供dispatch使用。其中handlerQueue的结果类似：`[{elem: node ..., handlers: this},...,{elem: node ..., handlers: list}]`。

{% highlight javascript %}
handlers: function( event, handlers ) {
  //触发事件的节点
  cur = event.target;
  //事件列表中委托事件的个数，目前事件列表中存放css选择器以及对应的事件处理函数
  delegateCount = handlers.delegateCount,
  //针对每一个触发事件，模拟冒泡过程，匹配到要代理节点，并放到匹配数组中，this指向委托节点，而cur指向事件触发节点，从事件触发节点到委托节点模拟冒泡过程，查找匹配需要委托选择器的节点。
  /×
   ×  注意：一个节点可能匹配多个选择器，相应的也会被触发多次，而模拟冒泡过程也保证了离event.target比较进的节点放在队列的最前面
   ×/
    for ( ; cur !== this; cur = cur.parentNode || this ){
         for ( i = 0; i < delegateCount; i++ ){
                //在冒泡过程中，匹配节点
                sel = handleObj.selector + " ";
                matches[ sel ] = handleObj.needsContext ?
                jQuery( sel, this ).index( cur ) >= 0 :
                jQuery.find( sel, this, null, [ cur ] ).length;
                //将匹配到的节点压入匹配数组
                matches.push( handleObj );
         }
         //对于代理节点
         if ( matches.length ) {
           //将匹配到的节点和对应的事件处理元数据列表放到handlerQueue队列中，以供dispatch使用，注意这里elem为匹配到的节点（要委托的节点），这是为了保证调用委托handler函数的内部this是正确的。
            handlerQueue.push({ elem: cur, handlers: matches });
         }
    }
    // 如果还有不需要代理的事件，则直接放入队列尾部
    if ( delegateCount < handlers.length ) {
        //注意这里的elem为this(代理节点)，这也保证了非委托handler内部this的正确指向
        handlerQueue.push({ elem: this, handlers: handlers.slice( delegateCount ) });
    }
}
{% endhighlight %}


理解了handlers函数的实现，dispatch函数的逻辑就简单了:
 
{% highlight javascript %}
dispatch: function( event ){
        //获取handlerQueue队列
        handlerQueue = jQuery.event.handlers.call( this, event, handlers );
        //对每一个需要触发的节点，执行相应的处理函数
        while ( (matched = handlerQueue[ i++ ]) && !event.isPropagationStopped() ) {
               while ( (handleObj = matched.handlers[ j++ ]) && !event.isImmediatePropagationStopped() ) {
                    //调用事件处理函数，这时已经保证了委托函数先调用的原则，同时保证了this指向的正确性
                    handleObj.handler.apply( matched.elem, args );
              }
        }
}
{% endhighlight %}

---
没错，现在我们基本分析完了jQuery事件系统的实现。可以说jQuery的实现参考了Dean Edwards的代码，同时添加了事件委托的实现。

本文我需要参考了如下博客
[http://www.cnblogs.com/lidabo/archive/2012/04/01/2429128.html](http://www.cnblogs.com/lidabo/archive/2012/04/01/2429128.html)
[http://www.cnblogs.com/wangfupeng1988/p/3659470.html](http://www.cnblogs.com/wangfupeng1988/p/3659470.html)



