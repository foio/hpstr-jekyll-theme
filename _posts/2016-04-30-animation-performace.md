---
layout: post
title: 前端高性能动画最佳实践
description: "高性能动画,animation,最佳实践,will-change,"
modified: 2016-04-30
tags: [javascript,高性能动画]
image:
  background: triangular.png
comments: true
---

何为高性能动画？让人感觉流程顺滑即可。24fps的电影就能让人感觉到流畅，但是游戏却要60fps以上才能让人感觉到流畅。分析原因，我们得出如下结论：

```
(1)视频的每一帧记录的是一段时间段(1/24s)的信息，而游戏的每一帧都由显卡绘制，它只能生成一个时间点的信息;
(2)视频的帧率是稳定的，而在系统负载不平稳时，显卡很难保证游戏帧率的稳定性；
```

前端动画与游戏的原理类似，我们设计高性能动画的基本思路就是`提高帧率`和`稳定帧率`。让我们首先一起了解一下浏览器渲染页面的基本过程。

##1.理解浏览器渲染流水线

渲染的基本流程是：扫描HTML文档结构、计算对应的CSS样式并生成RenderTree，然后根据RenderTree进行布局和绘制，基本过程示意图如下：

<figure>
	<img src="/images/webkitflow.png" alt="webkit flow">
</figure>

为了更简单的分析和定位渲染性能问题，我们将渲染流程抽象为五大步骤：

<figure style="width:300px;margin: 0 auto;">
	<img src="/images/render-flow.jpg" alt="render flow">
</figure>

>(1).Recalculate Style: 流水线中的第一步通常是使用javascript计算出需要如何操作DOM结构、并计算节点的最终样式规则

>(2).layout：第二步通常是根据节点的css规则，来计算节点在屏幕上位置和尺寸。由于页面是按着文档流从上到下、从左到有布局地，一个节点的布局发生变化，可能使得多个节点重新布局

>(3).update layer tree：一个页面可能有多个渲染层，layer tree用来维护各个渲染层的顺序

>(4).paint：绘制本质上就是填充像素的过程。包括绘制文字、颜色、图像、边框和阴影等，由此确定一个DOM元素所有的可视效果。绘制一般是在多个层(layer)上同时进行。

>(5).composite: 在多个层上分别完成绘制后，浏览器会按各个绘制层的正确顺序(layer tree中维持了各个图层的顺序)拼合成一个图层，最终显示在屏幕上。


理论上每一帧都要经过渲染流水线的处理，但渲染流水线中的有些步骤是可以跳过的。我们只修改节点的不影响布局的属性(背景图片、阴影等)时，就不需要重新layout了：

<figure style="width:300px;margin: 0 auto;">
	<img src="/images/render-flow-exlucde-layout.jpg" alt="render-flow-exlucde-layout">
</figure>

如果修改不触发绘制(直接在GPU中完成)的样式，比如transform、opacity等，甚至连paint都不需要了:

<figure style="width:300px;margin: 0 auto;">
	<img src="/images/render-flow-exlucde-layout-and-paint.jpg" alt="render-flow-exlucde-layout-and-paint">
</figure>


##2.监控动画性能

###(1) 使用chrome开发者工具
我们必须首先学会如何对动画的性能指标(帧率数、帧率稳定性)进行监控，才能有针对性的提高动画的性能。chrome开发者工具中的timeline是绝佳的工具，我们可以查看每一帧都经过渲染流水线的哪些步骤：

<figure>
	<img src="/images/chrome-timeline.jpg" alt="chrome-timeline">
</figure>

上图中，我选中了其中一帧，可以从最底部的Event Log中看到这一帧没有经过渲染流水线中的layout和paint阶段。

###(2) 通过时间戳计算帧率

chrome开发者工具中的timeline最大的问题就是其本身比较消耗资源，在开启timeline后，动画的帧率下降明显，因此其数据可能无法反映动画的正常运行情况。如果只是需要统计帧率，可以通过记录绘制每个帧消耗的时间来计算，[第三方库stats.js](https://github.com/mrdoob/stats.js)帮我们做了这些事情。下面是一个可视化的例子：

<a class="jsbin-embed" href="http://jsbin.com/ruxukiyejo/1/embed?output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?3.35.11"></script>


##3.提高动画性能指标

上文提到过，动画的性能指标有两个，帧率数和帧率稳定性。我们分别从动画实现，节点的处理，属性的选择等方面讨论如何提高这两个动画性能指标。

###(1).选择稳定的实现方式
css3动画使用起来非常简单，目前的浏览器支持率也不错，足以应对一般的交互需求，我们应该优先使用它。当浏览器不支持css3时，或动画场景过于复杂而仅凭css3无能为力时，就需要引入js来帮忙了。我们最常想到的js动画的实现方式，就是固定时间间隔修改元素的样式：

``` javascript
setInterval(function(){
    var anmationNode = document.getElementById('animation-node'); 
    //定期修改节点的样式
}, 100)
```

但这是一种非常粗暴的方式，其弱点是很明显的。浏览器的timer的触发时间点是不固定的，如果遇到比较长的同步任务，其触发时间点就会推迟，显然也就`保证不了动画帧率的平稳性`。HTML5为创建逐帧动画提供了一个新的API：`RequestAnimationFrame`，该方法在每次浏览器渲染时触发，其触发频率为60fps，我们可以通过这个函数来实现动画，而当动画中某些帧计算量太大无法在1/60s完成时，浏览器会将刷新评论降低到30fps，以保证帧率的稳定性。

```
function step(){
    //修改节点样式
    RequestAnimationFrame(step);
}
RequestAnimationFrame(step);
```

但是由于`RequestAnimationFrame`支持程度还不高(手机浏览器普遍不支持)，我们可以结合`RequestAnimationFrame`和`setInterval`实现一套逐渐增强和优雅降级的方案,下面是兼容各个浏览器的终极版本:

``` javascript
function getAnimationFrame() {
    if (window.requestAnimationFrame) { //较新浏览器
        return {
            request: requestAnimationFrame,
            cancel: cancelAnimationFrame,
        }
    } else if (window.mozRequestAnimationFrame && window.mozCancelAnimationFrame) { //firfox浏览器
        return {
            request: mozRequestAnimationFrame,
            cancel: mozCancelAnimationFrame
        }
    } else if (window.webkitRequestAnimationFrame && webkitRequestAnimationFrame(String)) {
        return {
            request: function(callback) {
                return: window.webkitRequestAnimationFrame(function() {
                    return callback(new Date - 0); //修正部分webkit版本下没有给callback传time参数的bug
                });
            },
            cancel: window.webkitCancelAnimationFrame || window.webkitCancelRequestAnimationFrame
        }
    } else { //用setInterval模拟requestAnimationFrame
        var millisec = 25; //40fps;
        var callbacks = [];
        var id = 0,
            cursor = 0;
        var timerId = null;

        function playAll() {
            var cloned = callbacks.slice(0);
            cursor += callbacks.length;
            callbacks.length = 0;
            var hits = 0;
            for (var i = 0, callback; callback = cloned[i++];) {
                if (callback !== 'cancelled') {
                    callback(new Data - 0);
                    hits++;
                }
            };
            if (hits == cloned.length) {
                clearInterval(timerId);
            }
        }

        timerId = window.setInterval(playAll, millisec);
        return {
            request: function(handler) {
                callbacks.push(handler);
                return id++;
            },
            cancel: function() {
                callbacks[id - cursor] = 'cancelled';
            }
        }
    }
}
```

###(2).为动画节点创建新的渲染层

通过将动画节点与文档中的其他节点隔离开来，可以有效的减少重新布局(relayout)和重新绘制(repaint)的面积，从而提高页面的整体性能。隔离动画节点与文档中的其他节点方法通常是为动画节点创建新的渲染层(render layer)。下面是创建渲染层的常用方法：

####<1> 使用3D变换

大家一定经常看到网上的文章说使用`transform: translate3d(0, 0, 0)/translateZ(0)`可以开启GPU加速，亲自试验以后发现其的确可以提高页面的渲染速度，我就曾经用它解决了一些低端机的闪烁问题。
那么其原理是什么呢？这种方式并非一定能够开启GPU加速。

W3C标准是这么说的。

>Three-dimensional transforms can result in transformation matrices with a non-zero Z component (where the Z axis projects out of the plane of the screen). This can result in an element rendering on a different plane than that of its containing block. This may affect the front-to-back rendering order of that element relative to other elements, as well as causing it to intersect with other elements.

其主要意思就是3D变换会创建新的渲染层，而不是与其父节点在同一个渲染层中。在新的渲染层中修改节点不会干扰到其他节点，防止了对其他节点的重新布局(relayout)和重新绘制(repaint)，自然也就加快了页面的渲染速度。除了`transform: translate3d(0, 0, 0)/translateZ(0)`，我们还可以使用`will-change`。

####<2> 使用will-change
我们可以使用will-change让浏览器提前了解预期的元素变换，它允许浏览器提前做好适当的优化，使之最后能够快速和流畅的渲染。`will-change: transform`同样也会为节点创建新的渲染层。
```
.animation-element{
    will-change: transform;
}
```

我们可以通过chrome的开发者工具中timeline的layers标签，看到当前帧的渲染层。如下图: 

<figure>
	<img src="/images/chrome-layers.png" alt="chrome-layers">
</figure>

上图中右侧有对创建layer原因的描述: has a will-change hint。但是管理渲染层是有成本的，过多的渲染层可能会降低页面的渲染速度，因此我们应该避免滥用渲染层。

###(3).选择高效的动画属性

修改节点的大部分属性都会引起重新绘制，甚至是重新布局。而理想情况下，我们应避免重新绘制和重新布局。幸运的当仅仅修改`transfrom`属性或`opacity`属性，可以做到不重新绘制。具体的思路是：为需要创建动画的节点创建新的渲染层，并且在新渲染层中只修改`transform`和`opacity`属性。`只有做到以上两点才可以避免重新布局和重新绘制，真正使用GPU加速。`

###(4).避免引起多余的渲染

我们在实现动画的过程中，经常需要获取某个元素的属性，然后对该属性做出修改：

``` javascript
function step(){
    var animationNode = doucment.getElementById('animation-node');
    for(var i = 1; i <= 20 ; i++){
        animationNode.width = animationNode.width + 1；
    }
}
```

上述的for循环语句将导致浏览器进行20次多余的渲染，严重影响页面性能。通常来讲JS对页面样式的多次修改只会在页面下次刷新时渲染一次，而通过DOM API获取样式时，会强制页面完成一次渲染以体现最新修改后的值。上述例子就是这样导致浏览器多次渲染的。而正确的写法应该是读写分离。
 
``` javascript
var animationNode = doucment.getElementById('animation-node');
var initialWidth = animationNode.style.width;
for(var i = 1; i <= 20 ; i++){
    initialWidth+=1；
}
animationNode.style.width = initialWidth;
```

当我们在复杂页面上实现动画是，常常由于疏忽导致页面多余的渲染。这是我们可以借助[fastdom](https://github.com/wilsonpage/fastdom)来隔离对真实DOM的操作，[fastdom](https://github.com/wilsonpage/fastdom)将对节点样式的读写批量缓存、一次执行，防止多余的渲染。

---
参考资料

[http://taligarsiel.com/Projects/howbrowserswork1.htm](http://taligarsiel.com/Projects/howbrowserswork1.htm)

[http://matrix.h5jun.com/slide/show?id=117#/](http://matrix.h5jun.com/slide/show?id=117#/)

[http://melonh.com/sharing/slides.html?file=high_performance_animation#/](http://melonh.com/sharing/slides.html?file=high_performance_animation#/)

[http://www.infoq.com/cn/articles/javascript-high-performance-animation-and-page-rendering](http://www.infoq.com/cn/articles/javascript-high-performance-animation-and-page-rendering)

[https://developers.google.com/web/fundamentals/performance/?hl=zh-cn](https://developers.google.com/web/fundamentals/performance/?hl=zh-cn)

[https://github.com/wilsonpage/fastdom](https://github.com/wilsonpage/fastdom)

[https://github.com/mrdoob/stats.js](https://github.com/mrdoob/stats.js)
