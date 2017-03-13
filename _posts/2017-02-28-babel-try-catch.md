---
layout: post
title: 一种生产环境中高效定位JS异常的方案
description: "babel try catch，AST，webpack loader，global try catch"
modified: 2017-02-28
tags: [babel,AST,try catch]
image:
  background: triangular.png
comments: true
---



在我的一篇文章中：[捕获页面中全局Javascript异常](http://foio.github.io/javascript-global-exceptions/)，介绍了通过AST(抽象语法树)技术，借助于UglifyJs提供的AST的API，对源文件进行预处理，对每个函数自动地添加try catch包裹代码，从而捕获生产环境中的JS异常。通过使用[try-catch-global.js](https://github.com/foio/try-catch-global.js)，可以简单的如下源代码：

``` javascript

var test = function(){
    console.log('test');
}

```

转换成

``` javascript
var test = function() {             
    try {                           
        console.log("test");        
    } catch (error) {               
        (function(error) {          
            //your logic to handle error     
        })(error);                  
    }                               
};                 
```

但是由于我们发布到生产环境中的代码，往往都经过压缩和混淆，文件名、函数名、变量名已经不具有可读性了，捕获的异常堆栈信息的价值有限，简单的通过这些异常信息，我们依然很难定位到源文件中错误出处。本文我们就试图解决这个问题：依然基于AST(抽象语法树)对源代码进行try catch包裹，但会在catch语句中收集更多源文件的信息(包括文件名、函数名、函数起始行号等)，最后借助于babel和webpack的插件体系，提供一个工程化的解决方案。

###1. 自定义babel-plugin实现try catch包裹

在文章[捕获页面中全局Javascript异常](http://foio.github.io/javascript-global-exceptions/)中，我们使用的是UglifyJS提供的操作Javascript语法树的API，这套API比较底层，需要发力气才能啃透，不适合初学者使用。babel提供了抽象层次更高的操作语法树的API:[babylon](https://github.com/babel/babylon)，并且提供了一系列工具[babel-template](https://www.npmjs.com/package/babel-template)、[babel-helper-function-name](https://www.npmjs.com/package/babel-helper-function-name)等。



在使用Babel对Javascript源文件进行处理时，有三个主要步骤，分别是： 解析（parse），转换（transform），生成（generate）。Babel首先会将源文件转换为抽象语法树(AST)，然后对抽象语法树进行转换，最后由抽象语法树生成新的源代码，如下图所示。在转换（transform）阶段，Babel提供了非常便利的插件机制，开发者可以在插件中实现自己的AST转换。关于如何开发Babel插件，最好的教程就是[官方文档](https://github.com/thejameskyle/babel-handbook/blob/master/translations/en/plugin-handbook.md)。

![babel插件转换AST](/images//babel-plugin-ast.png)

在对AST的转换阶段，Babel使用babel-traverse对AST进行深度优先遍历，它的插件机制使得我们可以针对某个特定类型的语法树节点（比如，函数、条件语句等）注册钩子函数，从而完成我们对语法树的转换工作。

``` javascript
  visitor: {
    Function: {
      //遍历到函数时
    },
    ClassMethod: {
      //遍历catch语句块时
    }
    ......
  }
```

通过插件机制，我们可以对所有的函数和类方法节点进行转换，插入try catch包裹代码；同时，在babel解析后的语法树中包含了详细的源文件的元信息，我们可以将这些源文件信息透传到自定义的错误处理函数中。对于函数，我们可以如下处理：

``` javascript
visitor: {
        //只处理函数和类方法节点
        "Function|ClassMethod" {
            exit: function exit(path, state) {
                //深度优先搜索会遍历两次，需要避免重复
                if (shouldSkip(path, state)) {
                    return;
                }

                //如果函数体为空则不处理
                var body = path.node.body.body;
                if (body.length === 0) {
                    return;
                }

                //收集函数名
                var functionName = 'anonymous function';
                babelHelperFunctionName2(path);
                if (path.node.id) {
                    functionName = path.node.id.name || 'anonymous function';
                }

                //收集类方法名
                if(path.node.key){
                    functionName = path.node.key.name || 'anonymous function';
                }

                //函数起始行号
                var loc = path.node.loc;

                //异常变量名
                var errorVariableName = path.scope.generateUidIdentifier('e');

                //使用函数模板进行try catch包裹，需要注意的是AST无法获取到文件名信息，需要外部传入
                path.get('body').replaceWith(wrapFunction({
                    BODY: body,
                    FILENAME: t.StringLiteral(filename),
                    FUNCTION_NAME: t.StringLiteral(functionName),
                    LINE: t.NumericLiteral(loc.start.line),
                    COLUMN: t.NumericLiteral(loc.start.column),
                    REPORT_ERROR: t.identifier(reportError),
                    ERROR_VARIABLE_NAME: errorVariableName
                }));
            }
        },
```


babel提供了非常便利的工具babel template，其中隐藏了AST转换的细节，简单的使用函数模板就可以对函数进行任意转换，如下代码我们使用template对函数进行try catch包裹：

```javascript
const wrapFunction = template(`{
  try {
    BODY
  } catch(ERROR_VARIABLE_NAME) {
    REPORT_ERROR(ERROR_VARIABLE_NAME, FILENAME, FUNCTION_NAME, LINE, COLUMN)
    throw ERROR_VARIABLE_NAME
  }
}`)

```

通过使用babel插件，在babel对源代码进行处理时注册针对特定AST节点的钩子函数（本文我们只关心函数类型节点），使用bable-template对函数进行try catch包裹，并在catch语句中预埋入从AST中收集到的源文件信息。

 处理前的代码：

``` javascript
 function testA(){
    console.log(1);
}


class A {
    testB(){
        console.log(1);
    }
}

var testD = function(){
    console.log(1)
}

```

 处理后的代码：

``` javascript
function testA() {
    try {
        console.log(1);
    } catch (_e) {
        reportError(_e, "test.js", "testA", 4, 0);
    }
}

class A {
    testB() {
        try {
            console.log(1);
        } catch (_e2) {
            reportError(_e2, "test.js", "testB", 10, 4);
        }
    }
}

var testD = function testD() {
    try {
        console.log(1);
    } catch (_e4) {
        reportError(_e4, "test.js", "testD", 19, 12);
    }
};

```

 转换后的代码通过后续的混淆、压缩后发布到生产环境，生产环境中的代码发生异常时，catch语句中的reportError会将异常上报到日志平台，上报的信息中包含了我们从AST中预埋入的变量名、函数名、函数函数起始行号、文件名等信息，通过这些信息我们就可以快速定位到源代码中的异常位置。

###2. 使用webpack loader进行工程化构建

上文讲到使用babel插件对Javascript源代码生成的AST进行转换，最终对所有的函数生成try catch包裹代码。本小节我们考虑将构建流程集成到webpack中。webpack首先使用loader对源代码进行处理，然后将入口文件以及其依赖打包到一个chunk中。在webpack的编译流程中，我们可以借助于自定义的loader来实现对源代码的AST转换。

编写一个自定义的webpack的loader非常简单，简单的教程请参考[官方文档](https://webpack.github.io/docs/how-to-write-a-loader.html)，下文是一个非常简单的loader，其接收源代码内容作为输入，并转转换后的源代码作为输出。

``` javascript
module.exports = function(source) {
  //your logic to change source
  return source;
};
```

我们也实现了一个webpack的loader：[babel_try_catch_loader](https://github.com/foio/babel_try_catch_loader)，它借助于babel插件[babel-plugin-try-catch-wrapper](https://github.com/foio/babel-plugin-try-catch-wrapper)，通过AST技术对源代码进行处理。

``` javascript
var tryCatchWrapper = require('babel-plugin-try-catch')
...
module.exports = function (source, inputMap) {
    ......
    var transOpts = {
        plugins: [
            [tryCatchWrapper, {
                filename: filename,
                reportError: userOptions.reporter,
                rethrow: userOptions.rethrow
            }]
        ],
        sourceMaps: true
    };
    var result = babel.transform(source, transOpts);
    this.callback(null, result.code, result.map);
    ......
};
```

只需要在webpack配置文件中使用[babel_try_catch_loader](https://github.com/foio/babel_try_catch_loader)，我们就可以通过一行配置文件来将项目中源代码中所有的函数进行try catch包裹了。

``` javascript
loaders: [
            ......
            {
                test: /\.jsx|\.js$/,
                loader: 'babel-loader!babel-try-catch-loader?rethrow=true&verbose&reporter=reportError&tempdir=.tryCatchResult',
                exclude: /node_modules/
            }
            ......
        ],
```

上文中的配置文件，对测试环境中使用babel-loader；而对生产环境则使用babel-try-catch-loader处理项目中所有源代码，以保证经过后续的混淆、压缩并发布到线上环境的生产代码仍然具有较强的debug能力。详细的配置文件请参考[webpack-try-catch-demo](https://github.com/foio/webpack-try-catch-demo)。
