---
layout: post
title: 在网页中异步加载javascript
description: "jQuery事件系统研究"
modified: 2015-11-02
tags: [javascript]
image:
  background: triangular.png
comments: true
---


<style type="text/css">
td, table {
   border: solid gray 1px;
   padding: 10px;
}
</style>


前端工程师都知道script标签会阻塞网页上其他资源的加载。有时候这种阻塞是必要的，因为javascript可能会改变页面结构，进而对后续的资源(css,js)的作用产生影响。但是，当我们能够识别那些对页面结构不产生影响的javascript并且不希望阻塞其他资源时，我们就需要认真的研究一下，javascript异步加载的方式了。

---

###1. 并行的下载脚本

####(1) XHR eval

通过XHR技术我们也可以异步地获取js脚本，并通过eval()执行。

``` javascript
var xhrobj = getXHROject();
getXHROject.onreadystatechange = function(){
    if(xhrObj.readyState ==4 && 200 == xhrObj.status){
      eval(xhrObj.responseText);
  }
};
xhrObj.open(‘GET’,’A.js’,true)
xhrOjb.send(‘’);
```

由于XHR请求不能跨域，所以脚本必须和主页部署在相同的域中，脚本可并行下载，而且不阻塞其他资源，但是无法保证多个脚本的执行顺序。


####(2) script dom element

我们也可以直接在浏览器中插入script dom节点。

``` javascript
var scriptElem = document.createElement(‘script’);
document.getElementsByTagName(‘head’)[0].appendChild(scriptElem);
```

这种方式允许跨域加载js脚本，不阻塞其他资源下载。只有 Firefox 和 Opera 保证脚本按文档中出现的顺序执行，其他浏览器需要工程师自己在代码层面实现执行顺序的控制。Requirjs就是这样实现的。

####(3) document write script tag

``` javascript
document.write("<script type="text/javascript" src='A.js'></script>");
```

注意script Tag和script dom的区别，scritp Tag可以保证多个脚本并行加载，但是会阻塞其他资源并行下载。这种方式可以保证脚本按文档中出现的顺序执行

####(4) defer和async属性

目前大多数浏览器已经defer和async属性。

```
1. 如果 async="async"：脚本相对于页面的其余部分异步地执行（当页面继续进行解析时，脚本将被执行）
2. 如果不使用 async 且 defer="defer"：脚本将在页面完成解析时执行
3. 如果既不使用 async 也不使用 defer：在浏览器继续解析页面之前，立即读取并执行脚本
```
 async="async"不会阻塞其他资源，但是无法保证脚本的执行顺序。defer="defer"阻塞其他资源的加载，并且可以保证脚本的执行顺序，但是要到页面解析完成后才开始执行脚本。



总结如下:

|并行性	    |        异步性	     |       顺序性            |           其他限制|
| ------------- |:-------------|:-------------|:--------------------|
|script src	   |     脚本并行下载，阻塞其他资源|浏览器保证执行顺序	|无|
|XHR Eval | 脚本并行下载，不阻塞其他资源|浏览器不保证执行顺序|	要求同域|
|script dom element|	脚本并行下载，阻塞其他资源|浏览器不保证执行顺序|无|
|document write script tag|脚本并行下载，阻塞其他资源|浏览器保证执行顺序|无|
|defer|  脚本并行下载，不阻塞其他资源，脚本推迟到页面解析完成后执行|浏览器保证执行顺序|要求脚本不改变dom结构|
|async|脚本并行下载，不阻塞其他资源|需要业务逻辑自己保证顺序|无|

这么多种异步加载javascript脚本的方式，各有利弊，接下来研究一下如何控制脚本之间执行的顺序。

###2.脚本的执行顺序

####(1)保证行内脚本和外部脚本的执行顺序

当外部脚本按常规方式加载时，它会阻塞行内脚本的执行，可以保证顺序。但是脚本通过上述的几种方式异步加载时，就无法保证行内脚本和异步脚本之间的顺序。下面就讲解一下保证行内脚本和外部脚本保证执行顺序的技术。

#####[1].硬编码回调
如果web开发者能够控制外部脚本，可以在外部脚本回调行内脚本。

#####[2]onlode事件
添加script dom节点时，监听加载事件，当脚本成功加载时调用callback(外部脚本)函数。

``` javascript
//行内函数
function callback(){
	Console.log(‘calllback’);
}

//异步加载函数
function loadScript(url, callback){
    var script = document.createElement ("script")
    script.type = "text/javascript";
    if (script.readyState){ //IE
        script.onreadystatechange = function(){
            if (script.readyState == "loaded" || script.readyState == "complete"){
                script.onreadystatechange = null;
                callback();
            }
        };
    } else { //Others
        script.onload = function(){
            callback();
        };
    }
    script.src = url;
    document.getElementsByTagName("head")[0].appendChild(script);
}

//控制行内脚本和外部脚本的执行顺序
loadScript('a.js',callback);
</script>
```

#####[3]定时器

通过定时检查外部脚本的相应变量是否定义，可以判断外部脚本是否加载并执行成功。

``` javascript
<script src="MyJs.js"></script>
<script>
function callback(){
}

function checkMyJs(){
	if(undefined===typeof(MyJs)){
        setTimeout(checkMyJs, 300)
    }else{
        callback();
    }
}
</script>
```

这三种方法都可以保证行内脚本和外部脚本之间的执行顺序。其实最难的是保证多个外部脚本之间的执行顺序，这也是我们接下来要看的内容。

####(2)保证多个外部脚本之间的执行顺序

#####[1]. 同域中的脚本
对于同域中的多个外部脚本，可以使用XHR的方式加载脚本，并通过一个队列来控制脚本的执行顺序。

``` javascript
<script>
    ScriptLoader.Script = {
        //脚本队列
        queueScripts = [];
        loadScriptXhrInjection: function(url,onload,bOrder){
            var iQ = ScriptLoader.Script.queueScripts.length;
            if(bOrder){
                var qScript = {response: null, onload: onload, done: false}; 
                ScriptLoader.Script.queueScripts[iQ] = qScript;
            }
            var xhrObj = ScriptLoader.Script.getXHROject();
            xhrObj.onreadystatechange = function(){
                if(xhrObj.readyState == 4){
                    //有顺序要求的脚本需要添加的队列，按添加顺序执行
                    if(bOrder){
                        //有顺序要求的脚本需要设置加载和执行状态
                        ScriptLoader.Script.queueScripts[iQ].response = xhrObj.responseText;
                        //执行脚本队列
                        ScriptLoader.Script.injectScripts();
                    }else{//没有顺序要求的脚本可直接执行
                        eval(xhrObj.responseText);
                        if(onload){
                            onload();
                        }
                    }
                }
            }
        }


        injectScripts: function(){
            var len = ScriptLoader.Script.queueScripts.length;
            //按顺序执行队列中的脚本
            for (var i = 0; i < len; i++) {
                var qScript = ScriptLoader.Script.queueScripts[i];
                //没有执行
                if(!qScript.done){
                    //没有加载完成
                    if(!qScript.response){
                        //停止，等待加载完成, 由于脚本是按顺序添加到队列的，因此这里保证了脚本的执行顺序
                        break;
                    }else{//已经加载完成了
                        eval(qScript.response);
                        if(qScript.onload){
                            qScript.onload(); 
                        }
                        qScript.done = true;
                    }
                }
            };
        },

        getXHROject: function(){
            //创建XMLHttpRequest对象
        }
    }

    ScriptLoader.Script.loadScriptXhrInjection('A.js',null,false);
    ScriptLoader.Script.loadScriptXhrInjection('B.js',InitB,true);
    ScriptLoader.Script.loadScriptXhrInjection('C.js',InitC,true);
</script>
```

#####[2]对于不同域的脚本

script dom element 可以异步脚本脚本，不阻塞其他资源，并且在firefox和opera可以保证执行顺序；而document write script 可以异步加载脚本，会阻塞其他资源，在所有浏览器都可以保证执行顺序。因此我们可以根据浏览器选择以上两种方案来控制
不同域的脚本的执行顺序。

``` javascript
<script>
    ScriptLoader.script{
       loadScriptDomElement:function(url, onload){
            var script = document.createElement ("script")
            script.type = "text/javascript";
            if (script.readyState){ //IE
                script.onreadystatechange = function(){
                    if (script.readyState == "loaded" || script.readyState == "complete"){
                        script.onreadystatechange = null;
                        onload();
                    }
                };
            } else { //Others
                script.onload = function(){
                    onload();
                };
            }
            script.src = url;
            document.getElementsByTagName("head")[0].appendChild(script);
        }    

        loadScriptDomWrite: function(url,onload){
            document.write('<script  src="'+url+'" type="text/javascript"></script>');  
            if(onload){
                if(elem.addEventListener){//others
                    elem.addEventListener(window,'load',onload);
                }else if(elem.attachEvent){ //IE
                    elem.addEventListener(window,'onload',onload);
                }
            }
        } 
		
		//根据浏览器选择浏览器加载js的方式
        loadScript: function(url,onload){
                if(-1 != navigator.userAgent.idexOf('Firefox') ||
                   -1 != navigator.userAgent.indexOf('Opera')){
                    //当浏览器为firefox和opera时通过Script Dom Element 保证脚本执行顺序
                        DomTag.script.loadScriptDomElement(url,onload); 
                }else{
                    //当为其他浏览器时，通过document write Script保证脚本执行顺序。此时脚本的加载会阻塞其他资源，这是一种折衷
                        DomTag.script.loadScriptDomWrite(url,onload);
                }
        }  
}
ScriptLoader.script.loadScript('A.js',initA);
ScriptLoader.script.loadScript('B.js',initB);
</script>
```

夜深了，城市的灯光污染，我无法看到今夜的月。
