---
layout: post
title: javascript 高性能优化建议（三）
category: 技术
tags: Javascript 高性能 建议
description: 高性能javascript读书笔记，将不知道的知识点记下来
---

### Ajax
1. 数据传输优化
    五种常用技术用于向服务器请求数据
    1. `XMLHttpRequest(XHR)` --- 最常用:就是ajax
    2. `Dynamic script tag insrtion` 动态脚本注入 --- 常用:克服了XHR的不能跨域的问题, 使用有限制
    3. `iframes` --- 作为数据传输技术时不常用
    4. `Comet` --- 作为数据传输技术时不常用
    5. `Multipart XHR` --- 较新,思路是后端把N个图片或文件的数据转换为base64字符串返回,前端再二次解析(比方图片瀑布流,大量的图可以通过这种自定义协议的方式传输,在readyState为3时开始触发返回值轮询,每次responseText发现新的分隔符就把上个资源所有数据作为一个完整资源进行处理)

2. 发送数据优化
    1. `XHR`
    2. `Beacons`图片信标技术, 把请求写在img src里面的黑科技
        
            var url = '/status_tracker.php' ;
            var params = [
                'step=2',
                'time=20151129'
            ] ;
            var beacon = new Image() ;
            beacon.src = url + '?' + params.join('&') ;
            beacon.onload = function(){
                if (this.width === 1){
                    //成功    
                }else if (this.width === 2){
                    //失败, 请重新并创建另一个信标
                } ;
            }
            beacon.onerror = function(){
                //不需要响应反馈的时候, 发送一个不带消息正文的204 No Content状态码
                //会在这里接收到, 客户端会尝试重试并创建另一个信标
            }

3. 传输数据格式探讨 
    `XML`出现的最早, 不过js程序员解析起来非常费劲
    `XPATH`好点, 不过并未得到广泛支持.
    `JSON`是前端和后端交互解析的首选, 解析方法有JSON.parse()和eval()两种, 推荐第一种, 需要认识到的是, 数据交换格式并不是死的, 在现有的技术方案下只要能实现前后端通信, 就有千千万万的方法来制定交互规则, 即使是简洁的json也能更简洁.
    `JSON-P`意为`JSON with padding`, 其实就是"动态脚本注入", 和ajax技术上并没有关系, 他的数据被当做原生的js, 所以解析速度非常块, 不足之处就是需要前后端联调.
    `HTML`是在当性能瓶颈不在带宽的时候推荐的, 服务端一次性生成所需的结构, 然后直接显示在页面上
    `自定义格式`则是自己制定规则, 能多简略就多简略

4. Ajax性能优化
    1. 缓存数据有两种方法
        * 服务端设置HTTP头信息
            用`get`方法发出请求, 用`Expires`头信息告诉浏览器缓存响应多久, 这个值是一个过期日期.
            `Expires: Mon, 29 Nov 2015 23:30:00 GMT`
        * 客户端本地数据存储
            编写一个ajax请求的实现, 当URL相同的时候就取既有的缓存. 管理也比较方便.
            核心代码其实和所有的js缓存机制一个样:

                if ( localCache[ url ] ){
                    callback.success( localCache[ url ] ) ;
                    return ;
                } 
            
            `localCache`会在每次`readyState`为4, 且不为404的时候将相应文本存储起来
                
                localCache[ url ] = req.responseText ;
                callback.success( req.responseText ) ;
                


### 编程实践
1. 避免双重求值
    所谓双重求值, 就是javascript里面有四种标准方法可以接受字符串作为要执行的语句。
这些双重求值的方法每次调用的时候都会创建一个新的**解释器/编译器**实例， 必然使得代码执行速度变慢。
        `var num = 5, num2 = 6 , result , sum ;`
    *   `result = eval('num1 + num2') ;` //eval()执行代码字符串
    *   `sum = new Function('arg1','arg2','return arg1+arg2') ;` //Functon()定义两参函数
    *   `setTimeout('sum = num1 + num2',100) ;` //setTimeout()执行代码字符串
    *   `setInterval('sum = num1 + num2',100) ;` //setInterval()执行代码字符串
    
    建议就是避免使用eval()和Function(), 而两个定时器用函数方式使用。

2. 使用Object/Array直接量
    这个。。又方便又效率，为啥不用？

3. 函数延迟加载（黑科技）
    第一种方法是重定义函数：
    第一次见这种写法，我受到了惊吓。
    我们设想一个用来绑定事件的公共方法，每次都得检测浏览器类型，才能确定用什么对应的方法来绑定事件，“我们就不能一次检测确定了版本，之后都是直接使用该版本的绑定方法吗？”
    
        function addHandler(target,eventType,handler){
            if (target.addEventListener){
                //复写现有函数
                addHander = function(target,eventType,handler){
                    target.addEventListener(eventType,handler,false)
                };
            }else{ //IE
                addHander = function(target,eventType,handler){
                    target.attachEvent('on'+eventType,handler);
                }
            }
            //调用新函数 
            addHandler(target,eventType,handler) ;
        }

    在函数第一次调用的时候, 根据当前浏览器类型就直接把原函数给覆盖掉了, 在函数里面修改函数自身的值, 确实是我一直以来没想到的事。
    或采用第二种方法，条件预加载
        
        var addHandler = document.body.addEventListener ? 
                function(target,eventType,handler){
                    target.addEventListener(eventType,hander,false) ;
                }:
                function(target,eventType,handler){
                    target.attachEvent('on'+eventType,handler) ;
                } ;

4. 位操作
    这个确实用的很少, 我们只举一例:
    
        for(var i=0,len=rows.length;i<lenl;i++){
            if (i & 1){ //原来是模除的方法 i%2 , 改用位运算, 平均提速50%
                className = 'odd' ;
            }else{
                claseName = 'even' ;
            }
        }

5. 原生方法
    你的代码永远没有原生方法块, 因为原生方法用低级语言写进了浏览器里面
    同理, 号称最快的jquery的css查询引擎也没有原生的`querySelector()`和`querySelectorAll()`的十分之一快。


### 构建并部署高性能JavaScript应用

### 工具

总结: 最后两章偏工程化一点, 并不能算是什么知识点.

