---
layout: post
title: 系统的研究Javascript继承实现机制
description: "系统的研究Javascript继承实现机制，分析simple-inheritance"
modified: 2015-12-02
tags: [javascript,inheritance]
image:
  background: triangular.png
comments: true
---

老生常谈的问题，大部分人也不一定可以系统的理解。Javascript语言对继承实现的并不好，需要工程师自己去实现一套完整的继承机制。下面我们由浅入深的系统掌握使用javascript继承的技巧。

###1. 直接使用原型链

这是最简粗暴的一种方式，基本没法用于具体的项目中。一个简单的demo如下：

``` javascript
function SuperType(){
	this.property = true;
}
SuperType.prototype.getSuperValue = function(){
	return this.property;
}
function SubType(){
	this.subproperty = false;
}
//继承
SubType.prototype = new SuperType();
SubType.prototype.getSubValue = function(){
	return this.subproperty;
}
var instance = new SubType();
```

这种方式的问题是原型中的属性会被所用实例共享，这显然不是一种常规意义上的继承。


###2.使用构造函数

构造函数本质上也只是一个函数而已，可以在任何作用域中调用，在子构造函数中调用父构造函数，就可以实现简单的继承。

``` javascript
function SuperType(){
	this.colors = {"red","blue","green"}
}
function SubType(){
	SuperType.call(this);	
}
var instance = new SubType();
```

这种实现的问题太多了，比如没法共享函数，而且 `instance instanceof SuperType` 为false。

###3. 组合使用原型和构造函数

结合上述的两种方法，就可以实现共享函数并且隔离属性的目的了。

``` javascript
function SuperType(name){
	this.name = name;
	this.colors = {"red","blue","green"}
}
SuperType.prototype.sayName = function(){
	//code
}
function SubType(name,age){
	SuperType.call(this,name);	
	this.age = age;
}
SubType.prototype = new SuperType();
var instance = new SubTYpe();
```

组合使用原型和构造函数是javascript中最常用的继承模式。使用这种方式，每个实例都有自己的属性，同时可以共享原型中的方法。但是这种方式的缺点是：无论什么情况，都会调用两次超类构造函数。一次是在创建子类原型时，另一次是在子类构造函数内部。这种问题该怎么解决呢？

###4. 寄生组合式继承

SubType的原型并不一定非要是SuperType的实例。只需是一个构造函数的原型是SuperType的原型的普通对象就可以了。

Douglas Crockford的方法如下：

``` javascript
function obejct(o){
	function F(){};
	F.prototype = o;
	return new F();
}
```

其实这也就是ES5中Object.create的实现。那么我们可以修改本文中的第3种方案：

```  javascript
function inheritPrototype(subType,superType){
	var prototype = object(superType.prototype);
	prototype.constructor = subType;
	subType.prototype = prototype;
}
function SuperType(name){
	this.name = name;
	this.colors = {"red","blue","green"}
}
SuperType.prototype.sayName = function(){
	//code
}
function SubType(name,age){
	SuperType.call(this,name);	
	this.age = age;
}
inheritPrototype(SubType,SuperType);
var instance = new SubTYpe();
```

其实寄生组合式继承已经是一种非常好的继承实现机制了，足以应付日常使用。如果我们提出更高的要求：比如如何在子类中调用父类的方法呢？

###5.simple-inheritance库的实现

看这么难懂的代码，起初我是拒绝的，但是深入之后才发现大牛就是大牛，精妙思想无处不在。我对每一行代码都有详细的注释。如果你想了解细节，请务必详细研究，读懂每一行。

{% gist foio/c6673281384bf0c0bcb0 %}

通过使用simple-inheritance库，我们就可以通过很简单的方式实现继承了，是不是发现特别像强类型语言的继承。

``` javascript
var Human = Class.extend({
  init: function(age,name){
  	this.age = age;
  	this.name = name;
  },
  say: function(){
    console.log("I am a human");
  }
});
var Man = Human.extend({
	init: function(age,name,height){
		this._super(age,name);
		this.height = height;
	},	
	say: function(){
		this._super();
		console.log("I am a man");	
	}
});
var man = new Man(21,'bob','191');
man.say();
```