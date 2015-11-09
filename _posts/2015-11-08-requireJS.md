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

本篇文章我们分析一下RequreJS的结构。没有任何准备就看源代码往往是一头雾水，我也并不打算在本文详解RequireJS的源代码，源代码处理很多细节问题，反而会影响我们从宏观上理解RequreJS的机构。你可以先了解几种在网页中异步加载javascript的方案（可以参考[我的另一篇博客](http://foio.github.io/javascript-async/)），然后思考还会碰到什么问题。就我而言，我提出如下两个基本问题：

```
(1).RequireJS是如何异步加载js模块
(2).RequireJS是如何处理模块间的依赖关系的(包括如何处理循环依赖)
```

接下来我就来分别分析这两个问题。在开始之前，我们先看一下本文中javascript代码的层次结构。

<figure>
<img src="/images/require-structure.png" alt="require structure">
</figure>

从图中可以看出，模块加载系统的require和define的实现，主要借助于4个子函数(矩形表示)和3个基本数据结构(椭圆形表示)，其中蓝色椭圆为require和define之间共同使用的数据结构。下文主要围绕着这个结构图展开，阅读过程中请随时参阅此图。

---

1. RequireJS是如何异步加载js模块
---

如果你看过[这篇专门讲解如何异步加载javascript模块的文章](http://foio.github.io/javascript-async/)，就会发现RequireJS用的就是其中的`script dom element`的方法。下面是具体实现的伪代码。

``` javascript
//by foio.github.io
function loadJS(url,callback){
	//创建script节点
	var node = document.createElement("script");
	//监听脚本加载完成事件，处理浏览器的兼容性，针对符合W3C标准的浏览器和不符合W3C标准的浏览器分别监听不同的事件
	node.[W3C?"onload":"onreadystatechange"] = function(){
		if(callback){
			callback();
		}
	};

	//监听onerror事件处理javascript加载失败的情况
	node.onerror = function(){}

	node.src=url;
	head.insertBefore(node,head.firstChild);
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
####<1>.define函数定义的模块放在哪里？

每个define函数定义的模块都放在factorys里面，留待require函数执行时使用

``` javascript
//by foio.github.io
factorys = {
	...
	id : {
		deps: [],//模块的依赖	
		factory: callback,//模块的回调函数
	}
	...
};
```


####<2>.开始加载的模块放在哪里，如何标记模块的加载状态？

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


####<3>.正在加载但是还没有加载完成的模块id列表

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
req.define = function(name, deps, callback){
	//根据模块名获取模块的url
	var id = getScriptId(name);
	//如果模块没有注册，就将模块加入factorys列表
	if(!factorys[id]){
		factorys[id] = {
			deps: deps,
			callback: callback,	
			//延迟执行函数，在加载模块时调用，用于检测循环依赖问题
			delay: function(id){
				//检测到循环时抛出异常
				if(checkCycle(modules[id].deps,id)){
					throw new error("circular dependency detected for module:"+name);
				}
			}
		};
	}
}
```

循环依赖检测函数的实现有些小技巧，需要理解上文提到的`modules`基本结构。因为checkCycle函数做了如下假设：
所有依赖模块都已经存放在modules结构中了。所以我们需要注意checkClylce的调用时机。具体实现请参考本文代码。
 
``` javascript
//by foio.github.io
function checkCycle(deps,id){
	//检查id的所有依赖模块
	for(var depid in deps){
		//如果模块已经加载完成则不可能存在循环依赖
		//如果depid与id相等，则检测到循环依赖，否则递归的检测id与depid的依赖模块
		if(modules[id].state !== 2 &&　(depid == id || checkCycle(module[depid].deps,id))){
			return true;
		}
	}
	return false;
}
```

###(3). require函数的基本实现
require函数的实现是相当复杂的，我们先确立程序的基本框架，再逐步深入到具体细节。

``` javascript
//by foio.github.io
var require = function(deps, callback){
	var dn = 0, //依赖的模块数
		cn = 0;	//已经加载的依赖模块数
	deps.map(function(el){
		//依赖数量加1
		dn++;
		//如果模块还没开始加载，则递归的调用require加载
		if(!modules[el]){
			//如果模块还没定义，则抛出异常
			if(!factorys[el]){
				throw new error("undefined module:"+name);
			}
			//加载js	
			loadJS(el,function(){
				//加载完成后执行依赖检查，如果没有依赖就执行callback函数
				checkDeps();	
			});
			//标志模块已经开始加载
			modules[el] = {
				id: el, 
				state: 1,//模块的加载状态	
				deps:factorys[el].deps,//模块的依赖关系
				factory: factorys[el].callback,//模块的回调函数
				exportds: null,//本模块回调函数callback的返回结果，供依赖于该模块的其他模块使用
			};
			//模块开始加载时，放入加载队列，以便检测加载情况
			loadings.unshift(el);						
			//递归的调用require函数加载依赖模块
			require(factorys[el].deps,factorys[el].callback);
			//这时el模块的所有依赖都已经开始加载并存放在modules结构中了，可以检测循环依赖了
			factorys[el].delay();
		}

		if(modules[el].state == 2){
			cn++; //已经加载的依赖模块数加1
		}
	});
	//所有依赖全部加载完成
	if(dn == cn){
		fireFactory(id,args,factory);
	}
	//检测依赖情况
	checkDeps();
}
```

这就是require函数的基本逻辑框架了，代码中使用了loadJS()，checkDeps(), 
fireFactory()等函数。其中loadJS函数我们章节“RequireJS如何异步加载js模块”中已经实现了。
下面我们分别来看看checkDeps()和fireFactory()的实现。

checkDeps是用来检查加载队列loadings中模块的依赖情况的，
一个模块的依赖全部加载完成后，就调用fireFactory执行该模块的回函数。

``` javascript
//by foio.github.io
function checkDeps(){
	//遍历加载列表
	for(var i = loadings.length, id; id = loadings[--i]){
		var obj = modules[id], deps = obj.deps, allloaded = true;									
		//遍历每一个模块的加载
		for(var key in deps){
			//如果存在未加载完的模块，则退出内层循环
			if(modules[key].state !== 2){
				allloaded = false;
				break;
			}
		}

		//如果所有模块已经加载完成
		if(allloaded){
			loadings.splice(i,1); //从loadings列表中移除已经加载完成的模块							
			fireFactory(obj.id, obj.deps, obj.factory);
		}
	}		
}
```

fireFactory的具体逻辑见如下代码，它的工作时从各个依赖模块收集返回值，然后调用factory：

``` javascript
//by foio.github.io
function fireFactory(id,deps,factory){
	//遍历id模块的依赖，为factory准备参数
	for (var i = 0, array = [] , d; d = deps[i];) {
		array.push(modules[d].exports);
	};
	//在context对象上调用factory方法
	var ret = factory.apply(context,array);	
	//记录模块的返回结果，本模块的返回结果可能作为依赖该模块的其他模块的回调函数的参数
	if(ret != void 0){
		modules[id].exports = ret;
	}
	return ret;
}
```

这些内容是我用将近一周的业余时间的研究心得，希望对你有帮助。当然，这些代码都只是javascript加载系统的基本框架，如果有考虑疏忽的地方，还请你指正。我计划在最近两周给出javascript模块加载系统的一个完整的可用demo。

原则上本文是禁止转载的，但是如果你非要转载我也无法阻止。但是请各位转载时务必注明出处，尊重我的劳动成果。
