---
layout: post
title: webpack代码分割技巧
description: "webpack代码分割，require.ensure，CommonChunkPlugin，DllPlugin和DllReferencePlugin"
modified: 2016-11-18
tags: [javascript,require.ensure,CommonChunkPlugin,DllPlugin,DllReferencePlugin]
image:
  background: triangular.png
comments: true
---

##1. 代码中定义分割点

webpack支持在代码中定义分割点。分割点指定的模块只有在真正使用时才加载，可以使用webpack提供的require.ensure语法:

``` javascript
$('#okButton').click(function(){
  require.ensure(['./foo'], function(require) {
    var foo = require('./foo');
    //your code here
  });
});
```

也可以像RequireJS一样使用AMD语法：

``` javascript
$('#okButton').click(function(){
  require(['foo'],function(foo){
    // your code here
  }]);
});
```

上面两种方式都会`以foo模块为入口将其依赖模块递归地打包到一个新的Chunk`，并在`#okButton`按钮点击时才异步地加载这个以foo模块为入口的新的chunk。


##2. 使用CommonsChunkPlugin分割代码

在理解CommonsChunkPlugin代码分割之前，我们需要熟悉webpack中chunk的概念，webpack将多个模块打包之后的代码集合称为chunk。根据不同webpack配置，chunk又有如下几种类型：

>
Entry Chunk: 包含一系列模块代码，以及webpack的运行时(Runtime)代码，一个页面只能有一个Entry Chunk，并且需要先于Normal Chunk载入

>
Normal Chunk: 只包含一系列模块代码，不包含运行时(Runtime)代码。

作为webpack代码分割的利器，网络上有太多CommonsChunkPlugin的文章，但以某一使用场景的入门案例为主。本文我们根据不同场景下的使用方法，分别介绍。

###2.1 提取库代码

假设我们需要将很少变化的常用库（react、lodash、redux）等与业务代码分割，可以在webpack.config.js采用如下配置：

``` javascript
var webpack = require("webpack");
module.exports = {
  entry: {
    app: "./app.js",
    vendor: ["lodash","jquery"],
  },
  output: {
    path: "release",
    filename: "[name].[chunkhash].js"
  },
  plugins: [
    new webpack.optimize.CommonsChunkPlugin({names: ["vendor"]})
  ]
};

```

上述配置将常用库打包到一个vender命名的`Entry Chunk`，并将以app.js为入口的业务代码打包到一个以business命名的`Normal Chunk`。其中`Entry Chunk`包含了webpack的运行时(Runtime)代码，所以在页面中必须先于业务代码加载。


###2.2 提取公有代码


假设我们有多个页面，为了优化网络加载性能，我们需要将多个页面共用的代码提取出来单独打包。可以在webpack.config.js进行如下配置：

``` javascript
var webpack = require("webpack");
module.exports = {
    entry: { 
          page1: "./page1.js", 
          page2: "./page2.js" 
        },
    output: { 
          filename: "[name].[chunkhash].js" 
        },
    plugins: [ new webpack.optimize.CommonsChunkPlugin("common.[chunkhash].js") ]
}
```

上述配置将两个页面中通用的代码抽取出来并打包到以common命名的`Entry Chunk`，并将以page1.js和page2.js为入口代码分别打包到以page1和page2命名的`Normal Chunk`。
其中`Entry Chunk`包含了webpack的运行时(Runtime)代码，所以`common.[chunkhash].js`在两个页面中都必须在page1.[chunkhash].js和page2.[chunkhash].js前加载。

在这种配置下，CommonsChunkPlugin的作用可以抽象：

>
将多个入口中的公有代码和Runtime(运行时)抽取到父节点

理解了CommonsChunkPlugin的本质后，我们看一个更复杂的例子：


``` javascript
var webpack = require("webpack");
module.exports = {
    entry: {
        p1: "./page1",
        p2: "./page2",
        p3: "./page3",
        ap1: "./admin/page1",
        ap2: "./admin/page2"
    },
    output: {
        filename: "[name].js"
    },
    plugins: [
        new webpack.optimize.CommonsChunkPlugin("admin-commons.js", ["ap1", "ap2"]),
        new webpack.optimize.CommonsChunkPlugin("commons.js", ["p1", "p2", "admin-commons.js"])
    ]
};
// page1.html: commons.js, p1.js
// page2.html: commons.js, p2.js
// page3.html: p3.js
// admin-page1.html: commons.js, admin-commons.js, ap1.js
// admin-page2.html: commons.js, admin-commons.js, ap2.js
```

我们可以用树结构描述上述配置的作用：

![commonchunk](/images/common-chunk-tree.png)

每一次使用CommonsChunkPlugin都会将共有代码和runtime提取到父节点。上述例子中，通过两次CommonChunkPlugin的作用，runtime被提取到common.js中。通过这种树型结构，我们可以清晰的看出每个页面对各个chunk的依赖顺序。

###2.3 提取Runtime(运行时)代码

使用CommonsChunkPlugins时，一个常见的问题就是：

>
没有被修改过的公有代码或库代码打包出的Entry Chunk，会随着其他业务代码的变化而变化，导致页面上的长缓存机制失效。

[github上有一个与此相关的问题](https://github.com/webpack/webpack/issues/1315#issuecomment-155100976)。本意就是在只修改业务代码时，而不改动库代码时，打包出的库代码的chunkhash也发生变化，导致浏览器端的长缓存机制失效。如图所示，app和vender的chunkhash都发生了变化。

![commonchunk](/images/common-chunk-vender.png)

![commonchunk](/images/common-chunk-vender2.png)

这主要是因为使用CommonsChunkPlugin提取代码到新的chunk时，会将webpack运行时(Runtime)也提取到打包后的新的chunk。通过如下配置就可以将webpack的runtime单独提取出来：

``` javascript
var webpack = require("webpack");
module.exports = {
  entry: {
    app: "./app.js",
    vendor: ["lodash","jquery"],
  },
  output: {
    path: 'release',
    filename: "[name].[chunkhash].js"
  },
  plugins: [
    new webpack.optimize.CommonsChunkPlugin({names: ['vendor','runtime']}),
  ]
};
```

这种情况下，当业务代码发送变化，而库代码没有改动时，vender的chunkhash不会变，这样才能最大化的利用浏览器的缓存机制。如下图所示：


![commonchunk](/images/common-chunk-runtime.png)

修改业务代码后，vender的chunkhash不会变化，方便使用浏览器的缓存：

![commonchunk](/images/common-chunk-runtime2.png)

由于webpack的runtime比较小，我们可以直接将该文件的内容inline到html中。


##3. 使用DllPlugin和DllReferencePlugin分割代码

通过DllPlugin和DllReferencePlugin，webpack引入了另外一种代码分割的方案。我们可以将常用的库文件打包到dll包中，然后在webpack配置中引用。业务代码的可以像往常一样使用require引入依赖模块，比如require('react'), webpack打包业务代码时会首先查找该模块是否已经包含在dll中了，只有dll中没有该模块时，webpack才将其打包到业务chunk中。

首先我们使用DllPlugin将常用的库打包在一起：

``` javascript
var webpack = require('webpack');
module.exports = {
  entry: {
    vendor: ['lodash','react'],
  },
  output: {
    filename: '[name].[chunkhash].js',
    path: 'build/',
  },
  plugins: [new webpack.DllPlugin({
    name: '[name]_lib',
    path: './[name]-manifest.json',
  })]
};
```

该配置会产生两个文件，模块库文件：vender.[chunkhash].js和模块映射文件：vender-menifest.json。其中vender-menifest.json标明了模块路径和模块ID（由webpack产生）的映射关系，其文件内容如下：

```
{
  "name": "vendor_lib",
  "content": {
    "./node_modules/.npminstall/lodash/4.17.2/lodash/lodash.js": 1,
    "./node_modules/.npminstall/webpack/1.13.3/webpack/buildin/module.js": 2,
    "./node_modules/.npminstall/react/15.3.2/react/react.js": 3,
    ...
    }
}    
```

![commonchunk](/images/webpack-dll-vender.png)

在业务代码的webpack配置文件中使用DllReferencePlugin插件引用模块映射文件：vender-menifest.json后，我们可以正常的通过require引入依赖的模块，如果在vender-menifest.json中找到依赖模块的路径映射信息，webpack会直接使用dll包中的该依赖模块，否则将该依赖模块打包到业务chunk中。

``` javascript
var webpack = require('webpack');
module.exports = {
  entry: {
    app: ['./app'],
  },
  output: {
    filename: '[name].[chunkhash].js',
    path: 'build/',
  },
  plugins: [new webpack.DllReferencePlugin({
    context: '.',
    manifest: require('./vendor-manifest.json'),
  })]
};

```

由于依赖的模块都在dll包中，所以例子中app打包后的chunk很小。

![commonchunk](/images/webpack-dll-app.png)

需要注意的是：dll包的代码是不会执行的，需要在业务代码中通过require显示引入。相比于CommonChunkPlugin，使用DllReferencePlugin分割代码有两个明显的好处：

>
（1）由于dll包和业务chunk包是分开进行打包的，每一次修改代码时只需要对业务chunk重新打包，webpack的编译速度得到极大的提升，因此相比于CommonChunkPlugin，DllPlugin进行代码分割可以显著的提升开发效率。

>
（2）使用DllPlugin进行代码分割，dll包和业务chunk相互独立，其chunkhash互不影响，dll包很少变动，因此可以更充分的利用浏览器的缓存系统。而使用CommonChunk打包出的代码，由于公有chunk中包含了webpack的runtime(运行时)，公有chunk和业务chunk的chunkhash会互相影响，必须将runtime单独提取出来，才能对公有chunk充分地使用浏览器的缓存。



本文所有demo代码均在github上：[https://github.com/foio/webpack-code-splitting-demos](https://github.com/foio/webpack-code-splitting-demos)

---


参考文献


https://robertknight.github.io/posts/webpack-dll-plugins/

http://engineering.invisionapp.com/post/optimizing-webpack/

https://github.com/webpack/docs/wiki/optimization

https://medium.com/@soederpop/webpack-plugins-been-we-been-keepin-on-the-dll-cdfdd6cb8cd7#.g79bu37wr

http://engineering.invisionapp.com/post/optimizing-webpack/

https://github.com/webpack/docs/wiki/optimization

https://github.com/webpack/webpack/issues/1315#issuecomment-155100976

https://github.com/zhengweikeng/blog/issues/10
