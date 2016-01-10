---
layout: post
title: React简述
category: 技术
tags: React demo 参考
description: 参考demo, 三分钟就会写简单react了
---

### 目的

为了让自己任何时候看见这篇blog时, 都能在五分钟内拿起React干些简单的活儿。

### 个人理解

React并不是一个MV*框架，不该去和Angular,vue比较。
应该拿她和JqueryUI来比较，他们都是用于构建组件的库。专注于从组件层面来构造web应用。

但是它比起jqueryUI有一些新奇的特性：

1. **JSX语法糖**。
  React增加了名为JSX的语法糖，通过其你可以方便得编写类似原生html字符的代码。
  JSX转换库：https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js
  React在线CDN：https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/react.js

2. **虚拟dom**。
  特性一提及的类似原生html的字符就是react的虚拟dom，我们对虚拟dom操作需要遵循react规定的规则。
  相对的，虚拟dom也带来了普通dom所不具有的需对优势。
  比如虚拟dom更新view的时候，通过react的调度，会选择最合理的操作方式，从而提升整体性能。

3. **状态更新**。、
  组件渲染时会绑定react组件对象的state和props，数据的改变会自动触发组件的update。
  这个就类似于MV*框架的双向绑定了。
  相比jqueryUI就没有自动更新功能，内部数据变更之后只能手动指定渲染。

react的最大优势就在于其所维护的虚拟dom，这是一个新的理念。

当年jquery一统江山靠的就是自己的一套jquery对象概念。解决浏览器兼容的同时还把dom操作玩到极致。怎么能不风靡。

随着react的虚拟dom理念慢慢被发掘，我们有理由相信react也会展翅高飞。


### 一个简单的demo带你了解react

我们构建一个最简单的react环境，用`type="text/jsx"`来表明我们编写的`script`是用jsx语法糖解析的js文件.

![react基础环境](http://7xny7k.com1.z0.glb.clouddn.com/reactEnvironment.png)

我们想实现两个自定义组件, 用react来做些常见的前端操作。

就是下面这玩意:
![react入门demo](http://7xny7k.com1.z0.glb.clouddn.com/demoReact.png)

他们的功能和一些知识点分别是:

1. 显示隐藏按钮`button`:
  * A.用react来自定义类名
  * B.用react来编写样式
  * C.用react来绑定点击事件
  * D.监听了react组件的mount方法, 在render前后会有log
  * E.学会react组件虚拟dom和真实dom的引用查找方法


2. 自定义输入框`input`:
  * F.用react来显示在render调用传递进来的prop属性值
  * G.用react绑定onchange事件, 改变input的值时, 会自动被react更新
  * H.监听了react组件的update方法, 在update前后也会有log
  * I.了解虚拟dom内部是{}来写表达式的
  * J.render方法写虚拟dom永远只能是单个元素


代码如下:

    //自定义组件<ClickButton>
    var ClickButton = React.createClass({ 
      render : function(){
        var styleObj = {
          backgroundColor : '#333' , //B.样式对象, 样式全部用驼峰命名, 和操作原生dom.style一样
          color: '#dddddd'
        }
        //E.子组件的ref属性是用来方便选择与定位
        //B.样式需要指定style对象, 或自己写一个{}对象
        //A.类名控制需要用className属性来控制
        return ( 
          <div>
            <button onClick={this.showOrHide} style={styleObj} className="likeThis">显示 | 隐藏</button><span ref="tip">测试点击能不能显示和隐藏</span> 
          </div>
        ) ; //C.onClick等驼峰式绑定方法是react定义事件的方式, 编写方式类似原生绑定, 但因为这里是虚拟dom, 所以自由度很高且没有一系列问题
      },
      showOrHide: function(event){ //C.绑定的click方法
        var tipE = React.findDOMNode( this.refs.tip ) ; //E. this.refs[ 子组件的ref属性的名字 ], 这个获取到虚拟的dom, 然后用findDOMNode来获取到真实dom
        if ( tipE.style.display === 'none'){
          tipE.style.display = 'inline' ;
        }else{
          tipE.style.display = 'none' ;
        }
        event.stopPropagation() ; //这个event是react封装后的event方法, 比原生的强
        event.preventDefault() ;
      },
      //D.注一: 生命周期一Mount
      componentWillMount: function(){
        console.log( 'will Mount' ) ;
      },
      componentDidMount: function(){
        console.log( 'did Mount' ) ;
      },
    
    }) ;
    //自定义组件<InputArea>
    var InputArea = React.createClass({
      getInitialState: function(){
        return {
          inputValue : '' ,
        }
      },
      render : function(){
        return (
          <div>
            <input type="text" onChange={this.changeTip} /><span>默认值: {this.props.defaultValue} 内部属性inputValue: {this.state.inputValue}</span>
          </div>
        ) ; //I.{}这个符号表示其内部是表达式
      },
      changeTip : function(event){ //G.onChange时触发
        this.setState({
          inputValue : event.target.value 
        }) ;
        event.stopPropagation() ;
        event.preventDefault() ;
      },
      //H.注二: 生命周期二Update
      componentWillUpdate: function(){
        console.log( 'will update' ) ;
      },
      componentDidUpdate: function(){
        console.log( 'did update' ) ;
      }
    }) ;
    //J.render们接受的虚拟dom元素都必须是单个元素, 不能是多个元素, 所以这里要渲染两个元素必须得用个div包裹起来
    //F.其上的属性可以在render时用this.prop获取到
    React.render( ( 
      <div>
        <ClickButton />
        <InputArea defaultValue="render调用时定义" />
      </div>
    ) , document.getElementById('container') )
    

注一:组件生命周期第一步: `Mount`
第一阶段是render前后, hock函数的触发顺序是:
`componentWillMount` ---  **render** --- `componentDidMount`

注二:组件生命周期第二步: `Updating`
第二阶段是渲染完成后, 如果这时候其中的state发生变动, `componentWillReceiveProps`调用, 
`shouldComponentUpdate`会用当前最新的状态与原来的组件状态值比较, 如果确实有变动才会触发视图的Update
接下来的钩子函数的触发顺序就是：
`cpmponentWillUpdate` --- **render**  --- `componentDidUpdate`
组件生命周期第三步 `Unmounting`
钩子函数`componentWillUnmount`会在组件元素被移除的时候触发

下面的图很好的描述了react组件的生命周期:
![react组件生命周期](http://7xny7k.com1.z0.glb.clouddn.com/reactPeriod.jpg)




拿起这个dome基本就可以干些小事了。没看懂没关系，去看看这个教程吧。不知道你看到这篇blog时作者还有没有继续更新。
[慕课网react入门](http://www.imooc.com/learn/504)


不得不说虚拟dom的理念实在惊艳，在jsx下，html，css和js都混合在了一起，虽然不适合传统的遵循样式结构代码分离的web网页的开发。

但确实是一个美妙的尝试，对我这种自诩文艺的前端程序员也是个美妙的编码体验。
