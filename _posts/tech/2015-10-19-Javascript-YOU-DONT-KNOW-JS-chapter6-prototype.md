---
layout: post
title: javascript 你不知道的原型继承
category: 技术
tags: Javascript
description: 关于js原型继承原理详解
---

### 来自面向类的语言的误区

Javascript的原型继承通常认为是动态语言版本的类继承。
人们对于“继承”的心理预期并不能掩盖javascript中原型继承和类继承几乎完全相反的行为表现。
类似其他面向类语言的术语，“类”，“构造函数”，“实例”，“多态”等，很多javascript书籍，或是大部分的javascript书籍都会把这些术语概念用javascript的特性来演示一遍。
通常来说，因为面向类语言的在语言发展过程中的绝对优势地位，那一套面向对象的设计模式深入人心并几近成为语言范式。
当这些术语变得无处不在，拓展自身含义甚至潜入js中潜移默化时，很容易将初学者引入某一种怪圈。
所以在接受类语言优秀的面向对象思维的之余，我们更要认清类继承与原型“继承”的区别。

### 狭义，或是说，真正的继承

继承意味着复制，新建一个副本。


### 原型“继承” 实质

js中的继承并不是新建一个副本，js会在两个对象之间创建一个关联。
关联的目的是让一个对象能访问另一个对象的属性和方法。
这种关联的方式 可以认为是一种 委托机制。


### 模仿出的“类”

下面是一代代javascript开发者绞尽脑汁想出的继承模式范式。

	function Class(name){
		this.name = name ;
	}
	Class.prototype.myName = function(){
		return this.name ;
	} ;

	var a = new Class('Ahkari') ;
	var b = new Class('hehe') ;

	a.sayName() ; //'Ahkari'
	b.sayName() ; //'hehe'

这段代码实例了两个Class，并且让实例确实能执行`sayName()`访问实例属性。
按照静态类理论，创建出来的a和b实例含有`Class.prototype`的副本，应当是与原对象完全无关系的复制过后的新属性集合。

然而我们知道这里一点复制操作也没，a和b的sayName方法实际上并不存在，我们调用sayName的时候因为无法在实例内部找到，通过对象`[[prototype]]`链关联在其上层才找到。
也就是其构造函数的`prototype`，`Class.prototype`上。
这个`[[prototype]]`在现代浏览器中面向开发者开放，名为`__proto__`，这例子里 `a.__proto__ === Class.prototype`成立。
还会涉及到一个十分不靠谱的属性，`constructor构造器`。就是所谓的 “实例”属性。
这个属性声称指向这个实例的构造器，当然，在这里a.constructor也确实完美的指向Class.
但这是无奈的假象。这个属性并没有经由Class缔造，他同样是从__proto__查找得到。
也就是说，在我们申明
	
	function Class(name){
		this.name = name ;
	}

的时候，`Class.prototype.constructor === Class`就一直是成立了，并且随着new的过程而扎根于实例的__proto__之中。如果我们有意而为，每个实例的constructor能指向任何东西，这不是一个可靠的属性，更像是玩票性质的语法欺骗，一如强行模仿类继承的原型继承。


### 你不知道的JS继承模式

基于js的原型继承我们可以设计出全新的“继承”设计模式。我们命名为 **对象关联**
我们通过代码来比较两种设计模式：

**面向对象**

	function Foo(who){
		this.me = who ;
	}
	Foo.prototype.indentify = function(){
		return "I am " + this.me ;
	} ;

	function Bar(who){
		Foo.call( this, who ) ; //继承内置属性
	}
	Bar.prototype = Object.create( Foo.prototype ) ; //通过Object.create继承"父类"Foo的原型
	Bar.prototype.speak = function(){
		alert( "Hello, " + this.identify() + "." ) ; //继承之后可以新增一些属性方法
	} ;

	var b1 = new Bar("b1") ;
	var b2 = new Bar("b2") ;

	b1.speak() ;
	b2.speak() ;

这段代码应该无需赘述，可能需要注意下Object.create()方法。这个方法接受一个对象参数，将这个对象作为新对象的原型并返回新对象。
可以理解为原型继承最效率的方法。

**对象关联**

	Foo = {
		init : function(who){
			this.me = who ;
		},
		identify : function(){
			return "I am " + this.me ;
		}
	} ;
	Bar = Object.create( Foo ) ; //Bar“子类”继承Foo“父类”，包括构造方法init()

	Bar.speak = function(){
		alert( "Hello, " + this.identify() + "." ) ; //“子类”新增方法。
	} ;

	var b1 = Object.create( Bar ) ; //申明继承Bar对象的b1对象
	b1.init("b1") ; //实例化b1
	var b2 = Object.create( Bar ) ; //申明继承Bar对象的b2对象
	b2.init("b2") ; //实例化b2

	b1.speak() ; //调用
	b2.speak() ;

实现了一样的功能，关联了三个对象。但是简洁了不少。
如果要考虑两者思维模型的区别，可以说，**对象关联**只关心**对象之间的关联关系**。
我们看到，在面向对象仿造的“类”模型中，自有属性放在构造函数代码块中等待new调用来初始化，继承属性则通过prototype来花式关联。
两种不同的处理方式，还有原型，new和构造函数调用这些“奇异理论”形成了复杂的关系网，让代码并不容易理解。

而在**对象关联**中，没有刻意设计或是模仿出一个类。
对象之间通过原型自然而然的关联在一起，实例也是关联的方式来申明。
而其他需要初始化的自有属性都用统一的init()方法调用。在此时完成实例化。
也就是说，声明对象通过Object.create(), 实例对象则通过原型里自定义的init()方法。
执行一步是就是“类”的继承，执行两步就是“类”的实例化。

**行为委托**或是说**对象关联**是一种和各个js教学书上都截然不同的继承设计模式，少见而强大。

**“通过认为对象之间是兄弟关系，互相委托。”**对象关联必然在设计思想上比基于“父与子”关系的类模式更贴近javsceipt原型理念的本质。
