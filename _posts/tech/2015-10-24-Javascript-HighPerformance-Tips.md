---
layout: post
title: javascript 高性能优化建议（一）
category: 技术
tags: Javascript 高性能 建议
description: 高性能javascript读书笔记，将不知道的知识点记下来
---

### 加载与执行

1. **内嵌js不要紧跟在`<link>`外链css之后**
    * 因为内嵌js执行时要获得最精准的样式信息，所以会阻塞页面去等待`<link>`下载完毕

2. `<script>`的defer标签已被现代浏览器支持。**并行下载，并在页面加载完，load前触发**
  * 如下代码，三个标签位置随便互换，最后执行结果都是一样的
 
    >	    <head>
    >			<title>含有Defer属性的js标签执行情况</title>
    >			<meta charset="UTF-8"></meta>
    >		</head>
    >		<body>
    >	    <script defer src="./defer.js"></script> //console.log('defer<script>执行')
    >	    <script src="./script.js"></script> //console.log('普通<script>执行')
    >	    <script src="./load.js"></script> //window.onload = function(){ console.log('页面onload事件触发') ; }
    >	    </body>
    
    >		普通<script>执行
    >		defer<script>执行
    >	    页面onload事件执行
  
  * 低版本的IE支持含defer属性的标签里写内联js，而现代浏览器遵循HTML5规范，defer只对含src属性的`<script>`起作用，即必须为外链js

3. **动态创建`<script>`时添加到head里比较保险**，避免body中元素可能正在被操作。


### 数据存取

1. js中有四种基本数据存取位置（方式），**访问后两种代价会高一点**
  * 字面量（字符串，数字，布尔，对象，数组，函数，正则，null与undefined）
  * 本地变量var
  * 数组元素
  * 对象成员  

### DOM编程

1. 区分浏览器的渲染引擎和js引擎，dom和javascript是独立实现的，他们之间**通信的消耗是dom操作消耗性能的根本原因**。
| 浏览器        | 渲染引擎   |  js引擎  |
| :--------:   | :-----:  | :----:  |
| Safari     | WebKit / WebCore |   JavascriptCore / SquirrelFish     |
| Chrome        |   WebKit / WebCore   |   V8   |
| Firefox        |    Gecko    |  SpiderMonkey / TraceMonkey / JagerMonkey  |
| IE   |   Trident  |   JScript   |

2. HTML集合具有实时性，时刻链接着底层文档。每次使用结果对象的时候，哪怕只是访问length都会**重复最初的查询的过程**。
    `document.getElementsByName`
    `docuemnt.getElementsByClassName`
    `document.getElementsByTagName`
    `document.images`
    `document.links`
    `document.forms`
    `docuemnt.forms[0].elements`
以上这些dom访问操作的结果都会实时反应文档中元素的状态，没有意识到集合的实时性的话会导致一些不易察觉的逻辑错误。
```javascript
//一个意外的死循环
var divs = document.getElementsByTagName('div') ;
for ( var i = 0 ; i < divs.length ; i++){ //每次访问divs,其length反应了实时状态
    document.body.appendChild( document.createElement('div') ) ;
}
```
需要注意的是，html集合是个类数组而并不是数组，虽然可以通过下标访问其中的元素，但是一些基本的Array类应该有的原型方法却并不能使用。
例如用`indexOf`判断某个元素是否存在于某个html集合中，有两种方法，
一种是借用Aarry原型：
`Array.prototype.indexOf.call( htmlCollection , htmlNode ) ;`
一种是将html集合转为普通数组：
```javascript
function toArray(htmlCollection){
    for (var ret=[],i=0;i<htmlCollection.length;i++){
        ret[i] = htmlCollection[i] ;
    }
    return ret ;
}
toArray( document.getElementsByClassName('test') ).indexOf( document.getElementsByClassName('test')[0] ) ; // 0 
toArray( document.getElementsByClassName('test') ).indexOf( document.getElementsByClassName('test')[1] ) ; // 1
toArray( document.getElementsByClassName('test') ).indexOf( document.getElementsByClassName('test')[2] ) ; // 2 
```

基于html集合实时性，我们可以用多种方法提升访问时的性能，例如需要重复读取属性时，将不变的属性用局部变量存储；同理，转换为数组时对其操作和读取的效率也会明显提高。
3. 选用现代浏览器优化过的元素节点API。
现代浏览器API遍历时只返回元素节点，而**旧的API会算上普通文本甚至是注释与空白**，因此运行缓慢
|功能|现代属性名API|现代API兼容性|被替代的属性API|
|:---:|:---:|:---:|:---:|
|获取当前节点的所有子节点|children|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 6+**|childNodes|
|获取当前节点的子节点个数|childElementCount|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 9+**|ChildNodes.length|
|获取当前节点里第一个子节点|firstElementChild|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 9+**|firstChild|
|获取当前节点里最后一个子节点|lastElementChild|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 9+**|lastChild|
|获取当前节点后一个节点|nextElementSibling|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 9+**|nextSibling|
|获取当前节点前一个节点|previousElementSibling|Firefox3.5+/Safari 4+/Chrome/Opera9.62+/**IE 9+**|previousSibling|

4. 使用现代浏览器的**选择器API**，基于css选择器语法，适合选择大量且条件相对复杂的元素。
`var element = document.querySelectorAll('#lily .Ahkari') ;`
`var element = document.querySelector('#lily .Ahkari') ;//获取匹配的第一个 `

5. 浏览器“重排”就是重构渲染树，重计算元素盒模型，更改几何属性。
浏览器“重绘”是在渲染树基础上重绘制元素，可以理解为绘制颜色这些样式属性。
    * 重排触发的情况：dom元素的添加和删除 | 位置改变 | 盒子尺寸改变 | 文字图片等内容改变 | 渲染器初始化 | 窗口resize
    * 浏览器会将一系列会触发重排的操作列队，之后一并执行以优化性能。但是如果在队列中途使用[[get]]操作访问元素布局属性，如`offsetTop`,`offsetLeft`,`offsetWidth`,`offsetHeight`,`scrollTop`|||,`clientTop`,|||,`getComputedStyle() (currentStyle in IE)`就会为了数据的准确性而强制执行当前队列，强制触发重排而打乱浏览器的重排优化。
所以合理控制元素属性的读与写能基于重排优化大大提高执行性能。下面介绍些重排优化方法，有些甚至很奇怪。
        1 合并样式修改，这个比较好理解
        ```javascript
        //初始代码
        var el = document.getElementById('mydiv') ;
        el.style.borderLeft = '1px' ;
        el.style.borderRight = '2px' ;
        el.style.padding = '5px' ;
        //优化为
        var el = document.getElementById('mydiv') ;
        el.style.cssText = 'border-left:1px;border-right:2px;padding:5px;' ;
        //或使用下述的css类名控制
        var el = document.getElementById('mydiv') ;
        el.className = 'active' ;
        ```
        
        2 强制脱离文档流，这个脱离的概念有点有趣，就是将元素diaplay:none;掉，执行完dom变化的操作后再disolay:block;回来。
        3 创建文档之外的文档片段，在其中操作dom。这个轻量级的document对象同样很有趣，我是第一次见这个API。**【这个是所被推荐的方法，因为只触发一次重排】**
        ```javascript
        var fragment = document.createDocumentFragment() ;
        //这里对fragment进行dom操作,fragment本身在文档之外
        document.getElementById('myList').appendChild( fragment ) ; //操作完之后将片段放回文档之中
        ```
        4 克隆节点，操作完成后替换旧节点。**经验证，此法并不能克隆原元素所绑定的事件，极不推荐【在IE旧浏览器上有能克隆事件的bug】。**
        ```javascript
        var old = document.getElementById('myList') ;
        var clone = old.cloneNode(true) ; //true,内部子元素也克隆
        //这里对clone进行操作
        old.parentNode.replaceChild(clone,old) ;
        ```