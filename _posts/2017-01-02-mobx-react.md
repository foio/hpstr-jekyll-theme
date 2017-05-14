---
layout: post
title: 使用mobx开发高性能react应用
description: "mobx与react性能优化，mobx最佳实践，mobx与redux对比"
modified: 2017-01-02
tags: [mobx,react,redux]
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


>react作为模块化的UI层框架，在前端领域正处于如日中天的地位。但如果仅仅使用react，往往需要在UI层中承载过多的业务逻辑，引入模块化的同时却破坏了分层。为此业界有很多解决方案，目前最流行的就是redux，关于redux的详细介绍请参考我的另外两篇文章：[redux,一种页面状态管理的优雅方案](http://foio.github.io/redux-state-manage/)和[react+redux渲染性能优化原理](http://foio.github.io/react-redux-performance-boost/)。redux是一个设计规范、严格的单向数据流框架，适用于大型项目。而本文将详细介绍一种更灵活的、适合于中小型应用的数据层框架mobx。

本文中的代码全部源于[react-mobx-isomorphic-todolist](https://github.com/foio/react-mobx-isomorphic-todolist)。

##1.mobx的基本用法

作为一个数据层框架，mobx基于一个最简单的原则：

>当应用状态更新时，所有依赖于这些应用状态的监听者（包括UI、服务端数据同步函数等），都应该自动得到细粒度地更新。

mobx推荐使用ES7的decorator语法，以实现最精炼的表达。通过对babel简单的配置我们就可以使用decorator的语法了，具体配置请参考[官方文档](https://mobxjs.github.io/mobx/best/decorators.html)。在理解mobx之前，我们需要搞清楚mobx中的两个基本概念Observable和Reactions：

>Observable: 需要被监听的应用状态；通过`@observable`修饰符可以细粒度地控制一个Class的哪个属性需要被监听。

>Reactions: 应用状态的监听者；当依赖的应用状态发生变化时，能够自动地执行相应的动作。`reaction`、`autorun`、`@observer`都可以生成一个Reactions。

mobx通过使用Observable和Reactions的抽象概念，分别表示需要被监听应用状态和应用状态的监听者。

###(1). 创建需要被监听的应用状态

通过对Class的属性简单的使用`@observable`修饰符，就定义了一个需要被监听的应用状态变量；然后直接在类中定义对应用状态变量的操作；我们就实现了一个灵活的Store层。如下代码中，创建了一个TodoItemModel的Store层：使用`@observable`修饰符将title和completed定义为需要被监听地应用状态变量。这样，mobx就会监听title和completed的变化，并在其发生变化时，执行这些变量的监听者(Reactions)。

``` javascript
class TodoItemModel {
    id;
    @observable title;
    @observable completed;

    toggle() {
        this.completed = !this.completed;
    }

    setTitle(title) {
        this.title = title;
    }

    ...
}

```

###(2).创建应用状态的监听者

上文中提到，有多种方式可以创建应用状态的监听者(Reactions)，包括`autorun`、`reaction`、`@observer`等，具体用法请参考官方文档。Reactions中有应用状态（使用`@observable`修饰符标记的变量）的依赖，在应用状态更新时，会自动执行相应的动作，比如：打印日志、向服务端同步数据、更新UI等。下面是几种常用的监听者(Reactions)。

使用autorun在应用状态变化时打印日志：

``` javascript
autorun(() => console.log(this.completed);
```

使用reaction在应用状态变化时向服务端同步数据:

``` javascript
reaction(
    () => this.toJS(),
    todo => fetch('/todos/' + todo.id, {
        method: 'PUT',
        headers: new Headers({'Content-Type': 'application/json'}),
        body: JSON.stringify(todo)
    })
)
```

使用@observer在应用状态变化时更新UI，和React组件结合使用，会在应用状态更新时，重新渲染组件：

``` javascript
@observer
class TodoItem extends React.Component {
    .....
    render() {
        //UI logic code ...
        return (
 			....
        );
    }
}
```

在使用`autorun`、`reaction`、`@observer`创建监听者(Reactions)时，我们并没有显示的指定当前的监听者(Reactions)监听的是哪个应用状态(Observable)，mobx却能够做到精确地更新，比如如下代码，reaction1和reaction2只有在各自依赖的应用状态变量变动时才会执行：

``` javascript
class TodoItemModel {
    id
    @observable title;
    @observable completed;
    ......
}

var todoItemStore = new TodoItemModel();

//只有在title发生变化时，reaction1才会执行
const reaction1 = reaction(
    () => todoItemStore.title,
    text => console.log(id+":"+text)
);

//只有在completed发生变化时，reaction2才会执行
const reaction1 = reaction(
    () => todoItemStore.completed,
    text => console.log(id+":"+completed)
);

```

mobx是如何做到：只有在监听者(Reactions)监听的应用状态(Observable)发生变化时，才执行监听者的动作的呢？

>mobx会记录监听者(Reactions)中对Observable变量的引用，通过引用在运行时动态地构建依赖图谱，从而实现精确的更新。这样mobx就可以保证某个Observable变量变化时，只执行对其有依赖的Reactions动作。

mobx借鉴了MVVM框架中常用的依赖收集技术，使用ES5的新特性Object.defineProperty对属性设置set和get来实现对对象属性变动地监听和依赖跟踪。关于Object.defineProperty的请参考[MDN文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。


##2.mobx-react渲染性能优化原理

我们可以通过使用@observer，将react组件转换成一个监听者(Reactions)，这样在被监听的应用状态变量(Observable)有更新时，react组件就会重新渲染。如下代码中，TodoItemModel中的应用状态变量有更新时，TodoItem UI会重新渲染:


``` javascript
class TodoItemModel {
    id
    @observable title;
    @observable completed;
    ......
}


@observer
class TodoItem extends React.Component {
    .....
    render() {
        //UI logic code ...
        let todo = this.props.todo;
        let title = todo.title;
        let complete = todo.complete;
        return (
 			....
        );
    }
}

var todoItem = new TodoItemModel();
todoItem.title = '';

```


在使用react的过程中，我们绕不开渲染性能优化问题，因为默认情况下react组件的shouldComponentUpdate函数会一直返回true，这回导致所有的组件都会进行耗时的虚拟DOM比较。在使用redux作为react的逻辑层框架时，我们可以使用经典的PureComponent+ShallowCompare的方式进行渲染性能优化，那么在使用mobx作为react的store时，我们该如何进行渲染性能优化呢？

###(1). 默认的基本渲染性能优化

通过分析源代码发现，在使用@observer将react组件转换成一个监听者(Reactions)后，mobx会为react组件提供一个精确的、细粒度的shouldComponentUpdate函数:

``` javascript
shouldComponentUpdate: function(nextProps, nextState) {
  ......
  // update on any state changes (as is the default)
  if (this.state !== nextState) {
    return true;
  }
  // update if props are shallowly not equal
  return isObjectShallowModified(this.props, nextProps);
}

```

借助于mobx框架对Observable变量引用的跟踪和依赖收集，mobx能够精确地得到react组件对Observable变量的依赖图谱，然后再用经典的ShallowCompare实现细粒度的shouldComponentUpdate函数，以达到100%无浪费render。这一切都是自动完成地，fantastic！使用mobx后，我们再也无需手动写shouldComponentUpdate函数了。


###(2). 使用transaction进行高级渲染性能优化

mobx在带来便利性的同时，也可能引入新的问题。上文反复提到：mobx会记录监听者(Reactions)中对Observable变量的引用，通过引用在运行时动态地构建依赖图谱，从而实现精确地、细粒度地更新。

如果一个React组件依赖于多个应用状态(Observable)变量，而一次操作需要更新多个应用状态(Observable)变量时，React组件就会进行多次渲染。在有些场景下，更新地粒度过细，也不是我们希望看到的场景。比如如下代码，调用TodoItemModel的reset方法设置了两个应用状态变量，会导致对应的UI组件重新渲染两次：


``` javascript

//todoItem UI组件，依赖于应用状态变量completed和title
@observer
class TodoItem extends React.Component {
    ...
    render() {
        //UI logic code ...
        let todo = this.props.todo;
        let title = todo.title;
        let completed = todo.complete;
        return (
            ......
        );
    }
}


class TodoItemModel {
    id;
    @observable title;
    @observable completed;
    ......
    reset() {
     	/*
     	分别设置title和completed值，
        会触发两次todoItem UI组件的重新渲染
        */
        this.completed = false;
        this.title= '';
    }
    ......
}

var todoItem = new TodoItemModel();
//调用reset将触发两次TodoItem UI组件的重新渲染
todoItem.reset();
```

![mobx-transaction-1](/images/mobx-transaction-1.png)

在语义层面，有时我们需要将多个应用状态(Observable)的更新，`视为一次操作，并希望只触发一次监听者(Reactions)的动作(UI更新、网络请求等)`。为此mobx提供了transaction功能，可以将对多个应用状态(Observable)的更新封装成一个事务，只有在事务执行完成后，才会触发一次对应的监听者(Reactions)。这就使得我们对组件的渲染有更精细化的控制。比如如下代码，使用transaction后，对应用状态(Observable)的两次更新只会触发一次UI的更新。

``` javascript
class TodoItemModel {
    id;
    @observable title;
    @observable completed;
    ......
    reset() {
     	/*
     	使用transaction设置title和completed值，
        只会在函数调用结束后触发一次todoItem UI组件的重新渲染
        */
        transaction(()=>{
            this.completed = false;
        	this.title= '';
        })
    }
    ......
}

var todoItem = new TodoItemModel();
//调用reset只触发一次TodoItem UI组件的重新渲染
todoItem.reset();
```

![mobx-transaction-2](/images/mobx-transaction-2.png)

##3.最佳实践

上文中提到，一般情况下，mobx和react结合时不需要任何的额外操作，就能够使我们的react组件达到100%无浪费渲染。但这是建立在正确的使用mobx的基础上的，mobx极其灵活，也容易误用。下面我们总结一些mobx的最佳实践。

###(1). 延迟对象属性地解引用

本质上，mobx使用ES5的新特性Object.defineProperty对属性设置set和get来实现对对象属性变动的监听和依赖跟踪。只有在真正需要使用Observable变量的Reactions中，再对其解引用，才能使得mobx构建出正确的依赖图谱。从而使得mobx能够精确的更新对特定属性有依赖的Reactions。

比如对于todolist，正确的写法是在子组件中才对需要使用的属性进行解引用：

``` javascript
@observer
class MainSection extends React.Component {
    .......
    render() {
        ......
        return (
                ......
                    {todos.map(
                        (todo) => <TodoItem key={todo.id} todo={todo} />
                    )}
                ......
        );
    }
}

@observer
class TodoItem extends React.Component {
    ......
    render() {
        let todo = this.props.todo;
        let title = todo.title;
        let complete = todo.complete;
        return(
            ......
        ) 
    }
}

```

如果在父组件中对属性过早的解引用，而向子组件传递原始类型的变量，则导致mobx无法搜集到子组件对应用状态的依赖，如下代码是错误的：

``` javascript
@observer
class MainSection extends React.Component {
    .......
    render() {
        ......
        return (
                ......
                    {todos.map(
                        (todo) => <TodoItem key={todo.id} title={todo.title} completed={todo.completed}/>
                    )}
                ......
        );
    }
}

@observer
class TodoItem extends React.Component {
    ......
    render() {
        let title = this.props.title;
        let complete = this.props.complete;
        return(
            ......
        ) 
    }
}

```


###(2). 不要吝啬使用@observer

github上有个经典的问题：[Do child components need `@observer`](https://github.com/mobxjs/mobx/issues/101)？在回答这个问题的过程中，mobx的作者对mobx的设计思想也进行了很好的回答。我最初接触mobx时，我也有同样的疑惑：子组件需要使用`@observer`标记吗？应用状态可以通过父组件传递到子组件啊？

mobx作者[mweststrate](https://github.com/mweststrate)推荐对父组件和子组件都使用`@observer`。添加`@observer`对性能的影响可以忽略不计，不需要担心由此引发的性能问题，事实上作者在自己的项目中对所有的组件都是用`@observer`，也没有问题。

```
如果一个组件需要使用`@observable`变量（应用状态），就应该使用@observer修饰符。
```
上文`mobx-react渲染性能优化原理`一节中提到，借助于精确的依赖分析，mobx可以得出组件对`@observable`变量（应用状态）的依赖图谱，对使用`@observer`进行标记的组件，实现精准的shouldComponentUpdate函数，保证组件100%无浪费渲染。如果我们不对子组件使用@observer，我们就放弃了mobx对React组件的默认性能优化。


###(3). 不要吝啬使用@action

mobx2.2版本开始引入action，我们在使用mobx时，应该遵守如下规则:

```
凡是涉及到对应用状态变量修改的函数，都应该使用@action修饰符。
```
使用action后我们可以更清晰的看出，代码中的那一部分修改了`@observable`变量（应用状态）; mobx的官方调试工具，也能够提供更丰富的调试信息。在mobx-react高级渲染性能优化小节中，我们知道，使用transaction可以将多个应用状态(Observable)的更新视为一次操作，并只触发一次监听者(Reactions)的动作(UI更新、网络请求等)，从而更大程度地提升应用的性能，避免多余的UI渲染和网络请求。action中封装了transaction，对函数使用action修饰符后，无论函数中对`@observable`变量（应用状态）有多少次修改，都只会在函数执行完成后，触发一次对应的监听者。如下代码，reset函数只会触发一次UI更新。

``` javascript
@autobind
class TodoItemModel {
    id;
    @observable title;
    @observable completed;

    //使用action后，reset函数执行完成后，才会触发一次其监听者
    @action
    reset() {
        this.completed = false;
        this.title= '';
    }
}

```


需要注意的是action只能影响同步函数，函数中如果有异步调用，则需要对异步调用的函数也使用aciton包裹。请[参考官方文档](http://mobxjs.github.io/mobx/refguide/action.html)。



###(4). 服务端渲染时设置useStaticRendering为true

上文中，我们知道可以通过使用@observer，将react组件转换成一个监听者(Reactions)，这样在被监听的应用状态变量(Observable)有更新时，react组件就会重新渲染。而对于服务端的React组件，我们只需要它被渲染一次，而不需要组件监听模型的状态。事实上，如果服务端React组件像客户端组件一样监听模型的状态变化，就会造成严重的内存泄漏问题。官方提供了`useStaticRendering`方法，用于避免mobx服务端渲染的内存泄漏问题; 该方法只需要在server启动时设置一次。

``` javascript
useStaticRendering(true);
```

##4.mobx与redux的对比

不可否认，redux依然是react生态系统中最流行的数据层框架；但是mobx也有自己的优势。下面我们从完整性、状态传播方式、适用性等方面对mobx和redux进行简单的对比。

|        | redux                                      | mobx                 |
|:-------------:|:------------------------------------------|:-------------------|
| 逻辑层完整性性 | redux包括了store、view、action的整个数据流| mobx只关心store、view|
| 状态传播  | redux将组件分为容器型组件和展示型组件，状态只对容器型组件可见，容器型组件需要主动的使用mapStateToProps订阅状态| mobx对组件没有划分，使用@observer后会在运行时动态地订阅状态 |
| 适用性 | redux要求对整个应用使用单一的状态树，并要求状态的更新必须是immutable地，数据流规范，设计严谨。但是其复杂度也比较高，就连最基本的异步操作，也需要中间件(redux-thunk,redux-sage等)来解决。学习曲线较陡，适合大型项目使用 | mobx则对应用状态的设计没有特殊要求，极其灵活，异步操作也非常自然，学习曲线平缓，适合中小型项目,但缺少最佳实践|


参考文档：

----

https://github.com/foio/react-mobx-isomorphic-todolist

https://jsfiddle.net/mweststrate/wv3yopo0/

https://mobxjs.github.io/mobx/refguide/observable.html

http://www.cnblogs.com/rubylouvre/p/6058045.html

https://github.com/sorrycc/blog/issues/2

https://github.com/mobxjs/mobx-react-boilerplate

https://github.com/mobxjs/mobx/issues/101

https://www.mendix.com/tech-blog/making-react-reactive-pursuit-high-performing-easily-maintainable-react-apps/

https://mobxjs.github.io/mobx/best/devtools.html

https://medium.com/@robinpokorny/index-as-a-key-is-an-anti-pattern-e0349aece318#.zdtoxpkr8

https://mobxjs.github.io/mobx/best/react-performance.html

https://mobxjs.github.io/mobx/best/store.html

https://github.com/sorrycc/blog/issues/5?utm_source=tuicool&utm_medium=referral

https://medium.com/@mweststrate/becoming-fully-reactive-an-in-depth-explanation-of-mobservable-55995262a254#.vaxmn2rek
