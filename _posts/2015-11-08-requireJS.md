---
layout: post
title: RequireJS结构分析，实现自己的模块加载系统
description: "javascript RequireJS"
modified: 2015-11-08
tags: [javascript  RequireJS]
image:
  background: triangular.png
comments: true
---

近年来，前端变的越来越重，页面中的javascript代码量上升了一个量级，为了便于维护和团队协作，模块化是必经之路。针对javascript模块化，业界逐渐产出两种方案AMD和CMD，它们有什么区别呢？看看大牛玉帛在知乎上的回答:[http://www.zhihu.com/question/20342350/answer/14828786](http://www.zhihu.com/question/20342350/answer/14828786)。

其中最重要的区别是AMD是提前执行，CMD是延迟执行。

``` javascript
//by foio.github.io
//CMD 推崇依赖就近，代表为seajs
define(function(require,exports,module){
	var a = require('./a');	
	a.doSomething();
	var b = require('./b');
	b.doSomething();
})

//AMD 推崇依赖前置，代表为requireJs
//by foio.github.io
define(['./a','./b'],function(a,b){
	a.doSomething();	
	b.doSomething();
});

```

本篇文章我们实现一个自己的requreJS,我并不打算在源码中处理浏览器兼容性等细节，过于关注细节，反而会影响我们从宏观上理解代码机构。本文的目的是实现一个基本可用的javascript模块加载系统，带领读者理解要实现一个模块加载系统需要的知识结构。了解几种在网页中异步加载javascript的方案（可以参考[我的另一篇博客](http://foio.github.io/javascript-async/)），对理解本文有很大帮助。我提出如下两个基本问题：

```
(1).RequireJS是如何异步加载js模块
(2).RequireJS是如何处理模块间的依赖关系的(包括如何处理循环依赖)
```

接下来我就来分别分析这两个问题。在开始之前，我们先看一下本文中javascript代码的层次结构。

<figure>
<img src="/images/require-structure.png" alt="require structure">
</figure>

从图中可以看出，模块加载系统的require和define的实现，主要借助于几个子函数(矩形表示)和两个基本数据结构(椭圆形表示)，其中蓝色矩形表示存在递归调用。下文主要围绕着这个结构图展开，阅读过程中请随时参阅此图。

---

1. RequireJS是如何异步加载js模块
---

如果你看过[这篇专门讲解如何异步加载javascript模块的文章](http://foio.github.io/javascript-async/)，就会发现RequireJS用的就是其中的`script dom element`的方法。下面是具体实现的伪代码。

``` javascript
//by foio.github.io
//foioRequireJS的js加载函数 
foioRequireJS.loadJS = function(url,callback){
    //创建script节点
    var node = document.createElement("script");
    node.type="text/javascript";
    //监听脚本加载完成事件，针对符合W3C标准的浏览器监听onload事件即可
    node.onload = function(){
        if(callback){
            callback();
        }
    };
    //监听onerror事件处理javascript加载失败的情况
    node.onerror = function(){
        throw Error('load script:'+url+" failed!"); 
    }
    node.src=url;
    //插入到head中
    var head = document.getElementsByTagName("head")[0];
    head.appendChild(node);
}
```

2. RequireJS如何按顺序加载模块
---

用过RequireJS都知道，它主要就是两个函数require和define。其中define用于定义模块，require用于执行模块。

``` javascript
//by foio.github.io
//c.js
define(id,['a','b'],function(a,b){
	return function(){
		a.doSomething();	
		b.doSomething();	
		//doSomething();
	}	
});

//logic.js
require(id,['c'],function(c){
	return function(){
		c.doSomething();
		//doSomething();
	}	
});

```

要保证javascript模块的执行顺序，首先必须组织好依赖关系。

###(1)组织依赖关系

为了组织RquireJS需要哪些数据结构呢？看起来无从下手，我们可以对问题进行拆分。

####<1>.模块放在哪里，如何标记模块的加载状态？

moudules存储了所有已经开始加载的模块，包括加载状态信息、依赖模块信息、模块的回调函数、以及回调函数callback返回的结果。

``` javascript
//by foio.github.io
modules = {
	...
	id:{
		state: 1,//模块的加载状态	
		deps:[],//模块的依赖关系
		factory: callback,//模块的回调函数
		exportds: {},//本模块回调函数callback的返回结果，供依赖于该模块的其他模块使用
	}	
	...	
}
```


####<2>.正在加载但是还没有加载完成的模块id列表

每个脚本加载完成事件onload触发时，都需要检查loading队列，确认哪些模块的依赖已经加载完成，是否可以执行

``` javascript
//by foio.github.io
loadings = [
	...
	id,
	...
]
```


###(2). define函数的基本实现

再次强调，本文的目的是理解结构，而不是具体实现，因此代码中不会考虑浏览器兼容性，也不会考虑逻辑的完整性。define函数主要目的
是将模块注册到factorys列表中，方便require可以找到。同时必须处理循环依赖问题。

``` javascript
//by foio.github.io
foioRequireJS.define = function(deps, callback){
    //根据模块名获取模块的url
    var id = foioRequireJS.getCurrentJs();
    //将依赖中的name转换为id，id其实是模块javascript文件的全路径
    var depsId = []; 
    deps.map(function(name){
        depsId.push(foioRequireJS.getScriptId(name));
    });
    //如果模块没有注册，就将模块加入modules列表
    if(!modules[id]){
            modules[id] = {
                id: id, 
                state: 1,//模块的加载状态   
                deps:depsId,//模块的依赖关系
                callback: callback,//模块的回调函数
                exports: null,//本模块回调函数callback的返回结果，供依赖于该模块的其他模块使用
                color: 0,
            };
    }
};

```

###(3). require函数的基本实现
require函数的实现是相当复杂的，我们先确立程序的基本框架，再逐步深入到具体细节。 其实require函数主要的逻辑就是将main模块放入modules和loadings队列。然后就开始调用loadDepsModule加载main模块的依赖模块。
下面我们来看loadDepsModule的实现。

``` javascript
//by foio.github.io
foioRequireJS.require = function(deps,callback){
    //获取主模块的id
    id = foioRequireJS.getCurrentJs();

    //将主模块main注册到modules中
    if(!modules[id]){

        //将主模块main依赖中的name转换为id，id其实是模块的对应javascript文件的全路径
        var depsId = []; 
        deps.map(function(name){
            depsId.push(foioRequireJS.getScriptId(name));
        });

        //将主模块main注册到modules列表中
        modules[id]  = {
            id: id, 
            state: 1,//模块的加载状态   
            deps:depsId,//模块的依赖关系
            callback: callback,//模块的回调函数
            exports: null,//本模块回调函数callback的返回结果，供依赖于该模块的其他模块使用
            color:0,
        };
        //这里为main入口函数，需要将它的id也加入loadings列表，以便触发回调
        loadings.unshift(id);                       
    }
    //加载依赖模块
    foioRequireJS.loadDepsModule(id);
}

```

可以说loadDepsModule是模块加载系统中最重要的函数了。 loadDepsModule函数主要是递归的加载一个模块的依赖模块，通过loadJS在dom结构中插入script元素来完成js文件的载入和执行。这里loadJS的callback函数也很值得研究。 每一个模块都是通过define函数定义的，由于callback函数在模块加载完成后才会执行,所以callback函数执行时模块已经存在于modules中了。相应的，我们也要将该模块放入loadings队列以便检查执行情况；同时递归的调用 loadDepsModule加载该模块的依赖模块。loadJS的在浏览器的onload事件触发时执行，这是整个模块加载系统的驱动力。


``` javascript
//by foio.github.io
foioRequireJS.loadDepsModule = function(id){
    //依次处理本模块的依赖关系
    modules[id].deps.map(function(el){
        //如果模块还没开始加载，则加载模块所在的js文件
        if(!modules[el]){
            foioRequireJS.loadJS(el,function(){
                //模块开始加载时，放入加载队列，以便检测加载情况
                loadings.unshift(el);                       
                //递归的调用loadModule函数加载依赖模块
                foioRequireJS.loadDepsModule(el);
                //加载完成后执行依赖检查，如果依赖全部加载完成就执行callback函数
                foioRequireJS.checkDeps();  
            });
        }
    });
}   

```

下面我们再来分析一下checkDeps函数，该函数在每次onload事件触发时执行，检查模块列表中是否已经有满足执行条件的模块，然后开始执行。checkDeps也有一个小技巧，就是当存在满足执行条件的模块时会触发一次递归，因为该模块执行完成后，可能使得依赖于该模块的其他模块也满足了执行条件。

``` javascript
//检测模块的依赖关系是否处理完毕，该函数在每一次js的onload事件都会触发一次
foioRequireJS.checkDeps = function(){
    //遍历加载列表
    for(var i = loadings.length, id; id = loadings[--i];){
        var obj = modules[id], deps = obj.deps, allloaded = true;                                   
        //遍历每一个模块的加载
        foioRequireJS.checkCycle(deps,id,colorbase++);
        for(var key in deps){
            //如果存在未加载完的模块，则退出内层循环
            if(!modules[deps[key]] || modules[deps[key]].state !== 2){
                allloaded = false;
                break;
            }
        }

        //如果所有模块已经加载完成
        if(allloaded){
            loadings.splice(i,1); //从loadings列表中移除已经加载完成的模块                          
            //执行模块的callback函数
            foioRequireJS.fireFactory(obj.id, obj.deps, obj.callback);
            //该模块执行完成后可能使其他模块也满足执行条件了，继续检查，直到没有模块满足allloaded条件
            foioRequireJS.checkDeps();
        }
    }       
}   
```

最后我们分析一下，具体的执行函数fireFactory。我们知道，无论是require函数还是define函数，都有一个参数列表，fireFactory首先处理的问题就是收集各个依赖模块的返回值，构建callback函数的参数列表；然后调用callback函数，同时记录模块的返回值，以便其他依赖于该模块的模块作为参数使用。

``` javascript
//fireFactory的工作是从各个依赖模块收集返回值，然后调用该模块的后调函数
foioRequireJS.fireFactory = function(id,deps,callback){
    var params = [];
    //遍历id模块的依赖，为calllback准备参数
    for (var i = 0, d; d = deps[i++];) {
        params.push(modules[d].exports);
    };
    //在context对象上调用callback方法
    var ret = callback.apply(global,params);    
    //记录模块的返回结果，本模块的返回结果可能作为依赖该模块的其他模块的回调函数的参数
    if(ret != void 0){
        modules[id].exports = ret;
    }
    modules[id].state = 2; //标志模块已经加载并执行完成
    return ret;
}
```

这些内容是我用将近一周的业余时间的研究心得，希望对你有帮助。当然，这些代码都只是javascript加载系统的基本框架，如果有考虑疏忽的地方，还请你指正。文中的代码只是片段，完整的代码在我的github：[https://github.com/foio/MyRequireJS](https://github.com/foio/MyRequireJS)

如果你认真的读到这里，我有理由相信你是一个对技术有追求的人，我和你是同类人。 本文是禁止转载的，希望我们互相尊重大家的劳动成果。
