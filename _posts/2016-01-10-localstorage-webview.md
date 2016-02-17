---
layout: post
title: 使用localstorage和预加载做到webview秒开
description: "使用localstorage和预加载做到webview秒开"
modified: 2016-01-10
tags: [javascript]
image:
  background: triangular.png
comments: true
---


提到网页加载速度优化，大家都会想到静态资源上CDN，CSS和JS文件合并，图片合并成雪碧图等常用手段；但是在某些特殊情况下这些常用方法也无法达到理想的效果。比如，在国际化场景下，很多国家还停留在2G网络阶段，无论如何优化，都无法避免过慢的网络请求。最近一直在做国际化(主要是印尼和泰国)背景下的webview性能优化，也算有一些经验。由于我们的产品是面向android用户的，而android手机对H5支持很好，因此我们主要是应用H5的新特性。

###1.H5的menifest缓存的局限

H5的menifest缓存机制是大家经常提起的方法，这是一种使用起来很简单的机制，其基本原理就是：当menifest文件有更新时，就会更新整个应用，否则就不会请求网络而是使用本地缓存的资源。但是我们使用起来却有不少问题：比如即使menifest文件发生变化，应用也不能及时的更新。对于menifest机制我们的结论是：menifest可以用于缓存基本不发生变化的文件(比如基本样式表base.css,javascript类库jquery.js等)，`对于业务代码一定不要使用menifest缓存，否则可能导致业务代码无法更新`。

新浪微博中大量的使用了menifest缓存机制，但是他们也只缓存了类库和基本样式表，而没有缓存业务代码。这是一个例子 [http://card.weibo.com/article/h5/s#cid=1001603928694585472076](http://card.weibo.com/article/h5/s#cid=1001603928694585472076)，其menifest文件如下：

```
CACHE MANIFEST
# v = 8f708b371de7949de35b8cec4dac678b
CACHE:
# CSS and CSS resource image
http://u1.sinaimg.cn/apps/media/css/article_201511121505.css
http://u1.sinaimg.cn/apps/media/img/bg_cricle.png
http://u1.sinaimg.cn/apps/media/img/bg_criclein.png
http://u1.sinaimg.cn/apps/media/img/bg_imgslayer.png
http://u1.sinaimg.cn/apps/media/img/font/myfont_v3.eot
http://u1.sinaimg.cn/apps/media/img/font/myfont_v3.woff
http://u1.sinaimg.cn/apps/media/img/font/myfont_v3.ttf
http://u1.sinaimg.cn/apps/media/img/font/myfont_v3.svg
http://u1.sinaimg.cn/apps/media/img/lib/icons.svg
http://u1.sinaimg.cn/apps/media/img/lib/icon_SmartisanNote.svg
http://u1.sinaimg.cn/apps/media/img/lib/icon_Smart.svg

# Placeholder image
#http://ww3.sinaimg.cn/small/6f21f059jw1e4vxadm276j20f008ca9w.jpg

# js file
http://u1.sinaimg.cn/apps/media/js/jquery.min.js
http://u1.sinaimg.cn/apps/media/js/page/article/show/offline_201511121505.js

# Emoticons top 10
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6a/laugh.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/0b/tootha_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6d/lovea_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/c4/liwu_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/9d/sada_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/40/hearta_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/d9/ye_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/19/heia_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/8c/hsa_org.gif
http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/d0/z2_org.gif

NETWORK:
*

```

###2.使用localstorage缓存js和css文件

localstorage设计之初是为了缓存应用数据的，关于是否应该使用它缓存js和css文件，知乎上有一篇讨论：
[静态资源（JS/CSS）存储在localStorage有什么缺点？为什么没有被广泛应用？](https://www.zhihu.com/question/28467444)。其中得票最多的基本上概况了localstorage的适用场景以及优缺点：

```
PC上因为localstorage兼容性不好，而且网速较快，因此实用价值不大
移动端单页面应用(webapp)，因为localstorage兼容性好，网速慢，所以值得尝试
localstorage的兼容性问题

```

UC的前端团队推出了基于localstorage存储前端静态资源的[Scrat-webapp模块化开发体系](http://scrat.io/#!/index)，而且基于这个模块化体系开发了很多项目：[神马视频](http://tv.uc.cn/mobile/4.8.3/index.html?uc_param_str=frdnsnpfvecplabtbmntnwpvsslibieisinipr#!/index)、[NBA直播](http://nba.uc.cn/?uc_param_str=dnfrvecpntsspimiei#!/index)。大家可以通过chrome的开发者工具查看localstorage的使用情况。

基于以上的行业经验，我们也选用了localstorage作为自己的前端资源缓存，并实现了自己的基于localstorage的静态资源加载器，其实现原理可以参考我的博客:[在网页中异步加载javascript](http://foio.github.io/javascript-async/)
的XHR Eval方法。具体代码如下：


``` javascript
;(function (global) {
    'use strict';

    //检查文件类型
    var TYPE_RE = /\.(js|css)(?=[?&,]|$)/i;
    function fileType(str) {
        var ext = 'js';
        str.replace(TYPE_RE, function (m, $1) {
            ext = $1;
        });
        if (ext !== 'js' && ext !== 'css') ext = 'unknown';
        return ext;
    }

    //将js片段插入dom结构
    function evalGlobal(strScript){
        var scriptEl = document.createElement ("script");
        scriptEl.type= "text/javascript";
        scriptEl.text= strScript;
        document.getElementsByTagName("head")[0].appendChild(scriptEl) ;
    }

    //将css片段插入dom结构
    function createCss(strCss) {
        var styleEl = document.createElement('style');
        document.head.appendChild(styleEl);
        styleEl.appendChild(document.createTextNode(strCss));
    }

    // 在全局作用域执行js或插入style node
    function defineCode(url, str) {
        var type = fileType(url);
        if (type === "js"){
            //with(window)eval(str);
            evalGlobal(str);
        }else if(type === "css"){
            createCss(str);
        }
    }

    // 将数据写入localstorage
    var setLocalStorage = function(key, item){
        window.localStorage && window.localStorage.setItem(key, item);
    }

    // 从localstorage中读取数据
    var getLocalStorage = function(key){
        return window.localStorage && window.localStorage.getItem(key);
    }

    // 通过AJAX请求读取js和css文件内容，使用队列控制js的执行顺序
    var rawQ = [];
    var monkeyLoader = {
        loadInjection: function(url,onload,bOrder){
            var iQ = rawQ.length;
            if(bOrder){
                var qScript = {key: null, response: null, onload: onload, done: false};
                rawQ[iQ] = qScript;
            }
            //有localstorage 缓存
            var ls = getLocalStorage(url);
            if(ls !== null){
                if(bOrder){
                    rawQ[iQ].response = ls;
                    rawQ[iQ].key = url;
                    monkeyLoader.injectScripts();
                }else{
                    defineCode(url, ls)
                    if(onload){
                        onload();
                    }
                }
            } else {
                var xhrObj = monkeyLoader.getXHROject();
                xhrObj.open('GET', url, true);
                xhrObj.send(null);
                xhrObj.onreadystatechange = function(){
                    if(xhrObj.readyState == 4){
                        if(xhrObj.status == 200){
                            setLocalStorage(url, xhrObj.responseText);
                            if(bOrder){
                                rawQ[iQ].response = xhrObj.responseText;
                                rawQ[iQ].key = url;
                                monkeyLoader.injectScripts();
                            }else{
                                defineCode(url, xhrObj.responseText)
                                if(onload){
                                    onload();
                                }
                            }
                        }
                    }
                }
            }
        },

        injectScripts: function(){
            var len = rawQ.length;
            //按顺序执行队列中的脚本
            for (var i = 0; i < len; i++) {
                var qScript = rawQ[i];
                //没有执行
                if(!qScript.done){
                    //没有加载完成
                    if(!qScript.response){
                        console.error("raw code lost or not load!");
                        //停止，等待加载完成, 由于脚本是按顺序添加到队列的，因此这里保证了脚本的执行顺序
                        break;
                    }else{//已经加载完成了
                        defineCode(qScript.key, qScript.response)
                        if(qScript.onload){
                            qScript.onload();
                        }
                        delete qScript.response;
                        qScript.done = true;
                    }
                }
            }
        },

        getXHROject: function(){
            //创建XMLHttpRequest对象
            var xmlhttp;
            if (window.XMLHttpRequest)
                {
                    // code for IE7+, Firefox, Chrome, Opera, Safari
                    xmlhttp=new XMLHttpRequest();
                } else {
                    // code for IE6, IE5
                    xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
                }
                return xmlhttp;
        }
    };


    global.monkeyLoader = monkeyLoader.loadInjection;

})(this);
```

在页面中我们可以使用monkeyLoader加载自己的静态资源

``` javascript
monkeyLoader('a.js',callbackA,true);
monkeyLoader('b.js',callbackB,true);
```

其中monkeyLoader的三个参数分别是：资源文件路径、资源加载完成后的回调函数、以及资源是否需要有序。

###3.基于fis3的localstorage集成解决方案

[FIS3](http://fis.baidu.com/)作为一个优秀的前端工程化解决方案，其在业界的影响无处不在。作为狼厂的一员，我们在项目中也大量使用FIS3，下面我们讨论一下如何将localstorage集成到FIS3的编译流程中。

我们的目的是将通过FIS3工程化处理后的文件列表传递到monkeyLoader，以实现基于localstorage的加载。FIS3提供了静态资源映射表，当某个文件包含字符 __RESOURCE_MAP__，就会用表结构数据替换此字符，而该表记录了各个原始静态资源经过工程化后的目标文件。基于此，我们只需要对monkeyLoader进行简单的封装即可。下面是具体的封装代码：

``` javascript
fisMonkeyLoader = function(resourceMap,resourceList){
        var pkgList = [];
        for(var idx in resourceList){
            var resourceItem = resourceList[idx];
            var pakItem = resourceMap['pkg'][resourceMap['res'][resourceItem]['pkg']]['uri'];
            pkgList[pakItem] =  1;
        }
        for(var pkgIdx in pkgList){
            monkeyLoader(pkgIdx,null,true);
        }

    }

```

而使用方法也很简单：

``` javascript
__inline('../js/lib/monkeyLoader.js');
fisMonkeyLoader(__RESOURCE_MAP__,[
	'css/animate.css',
	'css/app.css',
	'js/lib/zepto-1.1.3.min.js',
	'js/app/app.js',
]);

```

fisMonkeyLoader根据__RESOURCE_MAP__找到原始资源对应的经过工程化处理以后的资源，并通过monkeyLoader进行加载。

也许你已经注意到了，使用localstorage只能加载同域的js和css文件，因为我们没办法通过js代码读取其他域下的静态资源的内容。而且localstorage有大小限制，一般情况下每个域名下最多使用5M的本地存储空间，因此我们务必要处理超出大小限制是的异常情况，各个浏览器抛出的异常不太一致，[这里给出了一个跨浏览器的异常检测方案](http://crocodillon.com/blog/always-catch-localstorage-security-and-quota-exceeded-errors)

###4.通过预加载解决首次加载过慢问题

针对目前常见的hybrid架构，我们可以通过让native客户端预加载webview从而提升首次速度。我们的实现是：客户端在启动时打开一个1px大小(用户不可见)的webview用于提前加载某些重要的页面，而这些页面通过上文中的localstorage机制存储了几乎全部js和css静态资源，从而得到webview秒开的体验。
