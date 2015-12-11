---
layout: post
title: Javascript异步编程模型进化
description: "Javascript异步编程模型进化,promise,generator,co"
modified: 2015-12-10
tags: [javascript,promise,generator,co]
image:
  background: triangular.png
comments: true
---

Javascript语言是单线程的，没有复杂的同步互斥；但是，这并没有限制它的使用范围；相反，借助于Node，Javascript已经在某些场景下具备通吃前后端的能力了。近几年，多线程同步IO的模式已经在和单线程异步IO的模式的对决中败下阵来，Node也因此得名。接下来我们深入介绍一下Javascript的杀手锏，异步编程的发展历程。

让我们假设一个应用场景：一篇文章有10个章节，章节的数据是通过XHR异步请求的，章节必须按顺序显示。我们从这个问题出发，逐步探求从粗糙到优雅的解决方案。

---

##1.回忆往昔之callback
在那个年代，javascript仅限于前端的简单事件处理，这是异步编程的最基本模式了。
比如监听dom事件，在dom事件发生时触发相应的回调。

``` javascript
element.addEventListener('click',function(){
	//response to user click	
});
```

比如通过定时器执行异步任务。

``` javascript
setTimeout(function(){
	//do something 1s later	
}, 1000);
```

但是这种模式注定无法处理复杂的业务逻辑的。假设有N个异步任务，每一个任务必须在上一个任务完成后触发，于是就有了如下的代码，这就产生了回调黑洞。

``` javascript
doAsyncJob1(function(){
	doAsyncJob2(function(){
		doAsyncJob3(function(){
			doAsyncJob4(function(){
				//Black hole	
			});
		})	
	});
});
```

---

##2.活在当下之promise

针对上文的回调黑洞问题，有人提出了开源的promise/A+规范，具体规范见如下地址：[https://promisesaplus.com/](https://promisesaplus.com/)。promise代表了一个异步操作的结果，其状态必须符合下面几个要求：

```
一个Promise必须处在其中之一的状态：pending, fulfilled 或 rejected.
如果是pending状态,则promise可以转换到fulfilled或rejected状态。
如果是fulfilled状态,则promise不能转换成任何其它状态。
如果是rejected状态,则promise不能转换成任何其它状态。
```

###2.1 promise基本用法

promise有then方法，可以添加在异步操作到达fulfilled状态和rejected状态的处理函数。

``` javascript
promise.then(successHandler,failedHandler);
```
而then方法同时也会返回一个promise对象，这样我们就可以链式处理了。

``` javascript
promise.then(successHandler,failedHandler).then().then();
```

MDN上的一张图，比较清晰的描述了Pomise各个状态之间的转换。

<figure>
          <img src="/images/promises.png"/>
 </figure>

假设上文中的doAsyncJob都返回一个promise对象，那我们看看如何用promise处理回调黑洞：

``` javascript
doAsyncJob1().then(function(){
	return doAsyncJob2();;
}).then(function(){
	return doAsyncJob3();
}).then(function(){
	return doAsyncJob4(); 
}).then(//......);
```

这种编程方式是不是清爽多了。我们最经常使用的jQuery已经实现了promise规范,在调用$.ajax时可以写成这样了：

``` javascript
var options = {type:'GET',url:'the-url-to-get-data'};
$.ajax(options).then(function(data){
					//success handler
				},function(data){
					//failed handler
});
```

我们可以使用ES6的Promise的构造函数生成自己的promise对象，Promise构造函数的参数为一个函数，该函数接收两个函数(resolve,reject)作为参数，并在成功时调用resolve，失败时调用reject。如下代码生成一个拥有随机结果的promise。

``` javascript
var RandomPromiseJob = function(){
	return new Promise(function(resolve,reject){
		var res = Math.round(Math.random()*10)%2;	
		setTimeout(function(){
			if(res){
				resolve(res);
			}else{
				reject(res);
			}
		}, 1000)
	});		
}

RandomPromiseJob().then(function(data){
	console.log('success');
},function(data){
	console.log('failed');
});

```
jsfiddle演示地址：[http://jsfiddle.net/panrq4t7/](http://jsfiddle.net/panrq4t7/)

promise错误处理也十分灵活，在promise构造函数中发生异常时，会自动设置promise的状态为rejected，从而触发相应的函数。

``` javascript
new Promise(function(resolve,reject){
	resolve(JSON.parse('I am not json'));
}).then(undefined,function(data){
		console.log(data.message);
});
```

其中then(undefined,function(data)可以简写为catch。

``` javascript
new Promise(function(resolve,reject){
	resolve(JSON.parse('I am not json'));
}).catch(function(data){
		console.log(data.message);
});
```

jsfiddle演示地址：[http://jsfiddle.net/x696ysv2/](http://jsfiddle.net/x696ysv2/)

###2.2 一个更复杂的例子

promise的功能绝不仅限于上文这种小打小闹的应用。对于篇头提到的一篇文章10个章节异步请求，顺序展示的问题，如果使用回调处理章节之间的依赖逻辑，显然会产生回调黑洞; 而使用promise模式，则代码形式优雅而且逻辑清晰。假设我们有一个包含10个章节内容的数组，并有一个返回promise对象的getChaper函数：

``` javascript
var chapterStrs = [
  'chapter1','chapter2','chapter3','chapter4','chapter5',
  'chapter6','chapter7','chapter8','chapter9','chapter10',
];

var getChapter = function(chapterStr) {
  return get('<p>' + chapterStr + '</p>', Math.round(Math.random()*2));
};
```
下面我们探讨一下如何优雅高效的使用promise处理这个问题。

###(1). 顺序promise

顺序promise主要是通过对promise的then方法的链式调用产生的。

``` javascript
//按顺序请求章节数据并展示
chapterStrs.reduce(function(sequence, chapterStr) {
	return sequence.then(function() {
		return getChapter(chapterStr);
	}).then(function(chapter) {
		addToPage(chapter);
	});
}, Promise.resolve());
```

这种方法有一个问题，XHR请求是串行的，没有充分利用浏览器的并行性。网络请求timeline和显示效果图如下：
<figure>
          <img src="/images/sequence-promise.gif"/>
 </figure>
<figure style="width:340px;margin:0 auto">
          <img src="/images/sequence-promise-timeline.gif"/>
 </figure>
 
查看jsfiddle演示代码： [http://jsfiddle.net/81k9nv6x/1/](http://jsfiddle.net/81k9nv6x/1/)
 
###(2). 并发promise,一次性

Promise类有一个all方法，其接受一个promise数组:

``` javascript
Promise.all([promise1,promise2,...,promise10]).then(function(){
});
```

只有promise数组中的promise全部兑现，才会调用then方法。使用Promise.all，我们可以并发性的进行网络请求，并在所有请求返回后在集中进行数据展示。

``` javascript
//并发请求章节数据，一次性按顺序展示章节
Promise.all(chapterStrs.map(getChapter)).then(function(chapters){
	chapters.forEach(function(chapter){
		addToPage(chapter);
	});
});
```

这种方法也有一个问题，要等到所有数据加载完成后，才会一次性展示全部章节。效果图如下：

<figure>
          <img src="/images/parallel-promise.gif"/>
 </figure>
<figure style="width:340px;margin:0 auto">
          <img src="/images/parallel-promise-timeline.gif"/>
 </figure>

查看jsfiddle演示代码：[http://jsfiddle.net/7ops845a/](http://jsfiddle.net/7ops845a/)

###(3). 并发promise，渐进式

其实，我们可以做到并发的请求数据，尽快展示满足顺序条件的章节：即前面的章节展示后就可以展示当前章节，而不用等待后续章节的网络请求。基本思路是：先创建一批并行的promise，然后通过链式调用then方法控制展示顺序。

``` javascript
chapterStrs.map(getChapter).reduce(function(sequence, chapterStrPromise) {
    return sequence.then(function(){
			return chapterStrPromise;
    }).then(function(chapter){
      addToPage(chapter);
    });
  }, Promise.resolve());
```

效果如下：
<figure>
          <img src="/images/parallel-promise.gif"/>
 </figure>
<figure style="width:340px;margin:0 auto">
          <img src="/images/parallel-sequence-promise.gif"/>
 </figure>
 

查看jsfiddle演示代码：http://jsfiddle.net/fuog1ejg/

这三种模式基本上概括了使用Pormise控制并发的方式，你可以根据业务需求，确定各个任务之间的依赖关系，从而做出选择。

 
###2.3 promise的实现

ES6中已经实现了promise规范，在新版的浏览器和node中我们可以放心使用了。对于ES5及其以下版本，我们可以借助第三方库实现，q([https://github.com/kriskowal/q](https://github.com/kriskowal/q))是一个非常优秀的实现，angular使用的就是它，你可以放心使用。下一篇文章准备实现一个自己的promise。

---
##3.憧憬未来之generater

异步编程的一种解决方案叫做"协程"（coroutine），意思是多个线程互相协作，完成异步任务。随着ES6中对协程的支持，这种方案也逐渐进入人们的视野。Generator函数是协程在 ES6 的实现.

###3.1 Generator三大基本特性
让我们先从三个方面了解generator。

###(1) 控制权移交

在普通函数名前面加*号就可以生成generator函数，该函数返回一个指针，每一次调用next函数，就会移动该指针到下一个yield处，直到函数结尾。通过next函数就可以控制generator函数的执行。如下所示：

``` javascript
function *gen(){
	yield 'I';
	yield 'love';
	yield 'Javascript';
}

var g = gen();
console.log(g.next().value); //I 
console.log(g.next().value); //love
console.log(g.next().value); //Javascript
```

next函数返回一个对象{value:'love',done:false}，其中value表示yield返回值，done表示generator函数是否执行完成。这样写有点low？试试这种语法。

``` javascript
for(var v of gen()){
	console.log(v);
}
```

###(2) 分步数据传递

next()函数中可以传递参数，作为yield的返回值，传递到函数体内部。这里有点tricky，next参数作为上一次执行yeild的返回值。理解“上一次”很重要。

``` javascript
function* gen(x){
  var y = yield x + 1;
  yield y + 2;
  return 1;
}

var g = gen(1);
console.log(g.next()) // { value: 2, done: false }
console.log(g.next(2)) // { value: 4, done: true }
console.log(g.next()); //{ value: 1, done: true }
```

比如这里的g.next(2)，参数2为上一步yield x + 1 的返回值赋给y，从而我们就可以在接下来的代码中使用。这就是generator数据传递的基本方法了。


###(3) 异常传递

通过generator函数返回的指针，我们可以向函数内部传递异常，这也使得异步任务的异常处理机制得到保证。

``` javascript
function* gen(x){
  try {
    var y = yield x + 2;
  } catch (e){ 
    console.log(e);
  }
  return y;
}

var g = gen(1);
console.log(g.next()); //{ value: 3, done: false }
g.throw('error'); //error
```

###3.2 用generator实现异步操作

仍然使用本文中的getChapter方法，该方法返回一个promise，我们看一下如何使用generator处理异步回调。gen方法在执行到yield指令时返回的result.value是promise对象，然后我们通过next方法将promise的结果返回到gen函数中，作为addToPage的参数。

``` javascript
function *gen(){
	var result = yield getChapter('I love Javascript');
    addToPage(result);
}

var g = gen();
var result = g.next();
result.value.then(function(data){
    g.next(data);
});
```
gen函数的代码，和普通同步函数几乎没有区别，只是多了一条yield指令。

jsfiddle地址如下：[http://jsfiddle.net/fhnc07rq/3/](http://jsfiddle.net/fhnc07rq/3/)


###3.3 使用co进行规范化异步操作

虽然gen函数本身非常干净，只需要一条yield指令即可实现异步操作。但是我却需要一堆代码，用于控制gen函数、向gen函数传递参数。有没有更规范的方式呢？其实只需要将这些操作进行封装，co库为我们做了这些（[https://github.com/tj/co](https://github.com/tj/co)）。那么我们用generator和co实现上文的逐步加载10个章节数据的操作。

``` javascript
function *gen(){
	for(var i=0;i<chapterStrs.length;i++){
		addToPage(yield getChapter(chapterStrs[i]));
	}
}
co(gen);
```

jsfiddle演示地址：[http://jsfiddle.net/0hvtL6e9/](http://jsfiddle.net/0hvtL6e9/)


这种方法的效果类似于上文中提到“顺序promise”,我们能不能实现上文的“并发promise，渐进式”呢？代码如下：

``` javascript
function *gen(){
  var charperPromises = chapterStrs.map(getChapter);
  for(var i=0;i<charperPromises.length;i++){
		addToPage(yield charperPromises[i]);
	}
}
co(gen);
```

jsfiddle演示地址： [http://jsfiddle.net/gr6n3azz/1/](http://jsfiddle.net/gr6n3azz/1/)

经历过复杂性才能达到简单性。我们从最开始的回调黑洞到最终的generator，越来越复杂也越来越简单。

