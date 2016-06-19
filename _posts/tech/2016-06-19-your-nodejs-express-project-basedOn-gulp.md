---
layout: post
title:  搭建你的express中间层！
category: 技术
tags: nodejs express gulp
description: 介绍一个基于gulp工程化的nodejs中间层种子项目
---

### 知识背景
1. 前端: 
	狭义上的前端掌握html,css和javascript，据此在浏览器里面写出网页或web应用。然而这一层在整个网站的架构层面是狭小的，没有路由控制的网站，或是说，没有中间层的网站在功能上是不足以撑起多变的业务需求的。所以，现在的标准技术栈的前端是需要掌控到整个view层的。

2. nodejs
	nodejs的出现，将前端技术栈带到了“多维时代”。如果说一门语言限定了一个程序员的职能范畴的话，javascript程序员本来只能无奈的被禁锢在浏览器服务端，做些后端程序员嗤之以鼻的微小的工作。现在有了nodejs，前端工程师的职能将限定为整个view层，掌控全局。

3. express框架
	一个语言的兴起，有时候只需要一个框架。比如rails之于ruby。express虽然还没有到rails独揽江山的程度，但是已经是nodejs上最流行的服务端框架了。比起koa，他拥有完整的route和view功能。适合初次接触nodejs后端的js程序员。

4. gulp工程化工具
	基于express这种后端框架，一个能负责整个view层的项目其实已经能搞定了。但是和传统后端程序不同，前端项目里含有大量的图片，css和js脚本这些资源，他们都是在浏览器接受服务端html结果后自行发起网络才开始加载的。这些资源并没有域的限制，也不需要严格的安全性。所以他们一般都是放在CDN服务上，通过CDN的加速来实现静态资源高速下载。于是我们就需要对文件源码目录进行编译操作，实现静态资源的压缩和规则化。Gulp和两年前特别流行的grunt不同，回归编写js代码来实行任务的思路，同时基于流的“管道符处理思想”，能大大提高task任务的执行速度！

5. route路由
	所谓的路由，就是MVC模型中的controller层。在web体系下，可以理解他就是对不同的url进行不同的响应。

6. 其他的概念
	自己去看吧。也没啥好说的。今天要说的种子项目里并没有什么问题层的东西。

### 真丶五分钟弄出个后端项目
1. 来俺的Github上弄这个项目下来。[ehsy项目种子文件](https://github.com/Ahkari/ehsySeed/)，随便你克隆还是fork还是doowload zip包。

2. 去下载一个webStorm，不然你要是在命令行里搞node我就是不劝你你也搞不了。

3. 用你刚下载的webStorm打开你的萌萌的刚clone的这个文件夹

4. 在package.json上run npm install

5. 安装好了，在gulpfile.js上run default(其实因为限定了环境,不run也行)

6. 选择bin里的start, run它。

7. 你的项目已经跑起来了，不信，去浏览器访问localhost:8000试试。

### how to START?
执行start缘何能启动整个项目呢，不妨从源码上一点点看：

	
	#!/usr/bin/env node
	/**
	 * Module dependencies.
	 * 模块依赖
	 */
	var DEBUG = process.env.DEBUG;
	var debug = require('debug')('www.ehsy.com:server');
	var http = require('http');
	var app;
	if (DEBUG) { //环境切换
	    app = require('../src/app');	//debug环境下启用src目录下app程序
	}
	else {
	    try {
	        app = require('../dist/app'); //非debug环境下启用dist,也就是gulp编译后的app程序
	    }
	    catch (ex) {
	        console.error('Cannot start application, please build first.');
	    }
	}
	if (app) {
	    /**
	     * Get port from environment and store in Express.
	     */
	    var port = normalizePort(process.env.PORT || '8000'); //设定端口
	    app.set('port', port);
	    /**
	     * Create HTTP server.
	     */
	    var server = http.createServer(app); //启动express程序
	    /**
	     * Listen on provided port, on all network interfaces.
	     */
	    server.listen(port);
	    server.on('error', onError);
	    server.on('listening', onListening);
	    /**
	     * Normalize a port into a number, string, or false.
	     * 避免端口值是些啥乱七八糟的玩意
	     */
	    function normalizePort(val) {
	        var port = parseInt(val, 10);
	        if (isNaN(port)) {
	            // named pipe
	            return val;
	        }
	        if (port >= 0) {
	            // port number
	            return port;
	        }
	        return false;
	    }
	    /**
	     * Event listener for HTTP server "error" event.
	     * 绑定端口出问题了就得这么做
	     */
	    function onError(error) {
	        if (error.syscall !== 'listen') {
	            throw error;
	        }
	        var bind = typeof port === 'string'
	            ? 'Pipe ' + port
	            : 'Port ' + port;
	        // handle specific listen errors with friendly messages
	        switch (error.code) {
	            case 'EACCES':
	                console.error(bind + ' requires elevated privileges');
	                process.exit(1);
	                break;
	            case 'EADDRINUSE':
	                console.error(bind + ' is already in use');
	                process.exit(1);
	                break;
	            default:
	                throw error;
	        }
	    }
	    /**
	     * Event listener for HTTP server "listening" event.
	     * 绑定成功的logger
	     */
	    function onListening() {
	        var addr = server.address();
	        var bind = typeof addr === 'string'
	            ? 'pipe ' + addr
	            : 'port ' + addr.port;
	        debug((DEBUG ? '[DEBUG] ' : '') + 'Listening on ' + bind);
	    }
	}

### how to BUILD?
从上面可以看出，整个项目启动会根据是否处于调试环境而决定启动的是src还是dist里面的app程序。
dist目录是通过执行gulpfile里面的default task来进行的。

我们再来研读下gulpfile的工作。

	/*
	 * 项目依赖
	 */
	var gulp = require('gulp');
	var del = require('del'); //用来删除！
	var merge = require('merge-stream'); //用来整合流
	var uglify = require('gulp-uglify'); //用来压缩css和js
	var rev = require('gulp-rev'); //gulp的版本号工具
	var replace = require('gulp-rev-replace');
	var less = require('gulp-less'); //用来编译less
	/*
	 * task clean 用于删除dist目录
	 */
	gulp.task('clean', function () {
	    return del(['dist']); 
	});
	/*
	 * task compile , 依赖于clean, 将静态资源编译
	 */
	gulp.task('compile', ['clean'], function () {
	    var js = gulp.src('src/static/scripts/*.js', {base: 'src/static'})
	        .pipe(uglify()); //压缩静态目录中的js
	    var css = gulp.src('src/static/styles/*.less', {base: 'src/static'})
	        .pipe(less()); //压缩静态目录中的css
	    var img = gulp.src('src/static/images/*.less', {base: 'src/static'});
	    return merge(js, css, img)
	        .pipe(rev())
	        .pipe(gulp.dest('dist/static'))
	        .pipe(rev.manifest())
	        .pipe(gulp.dest('dist/tmp'));
	});
	/*
	 * task revision , 在每次编译时给静态资源更名增加版本号! 这个important
	 */
	gulp.task('revision', ['compile'], function () {
	    return gulp.src('dist/static/**')
	        .pipe(rev())
	        .pipe(gulp.dest('dist/static'))
	        .pipe(rev.manifest())
	        .pipe(gulp.dest('dist/tmp'));
	});
	/*
	 * task replace , 将hbs中用到这些静态资源的地方改为正确的更改后的名字
	 */
	gulp.task('replace', ['compile'], function () {
	    return gulp.src('src/views/*.hbs')
	        .pipe(replace({manifest: gulp.src('dist/tmp/rev-manifest.json')}))
	        .pipe(gulp.dest('dist/views'));
	});
	/*
	 * task copy , 依赖三个任务, 他src中除了静态文件以外的直接复制了过去
	 * 涉及静态资源编译的几个文件和目录都被排除在copy的列表之外了
	 */
	gulp.task('copy', ['clean', 'compile', 'replace'], function () {
	    return gulp.src([
	        'src/**',
	        '!src/static/**',
	        '!src/views/**'
	    ])
	        .pipe(gulp.dest('dist'));
	});
	/*
	 * task build , 最终构建! 
	 */
	gulp.task('build', ['clean', 'compile', 'replace', 'copy'], function () {
	    return del(['dist/tmp']);
	});
	/*
	 * task default 直接执行的任务
	 */
	gulp.task('default', ['clean', 'build']);

### how to RUN APP?
浏览完整个构建流程之后，我们知道项目里后端层面的代码都是没有变化的，那么我们可以正常阅读整个app入口来知晓express项目是如何启动的。

	/*
	 * 项目依赖
	 */
	var express = require('express');
	var logger = require('morgan');
	var cookieParser = require('cookie-parser');
	var bodyParser = require('body-parser');
	var app = express();
	// view engine setup
	app.set('views', __dirname + '/views');
	app.set('view engine', 'hbs');
	/*
	 * 中间件, 有时候也可以自己写中间件
	 */
	app.use(logger('dev')); //打log的中间件
	app.use(bodyParser.json()); //解析body数据的中间件
	app.use(bodyParser.urlencoded({extended: false})); //解析url参数的中间件
	app.use(cookieParser()); //解析cookie的中间件
	/*
	 * 自定义路由
	 */
	app.use('/static', express.static(__dirname + '/static')); //静态资源目录，可以直接通过路径访问
	app.use('/product', require('./routes/product')); //项目动态路由
	app.use('/', require('./routes/home')); //主页
	/*
	 * 异常
	 */
	// catch 404 and forward to error handler
	app.use(function (req, res, next) {
	    var err = new Error('Not Found');
	    err.status = 404;
	    next(err);
	});
	// error handlers
	// development error handler
	// will print stacktrace
	if (app.get('env') === 'development') {
	    app.use(function (err, req, res, next) {
	        res.status(err.status || 500);
	        res.render('error', {
	            message: err.message,
	            error: err
	        });
	    });
	}
	// production error handler
	// no stacktraces leaked to user
	app.use(function (err, req, res, next) {
	    res.status(err.status || 500);
	    res.render('error', {
	        message: err.message,
	        error: {}
	    });
	});
	module.exports = app;

这个文件没有常规的执行app.listen这个操作来启动服务，因为这里还是视作最外层sstart启动模块的依赖模块。他export出去后被start里面调用。

### how to OPERATION A REQUEST?
我们从最初一个请求开始，到返回给客户端一个页面。

顺序研究一遍！

1. 首先用户在浏览器输入地址，访问：localhost:8000/product/detail?pid=820

2. 根据app路由规则，`app.use('/product', require('./routes/product')); `他首先识别product一级path，然后把这个path下的里层path转交给`/routes/product`模块。当然我们也要意识到，在进入这个路由之前这个请求其实是按照顺序经过一系列路由，把格式规范化，把参数合法化的。

3. 在上述路由里，我们这样处理的。
	
		router.get('/detail', function (req, res, next) {
		    var pid = req.query.pid; //已经是人畜无害的简单对象参数了
		    if (pid) {
		        res.render('product-detail', {
		            pid: pid
		        }); //结束请求的一种方法，渲染view下面的这个模板
		    }
		    else {
		        res.redirect('/404'); //重定向
		    }
		});

4. render的过程是一个简单的后端模板替换，

		<p>product id: {{pid}}</p>

	我们在handlbar模板里这样写道， 在看下上面render的第二个参数，已经很明显了吧。当然，考虑到之前提及的静态资源要放在cdn服务器上，所以hbs模板里涉及静态资源的引用一定要用特殊的全局变量来控制，一般都是取app.local这个里面的全局变量的。用hbs方法来注入一些变量。我们努力完全掌控整个view的模板层。

5. 返回给浏览器html代码。浏览器拿到html后自己慢慢玩去吧！


### 然后，没有然后了
如果你能完整完成这个项目的架构，你应该是一个真正意义上的合格的前端了！

这么说，我感觉这个seed项目很值钱，你们下载一次就该支付宝转账给我钱。

配上我的这篇无码解说，实在是太棒了！

真是希望我在之前能遇上现在的自己！