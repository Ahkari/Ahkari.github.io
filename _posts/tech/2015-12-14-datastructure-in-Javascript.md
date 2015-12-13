---
layout: post
title: 数据结构操作方式汇总---基于javascript 
category: 技术
tags: 数据结构 算法 操作 javascript
description: 随时翻随时能回忆起数据结构的知识
---

作为笔记, 也方便日后参考, 所以直入主题

### 数组
数组是使用频率最高的数据结构, 几乎所有编程语言都原生支持, 这里仅介绍操作.
* 添加删除
    * Array.push()
    * Array.pop()
    * Array.shift()
    * Array.unshift()
    * Array.splice(startPos,acount,value1,value2,value3...)
* 多维数组
    * Array[one][two][three]... //生成,用for循环迭代
* 数组合并
    * Array1.concat(Array2)
* 数组迭代
    * Array.every(function(v,i){})  //返回false时迭代立即结束
    * Array.some(function(v,i){})  //返回true时迭代立即结束
    * Array.forEach(function(v,i){})  //同for循环的使用
    * var newArray = Array.map(function(v,i){})  //用return的结果生成新数组
    * var newArray = Array.filter(function(v,i){})  //用返回true的项生成新数组
    * var addResult = Array.reduce(function(prev,v,i){})  //返回数组累加结果
* 数组搜索
    * Array.indexOf(value)  
    * Array.lastIndexOf(value)
* 数组排序
    * Array.reverse() //数组反序
    * Array.sort() 
        * Array.sort() //默认全部视作字符来排序
        * Array.sort(fucntion(a,b){return a-b})  //升序排序
        * Array.sort(fucntion(a,b){return b-a})  //降序排序
        * Array.sort(comparePerson)  //自定义排序
                
                fucntion comparePerson(a,b){ //按照age属性升序排序
                    if (a.age < b.age){
                        return -1
                    }
                    if (a.age > b.age){
                        return 1
                    }
                    return 0
                } 
* 数组转为字符串
    * Array.toString()
    * Array.join('-')

### 栈
栈遵循后进先出的规则, 本质是数组的子集, 它对于添加和删除元素的操作限制更多, 我们这里来自主实现一个栈类.
* 创建
        
        function Stack(){
            var items = [] ;
            this.push = function(ele){
                items.push(ele) ;
            };
            this.pop = function(ele){
                return items.pop() ;
            }
            this.peek = function(){
                return items[items.length-1]
            }
            this.isEmpty = function(){
                return items.length === 0
            }
            this.size = function(){
                return items.length ;
            }
            this.clear = function(){
                items = [] ;
            }
            this.print = function(){
                console.log(items.toString()) ;
            }
        }
* 使用
    `var stack = new Stack() ;`
* 思考
    * 实现十进制转二进制
    * 实现十进制转任何进制

### 队列
队列遵循先进先出, 同样是数组的子集, 我们来模拟一个队列类.
* 创建
        
        function Queue(){
            var items = [] ;
            this.enqueue = function(ele){
                items.push(ele) ;
            }
            this.deque = function(){
                return items.shift() ;
            }
            this.front = function(){
                return items[0] ;
            }
            this.isEmpty = function(){
                return items.length === 0 ;
            }
            this.size = function(){
                return items.length ;
            }
            this.print = function(){
                console.log(items.toString()) ;
            }
        }
* 使用
    `var queue = new Queue()`
* 思考
    将一个数移除又立即加进来是不是就形成了循环队列了?那么击鼓传花游戏就可以完成了吧.

### 链表
链表是一种动态数据结构, 我们可以从中任意添加或移除项, 可按需扩容.
js中的数组不同于其他语言, 也能按需扩容随意增加和删除, 我们要把它和链表区分开来重新设计一个链表类.
他的特性:
链表元素不连续放置, 每个元素由一个存储本身的节点和一个指向下个元素引用组成.
移除添加元素时不需要移动其他元素, 但要使用指针
数组可以访问任何位置元素, 但是链表只能起点开始迭代查找目标元素
* 创建链表
        
        function LinkedList(){
            var Node = function(element){   //辅助类, Node节点
                this.element = element ;  //当前值
                this.next = null ;  //下个节点引用
            } ;
            var length = 0 ;
            var head = null ;
            this.append = function(element){ //链表尾部增加一个新项
                var node = new Node(element) , 
                    current ;
                if (head === null){
                    head === node ;
                }else{
                    current = head ;
                    while (current.next){
                        current = current.next ;
                    }
                    current.next = node ;
                }
                length++ ;
            } ;
            this.insert = function(position,element){ //特定位置增加一个新项
                if (position>=0 && position<=length){
                    var node = new Node(element) ,
                        current = head ,
                        previous,
                        index = 0 ;
                    if (position === 0){
                        node.next = current ;
                        head = node ;
                    }else{
                        while (index++<position){
                            previous = current ;
                            current = current.next ;
                        }
                        node.next = current ;
                        previous.next = node ;
                    }
                    length++ ;
                    return true ;
                }else{
                    return false ;
                }
            } ;
            this.removeAt = function(position){ //移除指定位置项
                if (position>-1 && position<length){    //检查越界
                    var current = head ,
                        previous ,
                        index = 0 ;
                    if ( position === 0 ){
                        head = current.next ;   //移除第一项
                    }else{
                        while (index++<position){
                            previous = current ;
                            current = current.next ;
                        }
                        previous.next = current.next ; //将previous与current下一项链接, 跳过current来移除他
                    }
                    length-- ;
                    return current.element ;
                }else{
                    return null ;
                }
            } ;
            this.remove = function(element){ //移除一项, 基于indexOf方法, 通过元素查索引, 然后移除索引处元素
                var index = this.indexOf(element) ;
                return this.removeAt(index) ;
            } ;
            this.indexOf = function(element){ //返回元素在列表中的索引
                var current = head ,
                    index = -1 ;
                while(current){
                    if (element === current.element){
                        return index ;
                    }
                    index++;
                    current = current.next ;
                }
                return -1 ;
            } ;
            this.isEmpty = function(){ //判断链表是否为空
                return length === 0 ;
            } ;
            this.size = function(){ //返回链表包含的元素个数
                return length ;
            } ;
            this.toString = function(){ //因为用自定义Node类, 所以得重写toString()方法
                var current = head ,
                string = '';
                while(current){
                    string += current.element ;
                    current = current.next ;
                }
                return string ;
            } ;
            this.getHead = function(){  //获取head
                return head ;
            } ;
            this.print = function(){ //打印
                console.log( this.toString() ) ;
            } ;
        }

* 使用
    `var link = new LinkedList() ;` 
* 拓展
    * 双向链表
        * 比起单向链表, 他能找到前一个元素
        * 比单向链表多一个往前搜索节点的能力
    * 循环链表
        * 如果是单向链, 那么循环链表的最后一个元素next指向head
        * 如果是双向链, 那么循环链表的最后一个元素next指向head, 同时nead的prev指向最后一个元素

### 集合
集合是由一组无序且唯一的项组成。

* 创建集合
    
        function Set(){
            var items = {} ;  //使用对象来存取
            this.add = function(value){
                if (!this.has(value)){
                    items[value] = value ;
                    return true ;
                }
                return false ;
            } ;
            this.remove = function(value){
                if (this.has(value)){
                    delete items[ value ] ;
                    return true ;
                }
                return false ;
            } ;
            this.has = function(value){
                //return value in set ;
                return set.hasOwnProperty(value) ;
            } ;
            this.clear = function(){
                items = {} ;
            } ;
            this.size = function(){
                return Object.keys(items).length ; //Object类内建函数来获取对象长度
            } ;
            this.values = function(){
                return Object.keys(items) ;  //获取所有属性
            } ;
        }
   
* 使用集合
        `var set = new Set() ;`
* 集合操作
    * 并集: 返回一个包含两个集合中所有元素的新集合
        
            this.union = function(otherSet){
                var unionSet = new Set() ;
                var values = this.values() ;
                for (var i = 0; i<values.length;i++){
                    unionSet.add(values[i]) ;
                }
                values = otherSet.values() ;
                for (var i = 0; i<values.length;i++){
                    unionSet.add(values[i]) ;
                }
                return unionSet ;
            }

    * 交集: 返回一个包含两个集合中共有元素的新集合
    
            this.intersection = function(otherSet){
                var intersectionSet = new Set() ;
                var values = this.values() ;
                for (var i=0;i<values.length;i++){
                    if (otherSet.has(values[i])){
                        intersectionSet.add( values[i] ) ;
                    }
                }
                return intersectionSet ;
            }
            
    * 差集: 返回一个包含所有存在于第一个集合且不存在与第二个集合的元素的新集合
            
            this.difference = function(otherSet){
                var differenceSet = new Set() ;
                var values = this.values() ;
                for (var i=0;i<values.length;i++){
                    if (!otherSet.has(values[i])){
                        differenceSet.add(values[i]) ;
                    }
                }
                return differenceSet ;
            }
    
    * 子集: 验证一个给定的集合是否是另一集合的子集
        
            this.subSet = function(otherSet){
                if (this.size()>otherSet.size()){
                    return false ;
                }else{
                    var values = this.values() ;
                    for (var i=0;i<values.length;i++){
                        if (!otherSet.has(values[i])){\
                            return false ;
                        }
                    }
                    return false ;
                }
            }
    
### 字典和散列表
字典散列表和集合一样都是可以存储不重复的值, 区别在于集合仅关注值本身, 而字典和集合关注的是"键"与"值"两个数据, 又称 "映射" 。

* 创建字典

        function Dictionary(){
            var items = {} ;
            this.set = function(key,value){
                items[ key ] = value ;
            } ;
            this.has = function(key,value){
                return key in items ;
            } ;
            this.remove = function(key){
                if (this.has(key)){
                    delete items[key] ;
                    return true ;
                }
                return false ;
            } ;
            this.get = function(key){
                return this.has(key)?items[key]:undefined ;
            } ;
            this.values = function(){
                var values = [] ;
                for (var k in items){
                    if (this.has(k)){
                        values.push( items[ k ] ) ;
                    }
                }
                return values ;
            } ;
            this.claer = function(){} ; //同set
            this.size = function(){} ; //同set
            this.keys = function(){} ; //同set
            this.getItems = function(){} ; //同set
        }
* 使用字典类
    `var dectionart = new Dictionary() ;`

* 创建散列表, 又称哈希表
他是字典类的一种散列表表现方式, 根据键名就能唯一确定散列值, 根据散列值能快速找到其在表中的值。
核心是`散列算法`, 比如用ASCII码值来唯一确定每个键值的散列值

        function HashTable(){
            var table = [] ;
            var loseloseHashCode = function(key){  //散列函数
                var hash = 0 ;
                for (var i=0;i<key.length;i++){
                    hash+=key.charCodeAt(i) ; //获取字符串指定位置处的ASCII码
                }
                return hash % 37 ;
            } ;
            this.put = function(key,value){
                var position = loseloseHshCode(key) ;
                console.log(position+'-'+key) ;
                table[position] = value ;
            } ;
            this.remove = function(key){
                table[ loseloseHshCode(key) ] = undefined ;
            } ;
            this.get = function(key){
                this table[ loseloseHshCode(key) ] ;
            } ;
        }
* 使用散列表(哈希表)
    `var hash = new HashTable() ;`
* 重要思考 其一
    哈希表完全根据散列函数计算出的值位来存储数据, 那么散列函数计算出重复的值的时候, 后put进去的数据就会把前面的给覆盖掉, 于是, 在散列表基础上如何处理"冲突"就变成了特别重要的事。
    1. 分离链接
        如果多个键指向同一个位置, 那么就在那个位置上创建一个链表来存储数据, 需要重写put, get, remove方法
                
                this.put = function(key,value){
                    var position = loseloseHashCode(key) ;
                    if (table[position] === undefined){
                        table[position] = new LinkedList() ;
                    }
                    table[position].append(new ValuePair(key,value)) ;
                }
                this.get = function(key){
                    var position = loseloseHashCode(key) ;
                    if ( table[position] !== undefined ){
                        var current = table[position].getHead() ;
                        while (current.next){ //遍历链表, 返回key满足的
                            if (current.element.key === key){
                                return current.element.value ;
                            }
                            current = current.next ;
                        }
                        if (current.element.key === key){
                            return current.element.value ; //如果在最后一个, 要特殊处理下
                        }
                    }
                }
                this.remove = function(key){
                    var position = loseloseHashCode(key) ;
                    if ( table[position] !== undefined ){
                        var current = table[position].getHead() ;
                        while (current.next){
                            if (current.element.key === key){
                                table[position].remove(current.element) ;
                                if (tabel[position].isEmpty()){
                                    table[position] = undefined ;
                                }
                                return true ;
                            }
                            current = current.next ;
                        }
                        if (current.element.key === key){
                            table[position].remove(current.element) ;
                            if (table[position].isEmpty()){
                                table[position] = undefined ;
                            }
                            return true l
                        }
                    }
                    return false ;
                }
    2. 线性探查
        向表中某位置加新元素, 如果索引index位置已占据, 尝试index+1位置, 以此类推
        仍然需要重写put(), get(), remove()方法。
        
            this.put = function(key,value){
                var position = loseloseHashCode(key) ;
                if (table[position]===undefined){
                    table[position] = new ValuePair(key,value) ;
                }else{
                    var index = ++position ;
                    while (table[index]!==undefined){
                        index++ ;
                    }
                    table[index] = new ValuePair(key,value) ;
                }
            } ;
            this.get = function(key){
                var position = loseloseHshCode(key) ;
                if (table[position].key !== undefined){
                    if (table[position].key === key){
                        return table[position].value ;
                    }else{
                        var index = ++position ;
                        while ( table[index] === undefined || table[index].key !== key ){
                            index++ ;
                        }
                        if ( table[index].key = key ){
                            return table[index].value ;
                        }
                    }
                }
                return undefined ;
            } ;
            this.remove = function(){
                //类似get方法, 不过只需对每个return(最后undefined应该不需要)操作改成赋值语句`=undefined` ;    
            }
* 重要思考 其二
之前为什么要解决冲突完全是因为哈希值生成的函数不够理想,
如果我们换一个优秀点的散列函数呢, 比如下面这个:
    
        var djb2HashCode = function(key){
            var hash = 5381 ;
            for (var i = 0 ;i<key.length;i++){
                hash = hash*33 + key.charCodeAt(i) ;
            }
            return hash % 1013 ;
        }

当然这不是最好的散列函数, 但是被社区最推崇的散列函数之一。
* 重要思考 其三
认知到哈希表本质, 他是把key和value之间的关系具体化了, 通过key就能直接定位到value的实际位置, 而不是像字典一样通过key值来黑盒式搜索。这个“定位方式”的生成过程就是散列函数。
因为key的可能值是无限的, 所以一个有限的哈希表必然会出现位置重复的情况, 这时候就会用到解决冲突的思路来优化哈希表了。 
明白哈希表的实质和局限, 你就能把它与其他数据结构有效区分并合理构建哈希表了。

### 树
树也是一种非顺序数据结构，她对于存储需要快速查找的数据非常有用。他有如下特性：
一个树结构由一系列具有父子关系的节点构成；
每个节点都有一个父节点（除了根）和零到多个子节点；
顶部是根节点，没有父节点，其余的节点就叫节点；
至少有一个子节点的节点称为内部节点，没有子元素的节点称为外部节点或叶节点；
节点的祖先节点和后代节点是比较宽泛的“往上”或“往下”的所有节点统称；
子树由节点和它的后代构成；
节点的深度，取决于它祖先节点的数量；
树的高度取决于所有节点深度的最大值；

* 创建二叉树和二叉搜索树
二叉搜索树`BST`是树的子集，只允许在左侧节点存储比父节点小的值, 在右侧存储比父节点大的值

        function BinarySearchTree(){
            var Node = function(key){
                this.key = key ;
                this.left = null ;
                this.right = null ;
            } ;
            var root = null ;
            var insertNode = function(node,newNode){}
            this.insert = function(key){} ;
            this.search = function(key){} ;
            this.inOrderTraverse = function(){} ;
            this.preOrderTraverse = function(){} ;
            this.postOrderTraverse = function(){} ;
            this.min = function(){} ;
            this.max = function(){} ;
            this.remove = function(key){} ;
        }

* 插入节点: 对应上面公有方法insert和私有方法insertNode 

        var insertNode = function(node,newNode){
            if (newNode.key < node.key){
                if (node.left === null){
                    node.left = newNode ;
                }else{
                    insertNode(node.left,newNode)
                }
            }else{
                if (node.right === null){
                    node.right = newNode ;
                }else{
                    insertNode(node.right,newNode) ;  //私有方法递归调用自身
                }
            }
        }
        this.insert = function(key){ //向树种插入一个节点, 需要调用一个私有方法
            var newNode = new Node(key) ;
            if (root === null){
                root = newNode ;
            }else{
                insertNode(root,newNode) ;
            }
        } ;
* 遍历二叉树
    所谓遍历, 其核心就是自身的递归 ;
    * 中序遍历: 节点本身放在中间的位置来访问, 也就是先访问左边比较小的, 再访问中间自身, 再访问右边比较大的, 期望会生成一行从小到大排序的数据
        
            var inOrderTraverseNode = function(node,callback){
                if (node !== null){ //递归终止的条件
                    inOrderTraveseNode(node.left,callback) ;
                    callback(node.key) ;
                    inOrderTraveseNode(node.right,callback) ;
                }
            }
            this.inOrderTraverse = function(callback){
                inOrderTraverseNode(root,callback) ;
            }
    
    * 先序遍历: 节点本身最先访问, 然后访问左边比较小的, 再访问右边比较大的
    
            var preOrderTraverseNode = function(node,callback){
                if (node !== null){
                    callback(node.key) ;
                    preOrderTraverseNode(node.left,callback) ;
                    preOrderTraverseNode(node.right,callback) ;
                }
            }
            this.preOrderTraverse = function(callback){
                preOrderTraverseNode(node,callback) ;
            }
    
    * 后序遍历: 先访问左侧小的, 再访问右侧大的, 最后是节点自身
        
            var postOrderTraverseNode = function(node,callback){
                if (node !== null){
                    postOrderTraverseNode(node.left , callback) ;
                    postOrderTraverseNode(node.right, callback) ;
                    callback(node.key) ;
                }
            }
            this.postOrderTraverse = function(callback){
                poestOrderTraverseNode(root,callback) ;
            }

* 搜索树
    所谓搜索, 对于树来说同样也是递归遍历实现。
    * 搜素最小值和最大值, 两个方法类似
            
            var minNode = function(node){
                if (node){
                    while(node && node.left !== null){
                        node = node.left ;
                    }
                    return node.key ;
                }
                return null ;
            }
            this.min = function(){
                return minNode(root) ;
            }
    
    * 搜索一个特定的值
        
            var searchNode = function(node,key){
                if (node===null){
                    return false ;
                }
                if ( key < node.key ){
                    return searrchNode(node.left,key) ;
                }else if( key > node.key ){
                    return searchNode(node.right,key) ;
                }else{
                    return true ;
                }
            }
            this.search = function(key){
                return searchNode(root,key) ;
            } ;
            
* 移除一个节点

        var removeNode = function(node,key){
            if ( node === null ){
                return null ;
            }
            if ( key < node.key ){
                node.left = removeNode(node.left, key) ;
                return node ;
            }else if( key < node.key ){
                node.right = removeNode(node.right, key) ;
                return node ;
            }else{
                if ( node.left === null && node.right === null ){
                    node = null ; //无子节点, 直接置null
                    return node ;
                }
                if ( node.left === null ){
                    node = node.right ; //只有右节点, 直接把右节点作为本节点
                    return node ;
                }else if ( node.right === null ){ 
                    node = node.left ;  //只有左节点, 直接把左节点作为本节点
                    return node ;
                }
                var aux = findMinNode( node.right ) ; //剩下就是含有左右两个节点的情况, 这时候需要找出右侧最小的一个, 这个值替换掉当前节点时, 这里还是会满足"右边都比节点大, 左边都比节点小的情况"!
                node.key = aux.key ;  //节点值替换为右侧最小的值
                node.right = removeNode( node.right , aux.key ) ;  //节点右侧指向删除了最小节点的node.right
                return node ;  //这个return node非常重要, 确保了每次返回的都是更新过的node节点
            }
        } ;
        this.remove = function( key ){
            root = removeNode(root, key) ;
        } ;
        
        
### 图
图是一种非线性数据结构, 是网络结构的抽象模型。是一组由边连接的节点（顶点）。
> 一个图`G = (V,E)` 由一下元素组成, V: 一组顶点, E: 一组边, 连接V中的顶点。

* 图的术语
由一条边连接在一起的顶点称为相邻顶点。
一个顶点的度是其相邻顶点的数量。
路径是顶点的一个连续序列。
简单路径要求不包含重复的顶点。环也是一个简单路径。
如果图中不存在环，则该图是五环的，如果图中每两个顶点间都存在路径，则该图是连通的。
>图可以是有向的也可以是无向的, 有向图的边都有一个方向。
如果图中每两个顶点间在双向上都存在路径，则该图是强连通的。
图可以是未加权的或是加权的, 边可以被赋予权值。

* 图的表示
    * 邻接矩阵
        二维数组展现, 相连的点就置1, 否则置0, 缺点是稀疏的图会有太多的0
    * 邻接表
        一个顺序的表, 只展现和他相连的点
    * 关联矩阵
        如果A到B的边能连上, 就置1, 一般用于边的数量比顶点多的情况

* 创建图 -- 以无向图举例
    
        function Graph(){
            var vertices = [] ;
            var adjList = new Dictionary() ;
        }

    向图中添加新的顶点, 同时把他关联的边也初始化出来, 为一个空数组(数组里面是和顶点相连的其他点):
        
        this.addVertex = function(v){
            vertices.push(v) ;   //
            adjList.set(v,[]) ;
        }
    
    对一个顶点增加边, 接受这两个要链接的顶点作为参数
        
        this.addEdge = function(v,w){
            adjList.get(v).push(w) ;
            adjList.get(w).push(v) ;
        }
        
    辅助方法: toString()
    
        this.toString = function(){
            var s = '' ;
            for (var i=0;i<vertices.length;i++){
                s += vertices[i] + ' -> ' ;
                var neighbors = adjList.get( vertices[i] ) ;
                for (var j=0; j<neighbors.length;j++){
                    s += neighbors[j] + ' ' ;
                }
                s += '\n' ;
            }
            return s ;
        }
        
* 图的遍历
    * 广度优先搜索: 以队列作为数据结构, 将顶点存入队列中, 最先入队列的顶点先被探索
    * 深度优先搜索: 以栈作为数据结构, 将顶点存入栈中, 沿着路径探索, 存在新的相邻节点就去访问
    
        当理解了广度搜索是用队列排序的话, 问题就好理解了, 下面实现一个广度搜索:
            
        1. 创建1一个队列Q
        2. 将v标注为被发现的(灰色), 并将v入队列Q
        3. 如果Q非空, 则运行以下步骤
            1. 将u从Q中出队列
            2. 将标注u为被发现的(灰色)
            3. 将u所有未被访问过的邻点(白色)入队列
            4. 将u标注为已被探索过的(黑色)
    
                    var initializeColor = function(){ //颜色初始化. 开始所有的顶点都未被探索
                        var color = [] ;
                        for (var i=0 ; i<vertices.length;i++){
                            color[vertices[i]] = 'white' ;
                        }
                        return color ;
                    }
                    this.bfs = function(v,callback){
                        var color = initializeColor(),
                            queue = new Queue() ;
                        queue.enqueue(v) ;
                        while (!queue.isEmpty()){
                            var u = queue.dequeue() ,
                                neighbors = adjList.get(u) ;
                            color[u] = 'grey' ;
                            for (var i=0;i<neighbors.length;i++){
                                var w = neighbors[i] ;
                                if (color[w] === 'white'){
                                    color[w] = 'grey' ;
                                    queue.enqueue(w) ;
                                }
                            }
                            color[u] = 'black' ;
                            callback && callback(u) ;
                        }
                    }
    
        思考如何实现深度优先搜索
        1. 标注v为被发现的(灰色)
        2. 对于v的所有为访问的邻点w, 执行访问w操作
        3. 标注v为已搜索的(黑色)
            
                var dfsVisit = function(u,color,callback){
                    color[u] = 'grey' ;
                    if (callback){
                        callback(u) ;
                    }
                    var neighbors = adjList.get(u) ;
                    for (var i=0 ;i<neighbors.length;i++){
                        var w = neighbors[i] ;
                        if (color[w] === 'white'){
                            dfsVisit(w,color,callback) ;
                        }
                    }
                    color[u] = 'black' ;
                } ;
                this.dfs = function(callback){
                    var color = initializeColor() ;
                    for (var i=0;i<vertices.length;i++){
                        if (color[vertices[i]]==='white'){
                            dfsVisit(vertices[i],color,callback) ;
                        }
                    }
                }
    
### 排序和搜索算法
排序和搜索算法广泛运用在待解决的日常问题中

* 排序算法
    * 冒泡排序 -- 时间复杂度O2
    * 选择排序 -- 时间复杂度O2
        找到数据结构中最小的放在第一位, 第二小的放在第二位
    * 插入排序
        实际上是从头开始构建了数组, 一个个的加进去, 所以能保证位置绝对正确
    * 归并排序 -- 时间复杂度`O(Nlogn)` -- 分治思想
        将大数组分割, 直到不能再分, 在最小颗粒度上再一步步拼接回来, 每次拼接需要排序, 最后归并完成的时候得到的就是有序的数组
    * 快速排序 -- 时间复杂度`O(Nlogn)` -- 分治思想
        1. 选择中间一项作为主元
        2. 创建两个指针, 左边指向第一项, 右边指向最后一项, 移动左指针直到找到个比主元大的, 左边停止, 右边开始移动直到找到个比主元小的, 那就把左右指的内容互换, 重复这个过程, 最后, 主元左边的都比主元小了, 主元右侧的都会比主元大, 这就是划分操作
        3. 对划分后的小数组重复之前操作, 直至数组已完全排序
    
* 搜索算法
    * 顺序搜索 -- 最低效
    * 二分搜索 
        * 要求被搜索的数据结构已排序
        * 选择数组中值
        * 如果待搜索值比中值小, 则查左边, 否则查右边
        
### 算法补充知识
* 递归
    这是种程序设计思想, 和分治理念息息相关
* 动态规划
    是一种将复杂问题分解成更小的子问题来解决的优化技术
* 贪心算法
    遵循一种近似解决问题的技术, 期盼通过每个阶段的局部最优选择, 从而达到全局的最优。
        