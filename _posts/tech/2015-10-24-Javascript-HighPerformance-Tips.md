---
layout: post
title: javascript 高性能优化建议（一）
category: 技术
tags: Javascript 高性能 建议
description: 高性能javascript读书笔记，将不知道的知识点记下来
---

### 加载与执行

1. 内嵌js不要紧跟在<link>外链css之后
  * 因为内嵌js执行时要获得最精准的样式信息，所以会阻塞页面去等待<link>下载完毕

2. `<script>`的defer标签已被现代浏览器支持。并行下载，并在页面加载完，load前触发
  * 如下代码，三个标签位置随便互换，最后执行结果都是一样的

	`<!DOCTYPE html>`
	`<html>`
	`<head>`
	`	<title>含有Defer属性的js标签执行</title>`
	`	<meta charset="UTF-8"></meta>`
	`</head>`
	`<body>`
	`<script defer src="./defer.js"></script> //console.log('defer<script>执行')`
	`<script src="./script.js"></script> //console.log('普通<script>执行')`
	`<script src="./load.js"></script> //window.onload = function(){ console.log('页面onload事件触发') ; }`
	`</body>`
	`</html>`

	`普通<script>执行`
	`defer<script>执行`
	`页面onload事件执行`

  * 低版本的IE支持含defer属性的标签里写内联js，而现代浏览器遵循HTML5规范，defer只对含src属性的`<script>`起作用，即必须为外链js

3. 动态创建`<script>`时添加到head里比较保险，避免body中元素可能正在被操作。


### 数据存取

1. js中有四种基本数据存取位置（方式），访问后两种代价会高一点
  * 字面量（字符串，数字，布尔，对象，数组，函数，正则，null与undefined）
  * 本地变量var
  * 数组元素
  * 对象成员  
