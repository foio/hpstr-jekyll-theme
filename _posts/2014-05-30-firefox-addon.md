---
layout: post
title: firefox扩展开发之Add-ons SDK
description: "firefox扩展开发"
modified: 2014-05-20
tags: [firefox]
image:
  background: triangular.png
---

firefox扩展开发主要有两种模式：

>1.Add-ons SDK扩展，sdk的特点是简单，适合初级玩家。这种开发方式最快的入门方法就是阅读[mozilla的官方文档](https://developer.mozilla.org/en-US/Add-ons/SDK)。

>2.bootstrap扩展，这种扩展开发方式从无到有完全手动，需要学习firefox的界面描述语言，适合高级玩家。 可以参考[IMB developwoker上的一篇入门文章](http://www.ibm.com/developerworks/cn/web/wa-lo-firefox-ext/)

本人属于初级玩家，所以本文主要描述使用Add-ons SDK建立一个具有基本功能的扩展。

------

1.环境篇
====

下载python2.5、2.6或者2.7，firefox，[Add-ons SDK](https://ftp.mozilla.org/pub/mozilla.org/labs/jetpack/addon-sdk-1.16.zip) 。确保python在系统的path中。解压下载的sdk的zip文件，在命令行模式下运行：


```
cd add-ons 
bin\active
#如果active(主要是初始化一些变量)运行成功，会改变当前的命令行前缀，类似如下:
(C:\Users\mozilla\sdk\addon-sdk) C:\Users\Work\sdk\addon-sdk>
#接下来继续在当前命令行测试环境是否ok，运行如下命令
cfg
#第一行的输出应该是：Usage: cfx [options] [command]
```
到此为止，扩展开发环境就算ok了。如果遇到问题，请浏览[官方文档](https://developer.mozilla.org/en-US/Add-ons/SDK/Tutorials/Installation)


-------

2.工具篇
====

#(1) 用cfx init产生扩展的基本框架

工欲善其事，必先利其器，好的工具能够大幅度提高开发效率。我要介绍的工具是cfx。

```
mkdir my-addon
cd my-addon 
cfx init
* lib directory created
* data directory created
* test directory created
* doc directory created
* README.md written
* package.json written
* test/test-main.js written
* lib/main.js written
* doc/main.md written
Your sample add-on is now ready for testing:
try "cfx test" and then "cfx run". Have fun!"
```

-----


#(2)实现扩展的具体逻辑

扩展的具体逻辑在lib/main.js中。假如我们的想在浏览器的工具栏上实现一个button，点击后在新的tab页面中打开www.baidu.com。具体的逻辑代码如下：

{% highlight javascript %}
var widgets = require("sdk/widget");
var tabs = require("sdk/tabs");
var widget = widgets.Widget({
  id: "mozilla-link",
  label: "Mozilla website",
  contentURL: require("sdk/self").data.url("icon-16.png"),
  onClick: function() {
    tabs.open("http://developer.mozilla.org/");
  }
});
{% endhighlight %}

当然你也可以自定义button的icon，只需要替换data目录下的icon-16.png，icon-32.png，icon-64.png

-----

#(3)浏览器中测试
使用cfx run在firefox中测试开发中扩展，cfx run命令会启动firefox并安装该插件。

-----

#(4)打包

使用cfx xpi打包插件为xpi包。打包后的xpi文件就可以直接使用浏览器安装了。

-----
