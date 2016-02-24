---
layout: post
title: 如何实现一个MVVM框架
description: "实现MVVM框架, Model View ViewModel"
modified: 2016-02-24
tags: [javascript,MVVM]
image:
  background: triangular.png
comments: true
---

MVVM(Model View ViewModel)最初由微软在Windows Presentation Foundation（WPF）和Silverlight中引入，近年来、它作为MVC的一种替代方案在前端也如日中天。像其他MV*一样，MVVM中的Model代表着我们应用的数据；而View代表着用户界面；最重要的是ViewModel，可以将其看作一个拥有双向数据处理能力的转换器，它将模型数据传递到视图，并将视图指令传递到模型。MVVM框架将前端工程师从繁琐的DOM操作中彻底地解放出来，让我们可以更专注于自己的业务。

接下来我们探讨一种实现双向绑定的方案，本文适合实际使用过MVVM框架的人阅读，包括AngularJS、Avalon等。最终效果如下：

<a class="jsbin-embed" href="http://jsbin.com/vasaqekozi/embed?output">JS Bin on jsbin.com</a>

<script src="http://static.jsbin.com/js/embed.min.js?3.35.9"></script>

[详细源代码在请到github下载](https://github.com/foio/mvvm-demo)。

##1.基本功能

双向绑定作为MVVM框架的最大特点，是如何实现的呢？MVVM数据流示意图如下：

<figure style="width:80%;margin:0 auto">
	<img src="/images/mvvm.png" alt="mvvm overview">
</figure>

示意图中可以看出双向数据流：

```
View将变动通知到ViewModel，然后ViewModel对Model进行更新。
Model将变动通知到ViewModel，然后ViewModel对View进行更新。
```
其中最核心的功能是对视图(View)和模型(Model)变动的监听。

###(1).视图变动的监听

MVVM框架都是通过相应的指令，在HTML中声明式的标记出需要监听的DOM节点。本文实现中，我们主要涉及到两个指令：`foio-controller`、`foio-model`以及一个表达式\{\{\}\}。
比如：

``` html
<input type="text" foio-model="nickname">
``` 

上述指令foio-model，声明将View中的input的变动通知到Model中的nickname。通过对的视图节点(input)注册监听函数就可以得到视图(input)的变动了。

``` javascript
//对视图中的input节点注册input事件监听函数
var elem = document.querySelector('input');
if (elem.addEventListener) {
        elem.addEventListener('input', callback, false);
    } else {
     	elem.attachEvent('oninput', callback);
}
```

###(2).模型变动的监听

对模型变动的监听可以通过ECMAScript5中的API实现。

``` javascript
Object.defineProperty(obj, prop, descriptor)
```

可以通过该API为对象添加一个属性，并设置该属性的gett函数和set函数，在访问属性时会触发相应的get函数和set函数。

``` javascript
var air = {};
Object.defineProperty(air, 'temperature', {
    get: function() {
        console.log('get!');
    },
    set: function(value) {
        console.log('set!');
    }
});

air.temperature = 15; //output: set!
air.temperature;	//outpu: get!
```

我们可以在set函数中得到模型的变动，并将相关变动通知到ViewModel。

##2.总体实现

MVVM的主要流程包括(View)视图扫描、(Model)模型构建、以及关联视图和模型(ViewModel)

###(1)View(视图)扫描

处理View(视图)必然涉及到对DOM结构的扫描，通过扫描抽取指令(本文只有三种指令，foio-controller、foio-model、{{}})；并对相应的节点进行如下处理：

```
绑定通知函数，用于在视图更新时通知ViewModel
绑定更新函数，用于在模型更新时通过该函数更新视图
```

针对不同的节点类型，这些通知函数和更新函数都是预先定义好的，存储在`directives`结构中。在节点扫描过程中，当遇到指令时，就通过executeBindings函数对相应的节点进行绑定处理。流程图如下：

<figure style="width:80%;margin:0 auto">
	<img src="/images/mvvm-flow.png" alt="mvvm overview">
</figure>

###(2)Model(模型)构建

而对Model的处理也主要是注册监听函数，用于在Model变化时得到通知，如上图所示。controller中的每一个变量都通过`Object.defineProperty(obj, prop, descriptor)`定义到Model上，其中descriptor上的get函数可以用于搜集依赖，而set函数则用于通知依赖于该Model的视图进行更新。
 
``` javascript
var descriptor = {
	var dependencyList = [];
	get: function() {
			//搜集依赖
	        dependencyList.push(this);
	        return value;
	    },
	set: function(newVal) {
	        if (oldVal === newVal) {
	            return;
	        }
	        oldVal = newVal;
	        //通知依赖于该Model的视图进行更新
	        for (var dependIdx in dependencyList) {
	            dependencyList[dependIdx].updateView(newVal);
	        }
	    }
}
```

###(3)关联模型和视图

View(视图)扫描的结果是一个元素集合

``` javascript
bindings = [
				{
				    type: type, //指令类型
					element: elem, //DOM节点
				    expr: value, //绑定的变量名称
				},
				{...}
			]
```

而Model(模型)构建的结果也是一个集合：

``` javascript
vmodels = {
			controller1: {
				expr1: value1,
				expr2: value2,
				binder: {expr1: function(){},expr2:function(){}}
			},
			controller1: {...}
		}
```

通过executeBindings函数，将视图和模型关联起来。

``` javascript
function executeBindings(bindings, vmodels) {
	for (var i = 0, binding; (binding = bindings[i++]);) {
		binding.vmodels = vmodels;
		directives[binding.type](binding);
	};
}

```
每一种指令都有不同的初始化函数，比如针对`foio-model`指令，当DOM节点为input类型时，初始化函数做了三件事：

```
监听input和DOMAutoComplete事件
注册对模型的依赖
提供更新该DOM节点的方法
```

详细代码如下：

``` javascript
directives['model']={
 		switch (binding.xtype) {
            case "input":
            	//绑定input事件
                binding.bound('input', updateVModel);
                //绑定DOMAutoComplete事件
                binding.bound('DOMAutoComplete', updateVModel);
                //注册对模型的依赖
                elem.value = closetVmodel.binder[binding.expr].apply(binding);
                //更新该DOM节点的方法
                binding.updateView = function(newVal) {
                    elem.value = newVal;
                };
            break;
        }
}
```

至此我们实现了一个基本的MVVM框架了，虽然只有三个指令，但是基本能够说明如何设计并实现一个MVVM框架了。
