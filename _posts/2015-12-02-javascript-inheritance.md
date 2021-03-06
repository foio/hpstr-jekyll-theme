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

``` javascript
(function(){
    //initializing用于控制类的初始化，非常巧妙，请留意下文中使用技巧
    //fnTest返回一个正则比表达式，用于检测函数中是否含有_super，这样就可以按需重写，提高效率。当然浏览器如果不支持的话就返回一个通用正则表达式
    var initializing = false,fnTest = /xyz/.test(function(){xyz;}) ? /\b_super\b/ : /.*/;
    //所有类的基类Class,这里的this一般是window对象
    this.Class = function(){};
    //对基类添加extend方法，用于从基类继承
    Class.extend = function(prop){
        //保存当前类的原型
        var _super = this.prototype;
        //创建当前类的对象，用于赋值给子类的prototype，这里非常巧妙的使用父类实例作为子类的原型，而且避免了父类的初始化(通过闭包作用域的initializing控制)
        initializing = true;
        var prototype = new this();     
        initializing = false;
        //将参数prop中赋值到prototype中，这里的prop中一般是包括init函数和其他函数的对象
        for(var name in prop){
            //对应重名函数，需要特殊处理，处理后可以在子函数中使用this._super()调用父类同名构造函数, 这里的fnTest很巧妙:只有子类中含有_super字样时才处理从写以提高效率
            prototype[name] = typeof prop[name] == "function" && typeof _super[name] == "function" && fnTest.test(prop[name])?
             (function(name,fn){
                return function(){
                    //_super在这里是我们的关键字，需要暂时存储一下
                    var tmp = this._super;  
                    //这里就可以通过this._super调用父类的构造函数了              
                    this._super = _super[name];
                    //调用子类函数  
                    fn.apply(this,arguments);
                    //复原_super，如果tmp为空就不需要复原了
                    tmp && (this._super = tmp);
                }
             })(name,prop[name]) : prop[name];
        }
        //当new一个对象时，实际上是调用该类原型上的init方法,注意通过new调用时传递的参数必须和init函数的参数一一对应
        function Class(){
            if(!initializing && this.init){
                this.init.apply(this,arguments);    
            }
        }       
        //给子类设置原型
        Class.prototype = prototype;
        //给子类设置构造函数
        Class.prototype.constructor = Class;
        //设置子类的extend方法，使得子类也可以通过extend方法被继承
        Class.extend = arguments.callee;
        return Class;
    }
})();
```


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
