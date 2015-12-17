---
layout: post
title: 使用navigation timing统计网页加载速度
description: "navigation timing,统计网页加载速度,window.performance"
modified: 2015-12-17
tags: [javascript,promise,generator,co]
image:
  background: triangular.png
comments: true
---


前端页面加载速度的优化是一个系统性的工程，包括常见资源合并压缩、静态资源请求多域名化、CDN等。用户请求显示一个网页的详细过程也是十分复杂，包括DNS解析、建立TCP连接、加载HTML以及静态资源、渲染页面dom结构等多个阶段，任何一个阶段出现瓶颈都会影响用户体验，我们如何在真实的用户环境中准确的统计各个阶段的耗时呢？本文给出一个可行的方案。

借助于W3C的Navigation Timing API(目前大多数浏览器都已经支持，包括IE9以上版本)，该API通过window.performance提供页面加载相关的信息，我们这里只需要关注window.performance.timing。打开chrome开发工具的console标签，输入window.performance.timing，典型的输出如下：


```
connectEnd: 1450349863256
connectStart: 1450349863216
domComplete: 1450349864919
domContentLoadedEventEnd: 1450349864091
domContentLoadedEventStart: 1450349864066
domInteractive: 1450349864066
domLoading: 1450349863308
domainLookupEnd: 1450349863216
domainLookupStart: 1450349863169
fetchStart: 1450349863167
loadEventEnd: 1450349864920
loadEventStart: 1450349864919
navigationStart: 1450349863167
redirectEnd: 0
redirectStart: 0
requestStart: 1450349863256
responseEnd: 1450349863299
responseStart: 1450349863298
secureConnectionStart: 0
unloadEventEnd: 0
unloadEventStart: 0
```

其中有各种浏览器事件的开始时间和结束时间，具体各个事件的涵义，可以参考W3C标准，这张图描述的非常清晰。

 <figure>
         <img src="/images/navigation-timing.jpg"/>
</figure>

比如我们要获取dns的时间：

```
domainLookupEnd-domainLookupStart
```
连接建立耗时：

```
connectEnd-connectStart
```
目前这些数据都可以在浏览器中读到，将他们发布到服务端机器上，就可以统计分析用户真实网络环境先的各项指标了。
发布到服务端机器的一种最简单的方法就是：将这些数据作为参数请求一个后端文件，然后我们就可以在access日志中得到这些数据。一段伪代码如下：

``` javascript
if(!window.performace){
	var timing = window.performance.timing.toJSON();
	var params = [];
	for(var ntItem in timing){
		params.push(ntItem+'='+timing[ntItem]);
	}
	var targetUrl = 'www.baidu.com?'+params.join('&');
	ajaxGet(targetUrl);
}
```

在服务端我们就可以从access日志中得到Navigation Timing API提供的时间信息了。

本文参考W3C文档：[http://w3c.github.io/navigation-timing/](http://w3c.github.io/navigation-timing/)



