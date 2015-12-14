---
layout: post
title: angular路由正则表达式
description: "angular路由正则表达式"
modified: 2015-12-14
tags: [javascript]
image:
  background: triangular.png
comments: true
---


本文源于对stackoverflow上一道问题的思考:[如何在正则angular路由中使用正则表达式中的通配符*](http://stackoverflow.com/questions/14216565/how-to-have-wildcards-in-angular-routes)

在使用angular的过程中，我们经常需要通过不同的url定位到不通过的页面。

这时候我们就可以借助angular的routeProvider提供的when方法来实现，比如

``` javascript
 myApp.config(['$routeProvider', '$httpProvider',function($routeProvider,$httpProvider) {
     //配置路由
     $routeProvider.
         when('/tab1', {
         templateUrl: 'sections/tab1.html',
         controller: 'Tab1Ctrl',
     }).
         when('/tab2', {
         templateUrl: 'sections/tab2.html',
         controller: 'Tab2Ctrl',
     }).
         when('/tab3', {
         templateUrl: 'sections/tab3.html',
         controller: 'Tab3Ctrl',
     }).
         otherwise({
         redirectTo: '/tab1'
     });
```
when方法的原型是：when(path, route)，其中path

* path可以包括以冒号开头的命名组，比如/tab1/:item

* path可以包含以冒号开头并以星号结束的命名组，比如/tab1/:item*

* path可以包含以冒号开头并以问号结束可选的命名组，比如/tab1/:item?

比如，path为：`/color/:color/largecode/:largecode*\/edit`会匹配`/color/brown/largecode/code/with/slashes/edit`
会匹配：

```
color: brown
largecode: code/with/slashes.
```

但是如果我们要实现以`/tab1`开头的url全部路由到`/tab1`呢？比如`/tab1click`、`/tab1touch`等都路由到/tab1。用正则表达式描述就是：`/tab1.*`全部路由到`/tab1`，使用上述的when方法我们无能为力了。 于是有人出来叫喧说，angular的路由连正则表达式都不支持，实在太弱了，但是其实我们是有方法实现完全按正则进行路由分发地。angular有个$locationChangeStart事件，我们可以监听这个事件，修改location。下面的例子就可以实现以/`tab1`开头的url全部路由到`/tab1`，即`/tab1.*`全部路由到`/tab1`。

``` javascript
myApp.config()
.run(['$rootScope','$location',function($rootScope,$location){
    $rootScope.$on('$locationChangeStart',function(event,next,current){
        var path = /#\/(tab1).*/i.exec(next);
        if(path && path[1]){
            $location.path('/tab1');
        }
    });
}]);

```
在事件$locationChangeStart触发时，我们就对路由的变化有了全部的控制，各种正则匹配自然不在话下。
