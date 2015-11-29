---
layout: post
title: javascript 高性能优化建议（二）
category: 技术
tags: Javascript 高性能 建议
description: 高性能javascript读书笔记，将不知道的知识点记下来
---

### 算法和流程控制

1. 循环语句性能提升
    1.  javascript提供的四种循环类型中（`for`，`while`，`do...while`，`for-in`）, 只有for-in循环明显要慢点。

            for (var i=0;i<items.length;i++){
                operation(items[i]) ;
            }

        上述循环代码每次循环体都会有items长度的查找操作,所以如下就可以提升性能.
    
            for (var i=0,len=items.length;i<len;i++){
                operations(items[i]) ;
            }

    2.  `Duff's Device`达夫设备。
        这是种循环体展开技术，最终目的是缩减循环次数，让修改后的代码每次迭代能执行原来的多次迭代。
        我们直接看这个比较成熟的一段代码吧：
        
            var i = items.length%8 ;
            while(i){
                operation( items[i--] ) ;
            }
            i = Math.floor( items.length/8 ) ;
            while(i){
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
                operation( items[i--] ) ;
            }

        比如我们要执行总共items.length长度的循环。假设有20次，我们打算用达夫设备缩减迭代次数，比如缩减迭代为原来的8分之一。
        那么我们先执行余数次，也就是20%8=4次的操作，然后执行两个大循环，每个里面包含了8次的操作。
        可见，达夫设备解决的问题是循环体大量迭代时的性能开销。但其本身代码要比一般的循环语句复杂不少。

2.  条件语句优化
    1. `if-else`多条件判断时从普通式变换为嵌套式
    
            if ( value === 0 ){
            }else if ( value === 1 ){
            }else if ( value === 2 ){
            }else if ( value === 3 ){
            }else if ( value === 4 ){
            }else if ( value === 5 ){
            }else if ( value === 6 ){
            }else if ( value === 7 ){
            }
        
        这种代码换做其他人估计直接换`swicth`了, 确实`switch`是个好办法, 如果要坚持使用`if-else`, 这个在每次找到匹配值之前所要经过的判断语句就显得非常多, 假如我的value是7, 那么在匹配到7的时候就要判断8次。我们用类似二分的思想就可以优化这个代码。
        
            if ( value <4 ){
                if (value<2){
                    if (value ===1){
                    }else if(value===0){
                    }
                }else{
                    if (value === 2){
                    }else if(value===3){
                    }
                }
            }else{
                if (value<6){
                    if (value === 4){
                    }else if(value === 5){
                    }
                }else{
                    if (value===6){
                    }else if(value ===7){
                    }
                }
            }
        
        可见，在抵达最终的匹配条件时，无论最终匹配到哪个也只有三次判断。可以说是以牺牲代码块的简洁度来提升性能。
        
    2. 转换为哈希表查找
    
        这个在键值之间存在逻辑映射时，能大大体现其性能优势。通过映射关系完全摒弃条件语句。
        如之前的代码，如果每个条件依次对应结果数组中value位置的值。
        那么就可以简单的变为如下操作：
    
            //结果数组
            var results = [result0,result1,result2,result3,result4,result5,result6,result7] ;
            //映射查值
            return results[value] ;
        
        这种"优化方式"本质上是脱离if语句的思维方式的, 追求逻辑映射来优化代码，是一种框架思维。
    
3. 递归优化
    1. `Memoization`递归缓存
        递归缓存能避免重复工作，是递归算法中非常有用的技术。需要掌握：
        如下阶乘代码：
        
            var fact6 = factorial(6) ;
            var fact5 = factorial(5) ;
            var fact4 = factorial(4) ;
    
        可见, 4的阶乘被算了3次, 我们需要找到一个办法把已计算过结果的数组存储供下次使用的办法。
        大家都猜到了，最方便的自然是函数闭包。
        我们重写阶乘方法，让他有缓存功能。
        
            function memfactorial(n){
                if (!menfactorial.cache){
                    menfactorial.cache = {
                        "0" : 1 ,
                        "1" : 1 
                    } ;
                    if ( !memfactorial.cache.hasOwnProperty(n) ){
                        menfactorial.cache[n] = n*menfactorial( n-1 ) ;
                    }
                    return menfactorial.cache[n] ;
                }
            }
        
    这就是个有存储功能的`memfactorial`函数, 他的数值缓存只有自己内部能访问到, 每次调用这个函数时, 就会自动的去查找内部数值缓存, 有则直接返回该值, 非常智能。一般在优化公共函数的时候这个方法特别好用，


### 字符串和正则表达式

1. 字符串链接优化
    1. 细分链接操作：
        比如这样一个例子：`str += "one" + "two"` 
        会经历下面四个步骤:
            1. 内存中创建临时字符串( 准备接受one和two的链接值 )
            2. 连接后的字符串"onetwo"被赋值给该临时字符串
            3. 临时字符串与str当前的值连接
            4. 结果赋值给str ;
        如果我们换成这样的语句呢? `str = str + "one" + "two" `
            1. 由于是直接通过str为基础, 简单的附加字符串, 所以不需要任何临时字符串, 第一步直接加上了onetwo
            2. 结果赋值给str
        简单来说, 浏览器都会给表达书左边的字符串分配的更多的内存(除IE以外), 这就使得这种在既有的长字符串后面链接其他字符的操作变得性能优异。
        然而这却并不适用IE, 或准确来说是IE8之前的浏览器, 这些老古董会在每一次连接字符串时都把左值复制到一块新分配的内存中。
        你就可以简单的想象下，假如我在一个字符串的基础上重复追加字符串，IE7会为逐渐增大的字符串不断复制并分配内存，那么运行时间和内存消耗都将是平方关系递增。
        
            var copyStr = '我是一个用来重复链接的字符串' ;
            var resultStr = '' ;
            var repeatTime = 5000 ;
            while (repeatTime--){
                resultStr += copyStr ;
            }
        
        毫无疑问上述代码在IE7下面性能会爆炸, 将是一个十几秒的操作
        解决的方法就是避开字符拼接的思路, 用数组项合并`Array.prototype.join`的方法来拼接字符串.
        
            var copyStr = '我是一个用来重复链接的字符串' ;
            var tmpArr = [] ;
            var resultStr = '' ;
            var repeatTime = 5000 ;
            while (repeatTime--){
                tmpArr[ tmpArr.length ] = copyStr ;
            }
            resultStr = tmpArr.join('') ;
            
        这个方法将有效解决IE7链接长字符的性能问题。
        
2. 正则表达式！
    这里的内容非常有意思，我们在了解每一个优化技巧之前，都要确确实实明白问题的起因，这就要求我们对正则的执行过程非常了解。
    例一：简单分支匹配
    `/h(ello|appy) ahkari/.test('hello everyone, happy ahkari')` ;
    这个会匹配`hello ahkari`或`happy ahkari`。 在正则查找过程中, 会在第一个`h`处设下第一个起始位置, 然后立刻记下分支点`(ello|appy)`, `ello`分支会在`e`字符处失败, 返回分支点会发现`appy`第一个字符就出问题了, 证明第一个起始位置是不能匹配的, 我们抛弃起始位置, 从下一个字符一个个排查, 尝试找到新的起始位置, 最终在第二个`h`处找到, 并证明`appy`分支是可行的, 至此`happy ahkari`匹配成功。
    例二：含有重复量词的回溯匹配 
    
        var str = '<p>paras.1</p>'+
            '<img src="smiley.jpg">'+
            '<p>paras.2</p>'+
            '<div>Div.</div>';
        /<p>.*</p>/i.test(str) ;
        
    这个正则期望找到`<p>XXXXX</p>`,这样的内容,中间可以是除了换行字符外的任何玩意.
    他的三四字符`.*`吞没了这个str剩余的所有内容, 最终在字符尾部匹配失败, 于是开始回溯, 以期望找到`</p>`, 因为是**贪婪**方式, 所以他会找到离末尾最近的`</p>`。
    如果我们把`.*`换成`.*？`, 换成**惰性**方式, 他会尝试无视, `.`所要求的吞并匹配, 而是从当前位置, 也就是`<p>`的下一个位置就开始尝试匹配`</p>`, 所以他找到的结果会是一个最短的`<P>xx</p>`字符串, 和**贪婪**方式操作完全不同。
    其实通过两者运算结果你也能弄明白他们不是一回事。。。在chrome里面简单验证就看出来了。
    ![正则惰性和贪婪的区别](http://7xny7k.com1.z0.glb.clouddn.com/reg.png)
    
    优化方案比较繁杂, 这里就写些要点吧
    1. 正则有时候为何慢?
        恰恰不是匹配成功慢, 而是匹配失败太慢了. 当你用正则匹配一个大字符串的小部分, 失败的情形比成功的情形多的太多。
    2. 用简单，必须的字元开始
        原因很简单，之前我们分析例子时都是有明确的开始字符的，这对于正则引擎定位起始位置非常有帮助，如果用一些`/one|two/`,`\s{1}`这些不好定起始位置的量词分支做开头, 会减慢引擎排除明显不匹配位置的速度.
    3. 使用量词模式, 具体化匹配模式, 使后面字元互斥
    4. 减少分支数量, 缩小分支范围
        
            +-------------+--------------+
            |    替换前   |     替换后   +
            +-------------+--------------+
            |   cat|bat   |     [cb]at   |
            |   red|read  |      rea?d   |
            |   red|raw   |   r(?:ed|aw) |
            |   (.|\r|\n) |    [\s\S]    |
            +-------------+--------------+
            
    	字符集[]比分支|更快, 因为使用了非回溯而是位向量的方式
    5. 使用非捕获组
    	`(?:非捕获)`和`(捕获)`的差别仅有是否给予分组用于后续处理这一个区别, 非捕获自然性能好点
    
    6. 只捕获感兴趣的文本以减少后处理
    	在确实需要捕获的时候记住这点.
    
    7. 暴露必需的字元
    	`/^(ab|cd)/`就暴露了起始锚, 而`/(^ab|^cd)/`就没有暴露`^`而导致无意思的搜索
    
    8. 使用合适的量词
    
    9. 把正则表达式赋值给变量并重用他们
    
    10. 将复杂的正则表达式拆分为简单的片段(化繁为简)
    
    综上, 正则优化建议看完后, 其实也解决了自己一直以来的很多问题。
    比如在一开始学习正则语法的时候就有过的疑问,遇到一个需要匹配的目标字符串时会疑惑: 为什么要这样写? 那样写不行吗? 他们的意义不是一样的吗?
    现在看起来当初的疑问是很有价值的, 因为正则语法决定了他能有很多种方法来匹配目标字符, 对于同一个字符的不用查找语句, 功能上他们可能相同, 但是实现方式不尽相同, 最终导致的就是性能差异.
    
    
### 快速响应的用户界面

1. 浏览器UI进程
    浏览器用来更新ui界面和执行javascript的是一个"线程", 所以如果javascript执行时间过长, 难保UI界面不会出现假死的情况.
    需要注意的是, 本节我们所关注的问题都是为了解决一个问题: 就是如何让浏览器在执行长时间javascript代码的时候不导致UI进程假死。
    本节的"优化", 其实在总执行时间上并没有什么缩减, 只是让UI操作每次都能快读响应, 造出一种快速响应的"假象"
    那么我们直接看最终优化结果吧:
    对于一个需要长时间运行的函数, 我们给他设定了如下优化条件:
    1. js代码在执行的过程中需要有休憩间隔，让UI操作能执行
    2. 一般超过100ms的js代码执行时间就会有UI假死的迹像，我们取50ms作为UI操作最大响应间隔
        
            function timeProcessArray(items,process,callback){
                var todo = items.concat() ;
                setTimeout(fucntion(){
                    var start = +new Date() ;
                    do{
                        process(todo.shift()) ;
                    }while(todo.length>0 && (+new Date()-start<50)); //未到50ms,继续执行js
                    if (todo.length>0){
                        setTimeout(arguments.callee,25) ; //休整25ms让出线程给UI操作
                    }else{
                        callback(items) ;
                    }
                },25);
            }

2. `Web Workers`----独立于UI线程之外的js执行接口
    理解`WebWorker`只需要看一个应用场景即可, 比如parse一个超级长的json字符串, 我们另起一个`web worker`来执行这个耗时很长的解析操作
    我们主进程js代码如下:
    新建`worker`对象, `postMessage`传递`json`字符串给其, 接受`worker`对象的结果
        
        var worker = new Worker('jsonparser.js') ;
        worker.onmessage = function(event){
            var jsonDate = event.data ;
            evaluateDate(jsonDate) ;
        } ;
        worker.postMessage(jsonText) ;

    `worker`对象`jsonparser.js`代码如下:
    `self.onmessage`会在`json`数据存在时执行, 并返回结果给主js
        
        self.onmessage = function(event){
            var jsonText = event.data ; 
            var jsonData = JSON.parse( jsonText ) ;
            self.poestMessage( jsonData ) ;
        }
    
    这种辅助js进程能将含下列任务的前端执行速度大大提升
    *   编码/解码大字符串
    *   复杂数学运算(包括图像或视频处理)
    *   大数组排序
    正好最近的项目里有不少解析文件的功能, 等有空拿项目开个刀验证下吧, 这个`web worke`r目前只在部现代浏览器上实现, 好消息是我们的项目也只支持firefox和chrome。
    

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