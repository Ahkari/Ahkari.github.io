---
layout: post
title: AngularJS简述
category: 技术
tags: AngularJS 核心概念介绍 参考
description: 参考介绍, 三分钟回想起angular大部分理念
---

### 简介 
AngularJS是为了克服HTML在构建应用上的不足而设计的前端框架，但并不是所有的web应用都适合使用angularJS来做。

从官方给的介绍来看，AngularJS试图成为一种端对端的解决方案，让他很适合构建一个`CRUD`(增加`Create`，查询`Retrieve`，更新`Update`，删除`Delete`)的应用。
他的一些特性也是针对了`CRUD`应用特点的，比如：数据绑定，基本模板标识符，表单验证，路由，深度链接，组件重用，依赖注入等。

不幸的，AngularJS因为为开发者呈现了一个更高层次的抽象来简化应用开发。所以如同所有的抽象技术，这都会损失一部分灵活性。

即是，一些DOM操作很频繁很复杂的应用，并不适合完全抽象出更高的层次，这些应用反而只能用一些更轻量，简单的技术比如jquery来做会更好。这种应用一般如游戏，图形界面编辑器等。他们和CRUD应用都有很大不同。

### AngularJS应用引导过程
1. 注入器injector将用于创建此应用的依赖注入
2. 注入器将会创建根作用域作为我们应用模型的范围
3. AngularJS将会链接根作用域中的DOM，从用ngApp标记的HTML标签开始，逐步处理DOM中指令和绑定

### 理想的项目结构
如下
![理想的angular项目结构](http://7xny7k.com1.z0.glb.clouddn.com/angulariProj.png)

### 核心概念介绍：双向绑定
如果你理解不了这个双向到底是怎么个双向法，不妨看看下面这个实例：
![input经典的双向绑定](http://7xny7k.com1.z0.glb.clouddn.com/angularDoubleBinding.png)
你在这个input框里输入东西的时候，其中value的任何改变会立即反映到定义的模型变量`yourname`上，此为一个方向，即ng-model将视图的改变反馈到模型。
同时，我们看到`{{}}`中，又显示了`yourname`的值，所以模型的变化会立即显示到文本上，此为另一个方向，即{{}}将模型内容实时反馈。

### 核心概念介绍：路由
起因是ajax改变了发送请求就要重新刷新页面的前后端开发模式。我们构建单页应用的时候就可以用前端来完全接管history和路由。
![简单路由](http://7xny7k.com1.z0.glb.clouddn.com/angularRoute.png)
简单路由如上。
复杂点的需要引入angular UI-Router
![复杂路由html](http://7xny7k.com1.z0.glb.clouddn.com/angularUIROUte.png)
![复杂路由js](http://7xny7k.com1.z0.glb.clouddn.com/angularRouteUI.png)
通过上图所示的模板嵌套思路，就可以构建稍微复杂的应用。
前端路由的基本原理就是如下几点：
`哈希#`,就是锚点不会导致浏览器刷新
`HTML5 新的history API`
而相应的使用路由我们也要遵循如下几个原则
1. 路由的核心是给应用定义“状态”
2. 使用路由机制会影响到应用的整体编码方式，因为你需要预先定义好状态
3. 考虑兼容性问题与“优雅降级”

### 核心概念介绍：指令
指令是angular里最复杂的内容，其实可以理解为是`组件`。我们下面按照知识点梳理。

1. 匹配`restrict`指令的四种方式（推荐A）
![指令匹配的四种方式](http://7xny7k.com1.z0.glb.clouddn.com/angularDirectiveFuction.png)
![定义指令的代码](http://7xny7k.com1.z0.glb.clouddn.com/ngRestrict.png)
模板可以直接返回html也可以用run方法缓存。

2. 指令的三个生命阶段
加载阶段：加载angularJS,找到ng-app,确定应用边界
编译阶段：遍历dom找到所有指令，找到指令所有的replace,compile,template等，重新转换dom结构
链接阶段：执行指令中的link函数，在link中操作dom，绑定事件监听器等。

3. 指令和控制器之间的交互
![link绑定事件](http://7xny7k.com1.z0.glb.clouddn.com/nglink.png)
这个介绍了如何和控制器交互，同时也展示了link操作dom的方法。 

4. 指令之间交互
![指令间交互](http://7xny7k.com1.z0.glb.clouddn.com/ngdirectivecommiute.png)
是通过内部定义的controller来暴露给其他指令用的，其他指令定义时require指定指令后，目标指令就会追加进link的arguments中

5. 指令的scope
一个指令创建的时候都要选择是继承自己的父作用域还是创建一个新的自己的作用域。
`scope:false`：可以直接使用父作用域中的变量，函数。
`scope:true`：创建一个新的作用域，继承自父作用域。也就是说，只在创建的时候和父作用域一样，往后的修改都是自身的不会改变控制器作用域里的值。
只需定义处加一个`scope:{}`
最后一个是`scope:{}`

6. 指令scope:{}的绑定策略
这会为我们创建一个新的与父作用域隔离的新的作用域。但是不代表我们不可以使用父作用域的属性和方法。
[angular指令scope详解](http://segmentfault.com/a/1190000002773689)
`@`，把当前属性作为字符串传递，你还可以绑定来自外层scope的值，在属性之中插入{{}}即可 
`=`，与父scope中的属性进行双向绑定
`&`，传递一个来自父scopt的函数，稍后调用
使用如下，当指令编译到模板的name时就会在scope中寻找是否有name的键值对，如果存在就按规则应用。
        
        scope: {
            // `myName` 就是原来元素中的`my-name`属性
            name: '@myName', 
            age: '=',
            // `changeMyAge`就是原来元素中的`change-my-age`属性
            changeAge: '&changeMyAge' 
        }

7. 常用内置指令介绍
    
    1. form
    可嵌套，自动校验，防止重复提交，type扩展，内置样式等。
    2. ngBind
    类似{{}}，用来双向绑定
    3. npApp
    应用入口
    4. ngInclude
    模板缓存

8. ERP系统常备angularUI指令组件介绍
    
    1. Form
    2. DatePicker
    3. FileUpload
    4. Tree
    5. DataGrid

9. 互联网/电商常用angularUI指令组件
    参考kissyGally（其实看不懂）

10. 指令思想来源
构建声明式UI，来源自adobe的flux应用。

### 核心概念介绍：Service
Service服务是一种**全局方法**，`Service`,`Provider`,`Factoy`本质都是Provider，只是定义方式有区别。本质是一种设计模式。
所以Service特性如下：
    1. Service都是单例，全局唯一
    2. Service都是用`$injector`实例，只需声明即可
    3. Service存在整个应用生命周期，可共享数据
    4. 在需要使用的地方利用**依赖注入**机制注入service
    5. 自定义的Service需要写在内置的Service后面
    6. 内置的Service用`$`符号开头，自定义的避免这样。
内置如:
`$http`，即ajax的angular封装方式。
![`$http`](http://7xny7k.com1.z0.glb.clouddn.com/ngserviceHttp.png)
又如`$watch`，用来监控数据变化。
![`$watch`](http://7xny7k.com1.z0.glb.clouddn.com/ngServiceWatch.png)
还有`$filter`，用来数据格式化。           
等等。

### 一个应用的开发流程
1. 界面原型设计
    页面切分, 理解交互原型.`active2P`
2. 切分功能模块并建立目录结构
    细分功能, 建立angular的工程目录结构
3. 使用angular-ui和Bootstrap编写UI
    编写UI页面, 完成结构
4. 编写Controller
5. 编写Service
6. 编写Filter
7. 单元测试和集成测试

### 小结
到这里其实本文已经戛然而止了。

在我刚接触前端的时候曾经觉得框架之上, 新技术才是潮流. 那时候对angular, react, vue这些被大家所推崇的框架十分的向往, 觉得只有框架才能解决我的问题。

然而在即将正式写前端一年之际, 当自己基础踏实了，写的业务场景多了。我却越发觉得前端不能盲目追求技术，并陷入了迷信框架的陷阱之中。

angular为什么被人推崇, 是因为他起到了前端工程化的推动作用, 他的设计师从Flash那一套声明式的组件构建理论中获得启发并将这套理论带到了前端开发界。
但是这是不是我们所有场景都适合呢？

并不，angular高度适合那种业务逻辑和前后端请求方式有规律的网页应用。比如管理系统，这种系统的view，modal，controller，service，route都是有据可循的。甚至组件都可以完全抽象出来。用angularJS自然再合适不过。

然而当系统复杂程度更高一层甚至几层，angular其笨重的特点让其难以处理极端不规则的UI与页面逻辑。

在这个高度上，只能删繁就简，使用组件高度的库来构建应用了。
比如react和jqueryUI。

另一个让我对angular有所顾忌的原因是因为他的理念过于繁琐，虽然是一些很不错的理念但是如果你的经验不足以看到本质的话，初学者很容易被带进概念的误区。

angular的所有技术都可以在一些应用场景中找到影子，比如路由可以用pjax，模板可以用模板引擎，指令是另一种UI组件封装等，当你经验足够你就可以构建自己的前端框架前端体系了。即使不需要写框架，你对这些框架也会得心应手的。

还是那句话，框架也是为了解决实际应用场景中的问题的。


**开发者，需要构建自己的理论体系与基础，不能盲信他人的理念。**
