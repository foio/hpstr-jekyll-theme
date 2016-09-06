---
layout: post
title: react+redux渲染性能优化原理
description: "react渲染性能优化,redux渲染性能优化"
modified: 2016-09-04
tags: [node memory]
image:
  background: triangular.png
comments: true
---


<style type="text/css">
td, table {
   border: solid gray 1px;
   padding: 10px;
}
table{
   margin: 0 auto;
}
</style>

>React基本上成了前端的必备技能，redux更是对react的锦上添花。我在[“redux,一种页面状态管理的优雅方案”](http://foio.github.io/redux-state-manage/)一文中介绍了react与redux结合的基本方法和高级技巧。

大家都知道，react的一个痛点就是非父子关系的组件之间的通信，其官方文档对此也并不避讳：

>For communication between two components that don't have a parent-child relationship, you can set up your own global event system. Subscribe to events in componentDidMount(), unsubscribe in componentWillUnmount(), and call setState() when you receive an event. 


而redux就可以视为其中的“global event system”，使用redux可以使得我们的react应用有更加清晰的架构。

本文我们来探讨，基于react和redux架构的前端应用，如何进行渲染性能优化。`对于小型react前端应用，最好的优化就是不优化`因为React本身就是通过比较虚拟DOM的差异，从而对真实DOM进行最小化操作，小型React应用的虚拟DOM结构简单，虚拟DOM比较的耗时可以忽略不计。而对于复杂的前端项目，我们所指的渲染性能优化，实际上是指，`在不需要更新DOM时，如何避免虚拟DOM的比较`。

###1. react组件的生命周期

工欲善其事，必先利其器。理解react的组件的生命周期是优化其渲染性能的必备前提。我们可以将react组件的生命周期分为3个大循环：挂载到DOM、更新DOM、从DOM中卸载。React对三个大循环中每一步都暴露出钩子函数，使得我们可以细粒度地控制组件的生命周期。

####(1)挂载到DOM
组件首次插入到DOM时，会经历从属性和状态初始化到DOM渲染等基本流程，可以通过下图描述：

<figure style="width:300px;margin: 0 auto;">
        <img src="/images/react-component-mount.png" alt="render flow">
</figure>


必须注意的是，挂载到DOM流程在组件的整个生命周期只有一次，也就是组件第一次插入DOM文档流时。在挂载到DOM流程中的每一步也有相应的限制：

```
getDefaultProps()和getInitialState()中不能获取和设置组件的state。
render()方法中不能设置组件的state。
```

####(2)更新DOM
组件挂载到DOM后，一旦其props和state有更新，就会进入更新DOM流程。同样我们也可以通过一张图清晰的描述该流程的各个步骤：

<figure style="width:300px;margin: 0 auto;">
        <img src="/images/react-component-update.png" alt="render flow">
</figure>

componentWillReceiveProps()提供了该流程中更新state的最后时机，后续的其他函数都不能再更新组件的state了。`我们尤其需要注意的是shouldComponentUpdate函数，它的结果直接影响该组件是不是需要进行虚拟DOM比较`，我们对组件渲染性能优化的基本思路就是：在非必要的时候将shouldComponentUpdate返回值设置为false，从而终止更新DOM流程中的后续步骤。


####(3)从DOM中卸载

从DOM中卸载的流程比较简单，React只暴漏出`componentWillUnmount`，该函数使得我们可以在DOM卸载的最后时机对其进行干预。

###2. react组件渲染性能监控

在进行性能优化前，我们先来了解如何对React组件渲染性能进行监控。React官方提供了[Performance Tools](https://facebook.github.io/react/docs/perf.html)，其使用起来也很简单，通过Perf.start启动一次性能分析，并通过Perf.stop结束一次性能分析。

``` javascript
import Perf from 'react-addons-perf'
Perf.start();
....your react code
Perf.stop();
```
调用Perf.stop后，我们就可以通过Perf提供的API来获取本次性能分析的数据指标。其中最有用的API是`Perf.printWasted()`，其结果给出你在哪些组件上进行了无意义的(没有引起真实DOM的改变)虚拟DOM比较，比如如下结果表明我们在TodoItem组件上浪费了4ms进行无意义的虚拟DOM比较，我们可以从这里入手，进行性能优化。

![Alt text](/images/react-perf-wasted.png)

而`Perf.printInclusive()`的结果则给出渲染各个组件的总体时间，通过它的结果我们可以找出哪个组件是页面渲染的性能瓶颈。

![Alt text](/images/react-perf-include.png)

和`Perf.printInclusive()`相似的API还有`Perf.printExclusive()`，只是其结果是组件渲染的独占时间，即不包括花费于加载组件的时间： 处理 props, getInitialState, 调用 componentWillMount 及 componentDidMount, 等等。

###3. 性能优化基本原理
使用上一小节的性能分析工具，我们可以轻易的定位出哪些组件是页面的性能瓶颈、哪些组件进行了无意义的虚拟DOM比较，本小节我们能探讨如何对基于react和redux架构的前端应用进行性能优化。强烈的建议你参考我的另一篇博文:[redux,一种页面状态管理的优雅方案](http://foio.github.io/redux-state-manage/)，以便更好的理解本小节。

####3.1 常规React组件性能优化

通过上文的React更新DOM流程，我们知道React提供了`shouldComponentUpdate`函数，它的结果直接影响组件是不是需要进行虚拟DOM比较以及后续的真实DOM渲染。而`shouldComponentUpdate`函数的默认返回值为true，这暗示着React总是会进行虚拟DOM比较，无论真实DOM是否需要重新渲染。我们可以通过根据自己的业务特性，重载`shouldComponentUpdate`，只在确认真实DOM需要改变时，再返回true。一般的做法是比较组件的props和state是否真的发生变化，如果发生变化则返回true，否则返回false。

``` javascript
 shouldComponentUpdate: function (nextProps, nextState) {
     return !isDeepEqual(this.props,nextProps) || !isDeepEqual(this.state,nextState);
 }
```

进行深度比较(isDeepEqual)来确定props和state是否发生变化是最常见的做法，其是否有性能问题呢？如果一个容器型组件有很多的子节点，而子节点又有其他子节点，对这种复杂的嵌套对象进行深度比较(isDeepEqual)是很耗时的，甚至会抵消由避免虚拟DOM比较所带来的性能收益。React官方推荐使用[immutable的组件状态](https://facebook.github.io/react/docs/advanced-performance.html)，以便更高效的实现shouldComponentUpdate函数。

immutable的状态有何优势呢？假设我们要修改一个列表中，某个列表项的状态，使用非immutable的方式：

``` javascript
var item = {
	id:1,
	text:'todo1',
	status:'doing'
}
var oldTodoList = [item1,item2,....,itemn];
oldTodoList[n-1].status = 'done';
var newTodoList = oldTotoList;
```

当我们需要确认oldTodoList和newTodoList的数据是否相同时，只能遍历列表(复杂度为O(n))，依次比较:

``` javascript
for(var i = 0; i < oldTodoList.length; i++){
	if(isItemEqual(oldTodoList[i],newTodoList[i])){
		return true;
	}
}
return false;
```

而如果使用immutable的方式：

``` javascript
var newTotoList = oldTodoList.map(function(item){
	if(item.id == n-1){
		return Object.assign({},item,{status:'done'})
	}else{
		return item;
	}
});
```
因为每一次变动，都会创建新的对象，因此比较oldTodoList和newTodoList是否有变化时，只需要比较其对象引用即可(复杂度O(1))：

``` javascript
return oldTodoList == newTodoList;
```

`我们优化的方向就是将`shouldComponentUpdate`中所有的props和state的比较算法复杂度降到最低`，而浅层对比(isShallowEqual)就是复杂度最低的对象比较算法:

``` javascript
 shouldComponentUpdate: function (nextProps, nextState) {
     return !isShallowEqual(this.props,nextProps) || !isShallowEqual(this.state,nextState);
 }
```

当组件的prop设state都是immutable时，`shouldComponentUpdate`的实现就非常简单了，我们可以直接使用facebook官方提供了`PureRenderMixin`，它就是对组件的props和state进行浅层比较的。

``` javascript
var PureRenderMixin = require('react-addons-pure-render-mixin');
React.createClass({
  mixins: [PureRenderMixin],

  render: function() {
    return <div className={this.props.className}>foo</div>;
  }
});
```

自己实现immutable化，还是很有挑战的，我们可以借助于第三方库[ImmutableJS](https://facebook.github.io/immutable-js/)，它是一个重型库，适合于大型复杂项目；如果你的项目复杂度不是很高，可以使用[seamless-immutable](https://github.com/rtfeldman/seamless-immutable)，它是一个更轻量级的库，基于ES5的新特性Object.freeze来避免对象的修改，因此其只能兼容实现ES5标准的浏览器。

####3.2 理解Redux状态传播路径
Redux使用一个对象存储整个应用的状态(`global state`)，当`global state`发生变化时，状态是如何传递的呢？这个问题的答案对我们理解基于redux的react应用的渲染性能优化至关重要。

Redux将React组件分为容器型组件和展示型组件。容器型组件一般通过connet函数生成，它订阅了全局状态的变化，通过mapStateToProps函数，我们可以对全局状态进行过滤，只返回该容器型组件关注的局部状态：

``` javascript
function mapStateToProps(state) {
    return {todos: state.todos};
}
module.exports = connect(mapStateToProps)(TodoApp);
```

`每一次全局状态变化都会调用所有容器型组件的mapStateToProps方法`，该方法返回一个常规的Javascript对象，并将其合并到容器型组件的props上。


而展示型组件不直接从`global state`获取数据，其数据来源于父组件。当容器型组件对应`global state`有变化时，它会将变化传播到其所有的子组件(一般为展示型组件)。简单来说容器型组件与展示型组件是父子关系：

| 组件类型      | 数据来源      |变化通知  |
|:------------- |:-------------:|:------------:|
| 展示型组件    | 父组件        | 父组件通知   |
| 容器型组件    | 全局状态      | 监听全局状态 |


组件的状态传递路径，可以用一个树形结构描述：

![Alt text](/images/redux-component-tree.png)

####3.3 理解Redux的默认性能优化


Redux官方对容器型组件和全局状态树有两个基本的假设，违背这些假设将使得Redux的默认性能优化无法起作用：

```
1. 容器型组件必须为Pure Component，即组件只依赖于state和props
2. 全局状态树(global state)的任何变动都是immutable的
```

>这种规范是有理由的：上文中我们提到过，每一次全局状态发生变化，所有的容器型组件都会得到通知，而各个容器型组件需要通过shouldComponentUpdate函数来确实自己关注的局部状态是否发生变化、自身是否需要重新渲染，默认情况下，React组件的shouldComponentUpdate总返回true，这里貌似有一个严重的性能问题：全局状态的任何变动都会使页面中的所有组件进入`更新DOM`的流程

幸运的是，用Redux官方API函数connect生成的容器型组件，默认会提供一个shouldComponentUpdate函数，其中对props和state进行了浅层比较`。如果我们不遵从Redux的immutable状态的规范和Pure Component规范，则容器型组件默认的shouldComponentUpdate函数就是无效的了。


在遵从Redux的immutable状态规范的情况下，当一个容器型组件的默认shouldComponentUpdate函数返回true时，则表明其对应的局部状态发生变化，需要将状态传播到各个子组件，相应的所有子组件也都会进行虚拟DOM比较，以确定是否需要重新渲染。如下图所示，`容器型组件#1`的状态发生变化后，所有的子组件都会进行虚拟DOM比较：

![Alt text](/images/redux-performance-loose.png)

`由于展示型组件对全局状态没有感知，我们就可以使用React的常规方法对展示型进行渲染性能优化了`。使用小节3.1中所提到的`常规React组件性能优化`方案，对每一个展示型组件实现shouldComponentUpdate函数：

``` javascript
 shouldComponentUpdate: function (nextProps, nextState) {
     return !isShallowEqual(this.props,nextProps) || !isShallowEqual(this.state,nextState);
 }
```

我们就可以避免展示型组件多余的虚拟DOM比较。比如当只有`展示型组件#1.1`需要重新渲染时，其他同级别的组件不会进行虚拟DOM比较。比如当只有`展示型组件#1.1`需要重新渲染时，其他同级别的组件不会进行虚拟DOM比较了

![Alt text](/images/redux-performance-best.png)

综上所述: 在容器型组件层面，Redux为我们提供了默认的性能优化方案；在展示型组件层面，我们可以使用常规React组件性能优化方案。

---
参考文献：

https://facebook.github.io/react/docs/component-specs.html

https://facebook.github.io/react/docs/perf.html

http://benchling.engineering/performance-engineering-with-react/

http://jlongster.com/Using-Immutable-Data-Structures-in-JavaScript

https://github.com/rtfeldman/seamless-immutable

https://facebook.github.io/react/docs/pure-render-mixin.html

https://facebook.github.io/react/docs/advanced-performance.html

https://www.toptal.com/react/react-redux-and-immutablejs

https://facebook.github.io/react/docs/advanced-performance.html
