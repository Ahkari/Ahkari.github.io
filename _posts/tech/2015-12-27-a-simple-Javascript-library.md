---
layout: post
title: 这是一个只有三行的js库
category: 技术
tags: node browserify 工作模式 javascript
description: 由三行代码窥见前端的全栈工作流
---

### 什么鬼？
起这么一个莫名其妙的标题，是因为我也想不到该叫啥名字。

其实是因为我今天总算看懂了github上那些JS的库的使用方式了。
觉得有点理解现在js工程师的主流工作模式了。于是写下现在的顿悟，以后会慢慢修改，直到我理解透彻。

### 没有工作流就是工作流
作为一个前端，如果是从HTML+CSS+JS入门一步步过来的。
那么可能会经历如下几个工作流：

* 最基础的工作流：
    HTML页面，`<link>`标签嵌入CSS样式文件，再用`<script>`标签引入js文件。
* 模块化工作流：
    1. 基于requireJS或seaJS，css老样子，js一般只引用一个加载器的文件，其余需要加载的文件都写在配置js里，在运行过程中会根据配置里的地址自动生成`<script>`标签来加载。
    2. AngularJS等大型框架，引入主框架js，剩下的大部分时间都在研究框架特性和如何和其他库兼容。
* 全栈式工作流：
    暂且称作这样吧，其实这步是融合在第二类里面的，只是第二类是完全可以独立在浏览器环境里的，而第三类必须结合node。是其他工作流的超集。

前端开发方式无数种，任何一种方式都是完成的。了解其他的工作流是有必要的，最起码要做到不能陌生。

### 以一个最简单的JS库为例

偶然发现的，一个用来判断是否是数组的库，
[isarray](https://www.npmjs.com/package/isarray)
总共只有三行代码，但是在npm里面每周却有这么惊人的下载量
![isarray惊人的下载量](http://7xny7k.com1.z0.glb.clouddn.com/isarray.png)
如果我们要用这个库，怎么用呢？如何整合呢？

### node环境下
在node环境下的工程是离不开npm的，我们进入命令行：
`$ npm install isarray` 
安装完成后，我们进入node环境测试一下
`$ node`
`var isarray = require('isarray');`
`console.log( isarray([]) )`
就打印出了结果:`true`

可见，我们轻松的引入了一个库文件。
同理，如果这是node的工程，我们在项目根目录下install的模块都可以被子文件夹中的js文件require到。
有了npm，node的依赖管理无比的轻松。

我们打开用户根目录，发现了一个名为`node_modules`的文件夹，再往里面看，我们就看到了我们所安装的库`isarray`。

node查找require模块的时候都是查找node_modules文件夹的，之前说node工程根目录下install的模块都能被require到就是这个原因：
假设根目录很深的一个js里面引用了isarray库。
node先查找本层有没有node_modules文件夹，
没有，它蹦到了外面一层，继续查有没有node_modules，
最后在根目录他找到了node_modules，进去后就发现了isarray，查找结束。


### 一个库的标配
我们之前安装了isarray，库本身只有三行，剩下那么多的玩意是做啥的呢？
![isarray里面的其他文件们](http://7xny7k.com1.z0.glb.clouddn.com/file.png)

1. 单元测试
    `test.js` 从名字就看出来他是个测试isarray的文件，我们看里面的代码，
    
        var isArray = require('./');
        var test = require('tape');
        
        test('is array', function(t){
          t.ok(isArray([])); //[]是数组时，这行测试用例通过
          t.notOk(isArray({})); //{}不是数组时，这行测试用例通过
          t.notOk(isArray(null)); //null不是数组时，这行测试用例通过
          t.notOk(isArray(false)); //false不是数组时，这行测试用例通过
        
          var obj = {};
          obj[0] = true;
          t.notOk(isArray(obj)); //obj{0:true}不是数组时，这行测试用例通过
        
          var arr = [];
          arr.foo = 'bar';
          t.ok(isArray(arr)); //arr[foo:bar]是数组时，这行测试用例通过
        
          t.end();
        });

    他依赖了isarray，同时还依赖了tape模块，这个模块我们知道是并没有的，因为我们isarray根目录下根本没有node_module，这时候我们就需要知道npm的另一个核心文件了：`package.json` ；
![package.json中的内容](http://7xny7k.com1.z0.glb.clouddn.com/packagejson.png)
    这个json是npm用来管理模块的，里面有很多的配置，我们这里关注其依赖（红框标注的部分），表明了这个isarray依赖`tape`的`~2.13.4`版本，我们有了`package.json`就不需要制定相关的依赖模块名了，直接执行：
    `$ npm install`
    结束后我们会看到`isarray`下多了一个`node_modules`文件夹，这时我们执行`test.js`，就可以看到单元测试的运行结果：
![test.js的运行结果](http://7xny7k.com1.z0.glb.clouddn.com/test.png)

2. README.md
这是库的描述文件，用makedown语法写成。
github上，会自动是根目录下的README.md并展示出来。

3. .gitignore
告诉git，忽略哪些文件的更改，在他们有变动的时候不放进版本控制里。
打开后看到我们对于这个库依赖的node_modules是不做版本控制的。

        node_modules

4. .travis.yml
这是jekyll配置文件, 自动生成静态页面会用到

### 浏览器环境
如果是那种主要应用场景在浏览器环境的前端库, 都会找到能直接链进`<script>`中的文件。
但是那种遵循common规范的模块化文件往往都不能直接拿进浏览器环境用。

本质原因是node环境下遵循commonJS规范，有着require，export，module这一系列浏览器环境下没有的全局方法。
所以将node下的模块在浏览器里面使用，只需要支持模块化全局方法就可以。

下面是一个较为完整的，把isarray融合进浏览器全部的代码。我们加以注释来研究下。

    /**
     * Require the given path.
     *
     * @param {String} path
     * @return {Object} exports
     * @api public
     */
    function require(path, parent, orig) { //全局require方法支持
      var resolved = require.resolve(path); //是否已注册过?
      // lookup failed
      if (null == resolved) { //没有注册直接抛出异常
        orig = orig || path;
        parent = parent || 'root';
        var err = new Error('Failed to require "' + orig + '" from "' + parent + '"');
        err.path = orig;
        err.parent = parent;
        err.require = true;
        throw err;
      }
      var module = require.modules[resolved]; //module指向注册过的模块
      // perform real require()
      // by invoking the module's
      // registered function
      if (!module.exports) { //如果模块没有暴露出接口
        module.exports = {};
        module.client = module.component = true;
        module.call(this, module.exports, require.relative(resolved), module); //执行模块, 暴露接口
      }
      return module.exports; //require最后返回该模块的全部接口
    }
    
    /**
     * Registered modules.
     */
    require.modules = {}; //用于存储注册的模块
    
    /**
     * Registered aliases.
     */
    require.aliases = {}; //用户存储模块依赖
    
    /**
     * Resolve `path`.
     *
     * Lookup:
     *
     *   - PATH/index.js
     *   - PATH.js
     *   - PATH
     *
     * @param {String} path
     * @return {String} path or null
     * @api private
     */
    require.resolve = function(path) { //用来查找该名字的模块是否已注册
      if (path.charAt(0) === '/') path = path.slice(1);
      var index = path + '/index.js';
      var paths = [ //这个路径有如下五种查找方式, 都会指向同一个模块
        path,
        path + '.js',
        path + '.json',
        path + '/index.js',
        path + '/index.json'
      ];
      for (var i = 0; i < paths.length; i++) {
        var path = paths[i];
        if (require.modules.hasOwnProperty(path)) return path; //存在即返回路径
      }
      if (require.aliases.hasOwnProperty(index)) {
        return require.aliases[index]; //或返回模块
      }
    };
    
    /**
     * Normalize `path` relative to the current path.
     *
     * @param {String} curr
     * @param {String} path
     * @return {String}
     * @api private
     */
    require.normalize = function(curr, path) { //
      var segs = [];
      if ('.' != path.charAt(0)) return path;
      curr = curr.split('/');
      path = path.split('/');
      for (var i = 0; i < path.length; ++i) {
        if ('..' == path[i]) {
          curr.pop();
        } else if ('.' != path[i] && '' != path[i]) {
          segs.push(path[i]);
        }
      }
      return curr.concat(segs).join('/');
    };
    
    /**
     * Register module at `path` with callback `definition`.
     *
     * @param {String} path
     * @param {Function} definition
     * @api private
     */
    require.register = function(path, definition) { //用于注册path
      require.modules[path] = definition;
    };
    
    /**
     * Alias a module definition.
     *
     * @param {String} from
     * @param {String} to
     * @api private
     */
    require.alias = function(from, to) { //用于注册依赖
      if (!require.modules.hasOwnProperty(from)) {
        throw new Error('Failed to alias "' + from + '", it does not exist');
      }
      require.aliases[to] = from;
    };
    
    /**
     * Return a require function relative to the `parent` path.
     *
     * @param {String} parent
     * @return {Function}
     * @api private
     */
    require.relative = function(parent) {
      var p = require.normalize(parent, '..');
      /**
       * lastIndexOf helper.
       */
      function lastIndexOf(arr, obj) {
        var i = arr.length;
        while (i--) {
          if (arr[i] === obj) return i;
        }
        return -1;
      }
    
      /**
       * The relative require() itself.
       */
      function localRequire(path) {
        var resolved = localRequire.resolve(path);
        return require(resolved, parent, path);
      }
    
      /**
       * Resolve relative to the parent.
       */
      localRequire.resolve = function(path) {
        var c = path.charAt(0);
        if ('/' == c) return path.slice(1);
        if ('.' == c) return require.normalize(p, path);
        // resolve deps by returning
        // the dep in the nearest "deps"
        // directory
        var segs = parent.split('/');
        var i = lastIndexOf(segs, 'deps') + 1;
        if (!i) i = 0;
        path = segs.slice(0, i + 1).join('/') + '/deps/' + path;
        return path;
      };
    
      /**
       * Check if module is defined at `path`.
       */
      localRequire.exists = function(path) {
        return require.modules.hasOwnProperty(localRequire.resolve(path));
      };
      return localRequire;
    };
    //将该模块引入浏览器的基础, 通过这个名字注册模块进require.module里面
    require.register("isarray/index.js", function(exports, require, module){
    module.exports = Array.isArray || function (arr) {
      return Object.prototype.toString.call(arr) == '[object Array]';
    };
    //描述依赖
    require.alias("isarray/index.js", "isarray/index.js");

当然我们可以用一个库来实现node模块到浏览器模块的转换,
就是有名的`browserify`

我们安装`browserify`

`$ npm install -g browserify`

以isarray中的`test.js`为例, 我们需要让他在浏览器中运行, console出结果。
我们直接：
`$ browserify test.js > test-version-for-browser.js`
`browseriy`会帮你找到该test.js所有的依赖集成进来, 如果多层依赖就会递归打包。

然后我们直接嵌入该文件进html中：
打开HTML就会在F12里看到结果:
![浏览器里面的test结果](http://7xny7k.com1.z0.glb.clouddn.com/browserTest.png)

如果我们想像node里面直接require这个模块呢? 
比如我们想在浏览器用isarray模块,

只需编译的时候加上控制参数 `-r` , 它会帮你把模块编译, 为你增加require的引用方式

`$ browserify -r index.js > isarray.js`

然后我们引入进html后, 就能正常require来使用这个模块了.

![浏览器里引用isarray](http://7xny7k.com1.z0.glb.clouddn.com/isarraybrowser.png)

需要注意的是路径引用, 拿不准的时候可以进编译好的源码里面直接看注册的名字是哪个。

![编译源码,查看注册的模块 ](http://7xny7k.com1.z0.glb.clouddn.com/existModule.png)


### so...
至此, 我们要说的玩意都说完了, 总之, node的出现把前端的整体工作模式都提高了一个档次。
了解require模块的加载使用方式，我们就不会再面对github上javascript相关模块的代码结构代码文件那么生疏那么棘手了。

这是篇重点不明，内容跑偏的blog，以后有想法会持续完善。

修订版本： 
version1.0 12/27/15 Sun 14:10   ------ 写了些isarray在node和浏览器环境下的使用方法




