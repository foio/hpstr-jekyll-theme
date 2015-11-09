---
layout: post
title: jQuery的回调模块实现原理
description: "jQuery的回调实现原理"
modified: 2015-04-12
tags: [jquery javascript]
image:
  background: triangular.png
comments: true
---

jQuery的回调系统是jQuery的基本功能之一,用于支撑jQuery的`ajax()`方法的实现和对promise规范的实现。jQuery提供了`callback`方法用于实现各种复杂回调功能.

###1.基本用法以及特殊参数

---

下面是使用callback最简单的例子,其中`callback` 函数没有任何参数,其中`add`,`fire`,`remove` 的含义通过方法名可以立刻知道.

{% highlight javascript %}
var callbacks = $.Callbacks();
callbacks.add( fn1 );
callbacks.add( fn2 );
callbacks.fire( "bar!" );
callbacks.remove( fn2 );
{% endhighlight %}

当然作为基础模块的回调功能这么简单,显然是无法支撑jQuery其他模块的.`callback`函数可以传入由空白分隔的多个flag,使得callback用不同的表现.常用的flag如下(多个flag可以同时使用)

```
once:确保回调列表的函数只能被触发一次.
memory:当一个回调函数添加到回到列表时,立即使用上一次传递的历史参数(上次回调列表必须执行成功)调用回调函数.主要用于实现Deferred
unique:确保一个回调函数只添加一次.
stopOnFalse:当回调列表中的函数返回false时,不会继续调用列表中后续的函数.
```

其中较难理解的flag是memory, 上个demo先:

{% highlight javascript %}
var callbacks = $.Callbacks( "memory" );
callbacks.add( fn1 );
callbacks.fire( "foo" );
callbacks.add( fn2 );
callbacks.fire( "bar" );
callbacks.fire( "foobar" );
 
/*
output:
fn1 says:foo
fn2 says:foo //这个输出是memory标记的作用
fn1 says:bar
fn2 says:bar
fn1 says:foobar
fn2 says:foobar
*/
{% endhighlight %}

###2.源码分析(版本2.1.4)

---

jQuery的callback函数的实现在源码的`3066~3288`行,源码的基本结构为:

{% highlight javascript %}
jQuery.Callbacks = function(options) {
    //内部变量
    var // 用于记录上次调用fire时的参数,用于memory回调列表
		memory,
		// 标记列表是否被触发过
		fired,
		// 标记列表是否正在触发
		firing,
		// 第一个回调函数
		firingStart,
		// 回调函数列表的长度
		firingLength,
		// 正在回调的函数在列表中的下标
		firingIndex,
		// 回调函数列表
		list = [],
		// 用于可用户回调的数据,当flag没有once时stack为false
		stack = !options.once && [],
	//参数转换
    options = typeof options === "string" ?
        (optionsCache[options] || createOptions(options)) :
        jQuery.extend({}, options);
    //方法实现
    fire = function() {}
    self = {
        add: function() {},
        remove: function() {},
        has: function(fn) {},
        empty: function() {},
        disable: function() {},
        disabled: function() {},
        lock: function() {},
        locked: function() {},
        fireWith: function(context, args) {},
        fire: function() {},
        fired: function() {}
    };
    return self;
};
{% endhighlight %}

下面我们分别详细研究以下`fire,add,remove`等几个主要函数的实现.

该函数主要是对函数回调列表进行遍历执行,但让要考虑stopOnFalse和memory标记的影响.

####(1)fire函数的实现

{% highlight javascript %}
//self中fire只是简单的封装了下callback中的fire函数
self.fire: function() {
    //讲self对象传递给callback.fire函数
    self.fireWith(this, arguments);
    return this;
}
callback.fire = function(data) {
    //如果callback工厂方法的参数中有memory标记,则记住本次调用fire的参数
    memory = options.memory && data;
    fired = true;
    firingIndex = firingStart || 0;
    firingStart = 0;
    firingLength = list.length;
    firing = true;
    //循环遍历回调函数列表
    for (; list && firingIndex < firingLength; firingIndex++) {
        //使用callback对象调用回调函数,请参阅代码3217~3220行,如果设置了stopOnFalse标记则在函数执行返回false时,不再调用回调列表中的后续函数
        if (list[firingIndex].apply(data[0], data[1]) === false && options.stopOnFalse) {
            //由于函数回调列表没有执行完成,阻止add方法使用memory参数调用新添加的函数
            memory = false; // To prevent further calls using add
            break;
        }
    }
    //执行完成,设置标记
    firing = false;
    if (list) {
        if (stack) {
            if (stack.length) {
                fire(stack.shift());
            }
        } else if (memory) {
            list = [];
        } else {
            self.disable();
        }
    }
}
{% endhighlight %}

####(2)add函数的实现

add函数主要实现往回调列表中添加函数的功能,考虑了unique标记的影响,并实现了memory标记的功能.如果该回调列表正在在执行,则新加入的函数也会被调用.

{% highlight javascript %}
add: function() {
    if (list) {
        //记录当前回调列表的长度
        var start = list.length;
        //用一个立即执行函数实现add的主要逻辑
        (function add(args) {  
            //遍历参数
            jQuery.each(args, function(_, arg) {
                var type = jQuery.type(arg);
                //添加函数
                if (type === "function") {
	                //如果有unique标记,且回调列表中已经存在该函数,则不添加
                    if (!options.unique || !self.has(arg)) {
                        list.push(arg);
                    }
                } else if (arg && arg.length && type !== "string") {
                    //如果函数时数组则使用递归
                    add(arg);
                }
            });
        })(arguments);
        //如果当前正在执行回调列表,则让本次执行包括新添加的函数
        if (firing) {
            firingLength = list.length;
        } else if (memory) {//否则立即用上次fire的参数执行该回调函数
            firingStart = start;
            fire(memory);
        }
    }
    return this;
}
{% endhighlight %}

####3.remove函数的实现
remove函数的实现相对较为简单,其实就是将函数从回调列表中移除,但同时要考虑对正在执行的回调列表的影响
{% highlight javascript %}
remove: function() {
    if (list) {
        jQuery.each(arguments, function(_, arg) {
            var index;
            //移除回调列表中的函数,注意:while循环会将多个同样的函数全部移除
            while ((index = jQuery.inArray(arg, list, index)) > -1) {
	            //删除下标为index的函数
                list.splice(index, 1);
                //需要考虑移除函数对正在执行的回调函数列表的影响
                if (firing) {
	                //减小回调列表长度,防止数组越界
                    if (index <= firingLength) {
                        firingLength--;
                    }
                    //当前移除函数的下表在当前执行函数的下标之前,需要使当前执行下标减一
                    if (index <= firingIndex) {
                        firingIndex--;
                    }
                }
            }
        });
    }
    return this;
},
{% endhighlight %}

Ok,分析完这3个函数的源码后,我们应该能够完全明白jQuery时如何实现回调的,以及各个标记时如何起作用的.

---
[jQuery的callback方法文档](https://api.jquery.com/jQuery.Callbacks/)
