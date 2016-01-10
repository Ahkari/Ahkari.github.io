---
layout: post
title: React简述
category: 技术
tags: React demo 参考
description: 参考demo, 三分钟就会写简单react了
---

### 目的

为了让自己任何时候看见这篇blog时, 都能在五分钟内拿起React干活。

### 个人理解

React并不是一个MV*框架，应该拿她和JqueryUI来比较，他们都是专注于组件的构建。
从组件层面来构造web应用。

但是它比起jqueryUI有一些新奇的特性：
1. **JSX语法糖**。
React通过增加语法糖来方便带着尖括号类html字符的代码的编写。
    JSX转换库：https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js
    React在线CDN：https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/react.js
2. **虚拟dom**。
从特性一带来的，在JSX中的尖括号html结构都是react的虚拟dom，对虚拟dom操作需要遵循一系列规则，同时虚拟dom也带来了普通dom所不具有的优势。
虚拟dom通过react的调度，能最小化真实dom的操作，从而在性能上得到提升。
3. **状态更新**。、
组件所应用的归属于react组件对象state如果发生了变更，会自动触发组件的update。
这个就类似于MV*框架的双向绑定了。jqueryUI没有自动更新功能，内部数据变更只能手动指定渲染。

我认为，react的最大优势就在于其维护的虚拟dom，这是一个新的理念。
一如当年jquery将dom操作玩的天花乱坠，靠的就是自己的一套jquery对象的理念。
如今react虽然入门阶段没有jquery便捷，但随着虚拟dom优势的逐渐发掘，有理由相信react会蓬勃发展。

### 一个简单的demo带你了解react
我们构建简单的环境，用type="text/jsx"来表明我们的这个是能用jsx语法的代码文件.
![react基础环境](http://7xny7k.com1.z0.glb.clouddn.com/reactEnvironment.png)

我们想实现两个自定义组件: 
就是下面这玩意:
![react入门demo](http://7xny7k.com1.z0.glb.clouddn.com/demoReact.png)
他们的功能是:
1. 显示和隐藏按钮:
    * 自定义类名
    * 自定义样式
    * 绑定点击事件
    * 绑定了mount事件,在render前后会有提醒
2. 自定义输入框:
    * 显示在render调用时的prop属性的值
    * 绑定onchange事件, 改变input的值, 会自动被react更新
    * 绑定update事件,在update前后也会有提醒
    
代码如下:

    //自定义组件<ClickButton>
    var ClickButton = React.createClass({ 
      render : function(){
        var styleObj = {
          backgroundColor : '#333' , //样式对象, 样式全部用驼峰命名, 和操作原生dom.style一样
          color: '#dddddd'
        }
        //子组件的ref属性是用来方便选择与定位
        //样式需要指定style对象, 或自己写一个{}对象
        //类名控制需要用className属性来控制
        return ( 
          <div>
            <button onClick={this.showOrHide} style={styleObj} className="likeThis">显示 | 隐藏</button><span ref="tip">测试点击能不能显示和隐藏</span> 
          </div>
        ) ; //onClick等驼峰式绑定方法是react定义事件的方式, 编写方式类似原生绑定, 但因为这里是虚拟dom, 所以自由度很高且没有一系列问题
      },
      showOrHide: function(event){
        var tipE = React.findDOMNode( this.refs.tip ) ; // this.refs[ 子组件的ref属性的名字 ], 这个获取到虚拟的dom, 然后用findDOMNode来获取到真实dom
        if ( tipE.style.display === 'none'){
          tipE.style.display = 'inline' ;
        }else{
          tipE.style.display = 'none' ;
        }
        event.stopPropagation() ; //这个event是react封装后的event方法, 比原生的强
        event.preventDefault() ;
      },
      //注一: 生命周期一Mount
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
        ) ; //{}这个符号表示其内部是表达式
      },
      changeTip : function(event){ //onChange时触发
        this.setState({
          inputValue : event.target.value 
        }) ;
        event.stopPropagation() ;
        event.preventDefault() ;
      },
      //注二: 生命周期二Update
      componentWillUpdate: function(){
        console.log( 'will update' ) ;
      },
      componentDidUpdate: function(){
        console.log( 'did update' ) ;
      }
    }) ;
    //render们接受的虚拟dom元素都必须是单个元素, 不能是多个元素, 所以这里要渲染两个元素必须得用个div包裹起来
    //其上的属性可以在render时用this.prop获取到
    React.render( ( 
      <div>
        <ClickButton />
        <InputArea defaultValue="render调用时定义" />
      </div>
    ) , document.getElementById('container') )
    

注一:组件生命周期第一步: Mount
第一阶段是render前后, hock函数的触发顺序是:
componentWillMount ---  **render** --- componentDidMount
注二:组件生命周期第二步: Updating
第二阶段是渲染完成后, 如果这时候其中的state发生变动, componentWillReceiveProps调用, 
shouldComponentUpdate会用当前最新的状态与原来的组件状态值比较, 如果确实有变动才会触发视图的Update
接下来的钩子函数的触发顺序就是：
cpmponentWillUpdate --- **render**  --- componentDidUpdate
组件生命周期第三步 Unmounting
钩子函数componentWillUnmount会在组件元素被移除的时候触发


拿起这个dome基本就可以干些小事了。没看懂没关系，去看看这个教程吧。
[慕课网react入门](http://www.imooc.com/learn/504)

短短一个小时内, 我确实对react着了迷, 虚拟dom这个概念太美了, 
因为是虚拟的, 我们可以自定义一套规则, 即有html的结构又有javascript组件化代码的编写风格, 
写起来真的是不一样的感觉, 以后一定要在项目里偷偷用起来~~


