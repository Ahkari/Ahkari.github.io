---
layout: post
title: PIXI v3源码解析 其二
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第二章
---

之前我们做做热身看了看工具类，现在正式进入游戏引擎核心。
DisplayObject类需要一个我们之前没有看的类, RenderTexture类。
他在DiaplayObject类上是这么引入的，

`RenderTexture = require('../textures/RenderTexture'),`

这个类用来辅助创建渲染材质。在我们要渲染复杂内容的时候会用到。

下面先我们插播材质类的解析，按照他的引用一步步分析。

### BaseTexture = require('./BaseTexture'),
渲染材质类依赖的第一个类是基础材质类BaseTexture。

构造时接受三个参数, 分别是材质源, 拉伸模式, 分辨率。

构造时如果材质参数存在则开始构造材质。

他自身需要关注的属性如下几个：
    
* resolution分辨率, 默认为1
* 基础宽高
* 真实宽高(假设材质需要拉伸的话)
* scaleMode拉伸模式, 线性或插值
* hasLoaded是否已载入成功
* isLoading是否正在载入中
* premultipliedAlpha是否需要乘以透明度参数, 只webGL有效

需要注意到，

在该js的16行：`EventEmitter.call(this);`,

163行， `BaseTexture.prototype = Object.create(EventEmitter.prototype);`

该类的继承自事件类。从基础材质类开始, PIXI终于开始引入了事件机制, 往后的源码中将有很多的hock函数。

那么我们中途稍微学习下这个EventEmitter的源码，我们可以通过查看pixi.js浏览器版本来看node里定义的这个模块代码都有啥：

    var prefix = typeof Object.create !== 'function' ? '~' : false; //是否需要前缀
    function EE(fn, context, once) { //监听器，含有需要执行的方法，执行源，以及是否单次这三个属性
      this.fn = fn;
      this.context = context;
      this.once = once || false;
    }
    function EventEmitter() { /* Nothing to set */ }
    EventEmitter.prototype._events = undefined;
    EventEmitter.prototype.listeners = function listeners(event, exists) {
      var evt = prefix ? prefix + event : event //加前缀
        , available = this._events && this._events[evt]; //获取指定名称的事件下绑定的回调函数集合
      if (exists) return !!available; //只是想知道这个事件有没有注册
      if (!available) return []; //如果可用事件为空
      if (available.fn) return [available.fn]; //如果不是数组，是单个含fn属性的
      for (var i = 0, l = available.length, ee = new Array(l); i < l; i++) {
        ee[i] = available[i].fn; //将数组内容赋给ee
      }
      return ee;
    };
    EventEmitter.prototype.emit = function emit(event, a1, a2, a3, a4, a5) { //事件触发器
      var evt = prefix ? prefix + event : event;
      if (!this._events || !this._events[evt]) return false; //获取存储的事件
      var listeners = this._events[evt] //监听的函数列表
        , len = arguments.length
        , args
        , i;
      if ('function' === typeof listeners.fn) {
        if (listeners.once) this.removeListener(event, listeners.fn, undefined, true);
        switch (len) { //将listeners.context作为触发对象，执行回调
          case 1: return listeners.fn.call(listeners.context), true;
          case 2: return listeners.fn.call(listeners.context, a1), true;
          case 3: return listeners.fn.call(listeners.context, a1, a2), true;
          case 4: return listeners.fn.call(listeners.context, a1, a2, a3), true;
          case 5: return listeners.fn.call(listeners.context, a1, a2, a3, a4), true;
          case 6: return listeners.fn.call(listeners.context, a1, a2, a3, a4, a5), true;
        }
        for (i = 1, args = new Array(len -1); i < len; i++) { //多于6个参数时
          args[i - 1] = arguments[i];
        }
        listeners.fn.apply(listeners.context, args);
      } else {
        var length = listeners.length
          , j;
        for (i = 0; i < length; i++) {
          if (listeners[i].once) this.removeListener(event, listeners[i].fn, undefined, true);
          switch (len) {
            case 1: listeners[i].fn.call(listeners[i].context); break;
            case 2: listeners[i].fn.call(listeners[i].context, a1); break;
            case 3: listeners[i].fn.call(listeners[i].context, a1, a2); break;
            default:
              if (!args) for (j = 1, args = new Array(len -1); j < len; j++) {
                args[j - 1] = arguments[j];
              }
              listeners[i].fn.apply(listeners[i].context, args);
          }
        }
      }
      return true;
    };
    EventEmitter.prototype.on = function on(event, fn, context) { //事件注册器
      var listener = new EE(fn, context || this) //新的监听器
        , evt = prefix ? prefix + event : event;
      if (!this._events) this._events = prefix ? {} : Object.create(null);
      if (!this._events[evt]) this._events[evt] = listener;
      else {
        if (!this._events[evt].fn) this._events[evt].push(listener); //放进指定事件的监听列表
        else this._events[evt] = [
          this._events[evt], listener
        ];
      }
      return this;
    };
    EventEmitter.prototype.once = function once(event, fn, context) {
      var listener = new EE(fn, context || this, true) //和on唯一不一样的地方就是他是单次的
        , evt = prefix ? prefix + event : event;
      if (!this._events) this._events = prefix ? {} : Object.create(null);
      if (!this._events[evt]) this._events[evt] = listener;
      else {
        if (!this._events[evt].fn) this._events[evt].push(listener);
        else this._events[evt] = [
          this._events[evt], listener
        ];
      }
      return this;
    };
    
源于node的这个事件模块的代码我们看完了，就可以理解pixi里面的这些事件机制了。

基本套路就是on或once对指定事件注册回调方法，通过emit来触发。

    BaseTexture.prototype.update = function ()
    {
        ....
        this.emit('update', this); //update最后, 触发他的update事件
    };
    ...
    source.onload = function ()
        {
            '''
            scope.emit('loaded', scope); //资源载入完毕时, 会触发loaded事件
        };

下面我们节选一个方法来看pixi对于整体框架里的对象是如何管理的:

    BaseTexture.prototype.destroy = function ()
    {   //材质的销毁方法
        if (this.imageUrl) //source是image转换来的
        { 
            delete utils.BaseTextureCache[this.imageUrl]; //删除这个图像url在基础材质中的缓存
            delete utils.TextureCache[this.imageUrl]; //删除这个图像url在材质中的缓存
            this.imageUrl = null; //将这个材质对象url置空
            if (!navigator.isCocoonJS)
            {
                this.source.src = ''; //image对象src置空
            }
        }
        else if (this.source && this.source._pixiId) //source并非图像转换来
        {
            delete utils.BaseTextureCache[this.source._pixiId]; //靠_pixiId删除映射
        }
        this.source = null; //删除
        this.dispose(); //触发删除该材质后的事件
    };

从这个销毁方法我们可以看到: 

一个健壮的类库对于其中的每个对象都是精心管理着的, 一个材质对象新建时我们会把他的信息用标识缓存起来, 统一管理还能确保不重复创建。 

同理, 销毁材质时需要把相关的内容都删除干净，去除引用。最后还可能有销毁完毕后的处置方法，让某些用到这个材质的对象都会做出相应的响应操作。


我们在阅读这部分源码的时候会心生一个疑惑，就是基础材质的第一个参数source究竟是一个什么样的对象呢？

在基础材质里对于这个对象我们操作了很多次，和他的src属性打交道，同时还有getContext方法。这个source究竟是image还是canvas对象啊？

这时候我们可以翻看官方API文档或看源码方法注释

` @param source {Image|Canvas} the source object of the texture.`

可见这个source可以是image对象或canvas对象, PIXI会根据对象检测自动处理的。之前源码中许多特性不够明确让人疑惑的部分很多都是分情况处理image或canvas导致的。


基础材质源码的最后，有两个方法完美解决了刚才的疑惑，下面就是PIXI对于image和canvas的分别处理的入口方法。

这两个方法没有挂载在原型链上。可以直接作为方法调用，并返回一个`new BaseTexture()`对象。简简单单就实现了基础材质的无new式创建！

    BaseTexture.fromImage = function (imageUrl, crossorigin, scaleMode)
    {
        var baseTexture = utils.BaseTextureCache[imageUrl];
        if (crossorigin === undefined && imageUrl.indexOf('data:') !== 0)
        {
            crossorigin = true;
        }
        if (!baseTexture)
        {
            var image = new Image();//document.createElement('img');
            if (crossorigin)
            {
                image.crossOrigin = '';
            }
            baseTexture = new BaseTexture(image, scaleMode); //在外面看起来是个无new式创建
            baseTexture.imageUrl = imageUrl;
            image.src = imageUrl; //重要，这才是从image对象链接到基础材质类的关键起始一步。
            utils.BaseTextureCache[imageUrl] = baseTexture; //材质注册
            baseTexture.resolution = utils.getResolutionOfUrl(imageUrl);
        }
        return baseTexture;
    };
    BaseTexture.fromCanvas = function (canvas, scaleMode)
    {
        if (!canvas._pixiId)
        {
            canvas._pixiId = 'canvas_' + utils.uid(); //canvas通过_pixiId标识
        }
        var baseTexture = utils.BaseTextureCache[canvas._pixiId];
        if (!baseTexture)
        {
            baseTexture = new BaseTexture(canvas, scaleMode);
            utils.BaseTextureCache[canvas._pixiId] = baseTexture;
        }
        return baseTexture;
    };

下一个材质是普通材质Texture类, 然而他又依赖一个VideoBaseTexture类,

我们先看这个类:

### VideoBaseTexture = require('./VideoBaseTexture'),

"音乐材质"类, PIXI中音乐作为资源的一种来对待。

核心方法`PIXI.VideoBaseTexture.fromUrl()`有四种加载url资源的方式。

有了基础材质我们看的一头雾水的经验，这次我们从入口方法开始看：

    VideoBaseTexture.fromUrl = function (videoSrc, scaleMode)
    {
        var video = document.createElement('video'); //这里是dom和canvas交互的最初始步骤
        if (Array.isArray(videoSrc))
        {
            for (var i = 0; i < videoSrc.length; ++i)
            { //构建多层dom
                video.appendChild(createSource(videoSrc[i].src || videoSrc[i], videoSrc[i].mime));
            }
        }
        else
        { //单层dom
            video.appendChild(createSource(videoSrc.src || videoSrc, videoSrc.mime));
        }
        video.load();
        video.play();
        return VideoBaseTexture.fromVideo(video, scaleMode);
    };
    VideoBaseTexture.fromUrls = VideoBaseTexture.fromUrl;
    function createSource(path, type) //在video中创建source，是video特性。
    {
        if (!type)
        {
            type = 'video/' + path.substr(path.lastIndexOf('.') + 1);
        }
        var source = document.createElement('source');  
        source.src = path; 
        source.type = type;
        return source;
    }

在这里主要是资源在dom部分的处理，然后我们进fromVideo看下：

    VideoBaseTexture.fromVideo = function (video, scaleMode)
    {
        if (!video._pixiId)
        {
            video._pixiId = 'video_' + utils.uid();
        }
        var baseTexture = utils.BaseTextureCache[video._pixiId];
        if (!baseTexture)
        {
            baseTexture = new VideoBaseTexture(video, scaleMode); //桥接至PIXI框架
            utils.BaseTextureCache[ video._pixiId ] = baseTexture;
        }
        return baseTexture;
    };

fromVideo将dom资源顺利的引入到PIXI的VideoBaseTexture类里，并携带了video标签对象。这个类便不再难以理解了。

VideoBaseTexture类继承自BaseTexture类。所以onload这些过程便不需要再写了。

只需要对音频资源做一些额外的处理。

多出了一些原型方法，他们会在video触发相应的事件时触发，并修改音频材质对象的状态。

`source.addEventListener('canplay', this._onCanPlay);`

`source.addEventListener('canplaythrough', this._onCanPlay);`

`source.addEventListener('play', this._onPlayStart.bind(this));`

`source.addEventListener('pause', this._onPlayStop.bind(this));`


下面是Texture依赖的另一个类TextureUvs

`TextureUvs = require('./TextureUvs'),`

这个类内容较少, 用来存储`Uvs of a texture`（猜测是预览图像，实现对象展示时的快照功能）。

我们接着看材质类Texture

### Texture = require('./Texture'),

最终的这个材质类集合了之前的所有的辅助材质类。
从他所暴露出的接口就可以看出，无论是来自image还是canvas还是video，都会用对应的辅助类来处理。

Texture.fromImage = function (imageUrl, crossorigin, scaleMode){...}
Texture.fromFrame = function (frameId){...}
Texture.fromCanvas = function (canvas, scaleMode){...}
Texture.fromVideo = function (video, scaleMode){...}
Texture.fromVideoUrl = function (videoUrl, scaleMode){...}
Texture.addTextureToCache = function (texture, id){...}
Texture.removeTextureFromCache = function (id){...}

上述方法里都会将生成的BaseTexture作为参数来初始化Texture, 例如这样：

    Texture.fromImage = function (imageUrl, crossorigin, scaleMode)
    {
        var texture = utils.TextureCache[imageUrl];
        if (!texture)
        {
            texture = new Texture(BaseTexture.fromImage(imageUrl, crossorigin, scaleMode));
            utils.TextureCache[imageUrl] = texture;
        }
        return texture;
    };
    
拿到的BaseTexture对象会作为new Texture对象的BaseTexture属性。

所以我们就能理清这两个类之间的关系了，BaseTexture是Texture的辅助类，是用来单纯展现或表示其材质的。

Texture在他们之上添加了frame属性（frame是PIXI的矩形类），制定了材质的显示范围。他是用框架中常用的`Object.defineProperties`来巧妙地定义的。

    Object.defineProperties(Texture.prototype, {
        frame: {
            get: function ()
            {
                return this._frame; //Texture.frame 时获取到 _frame
            },
            set: function (frame)  //Texture.frame = Frame 时会执行，封装了setter操作
            {
                this._frame = frame; //对frame的读写其实都是对内部变量_frame的操作，这两个属性值等价，setter操作不等价。
                this.noFrame = false; //还要额外处理的属性之一
                this.width = frame.width; //额外处理的属性之二
                this.height = frame.height; //额外处理的属性之三
                if (!this.trim && !this.rotate && (frame.x + frame.width > this.baseTexture.width || frame.y + frame.height > this.baseTexture.height))
                {
                    throw new Error('Texture Error: frame does not fit inside the base Texture dimensions ' + this);
                }
                this.valid = frame && frame.width && frame.height && this.baseTexture.hasLoaded;
                if (this.trim)
                {
                    this.width = this.trim.width;
                    this.height = this.trim.height;
                    this._frame.width = this.trim.width;
                    this._frame.height = this.trim.height;
                }
                else
                {
                    this.crop = frame;
                }
                if (this.valid)
                {
                    this._updateUvs(); //额外执行的方法
                }
            }
        }
    });

我们什么时候需要这样来定义属性呢？

我们先想一个问题，当前这个Texture类有很多属性，但是很多是使用者不需要关注的，他们可能只是用来记录状态或缓存用的。

但是我们对一些关键属性的操作会影响到他们，如果不对关键属性做自定义setter处理，那么就需要开发者在执行赋值操作后人工修改那些辅助属性。这显然是不可思议的。

于是`Object.defineProperties`封装关键属性的setter操作，让赋值操作能做许多其他的额外处理。这是一个健壮的框架所必须的技能。


下面我们回到本节最初我们要看的RenderTexture类, 在我们对BaseTexture和Texture了解完毕之后,

终于到了渲染部分。

### RenderTarget = require('../renderers/webgl/utils/RenderTarget'),

这个类是用来webGL模式下处理最基础的绘制对象的，接受的参数中最重要的就是他的第一个参数：从canvas标签上通过getContext('webgl')方法获取到的webGL绘制对象。

他依赖一个`StencilMaskStack = require('./StencilMaskStack');`遮罩相关的数据结构。


所以，这个类中很多方法因为涉及webgl的api，所以对前端工程师来说相对比较生僻。

### FilterManager = require('../renderers/webgl/managers/FilterManager'),

这个类管理整个过滤器, 继承自WebGlManage。

他是webgl特有的。

### CanvasBuffer = require('../renderers/canvas/utils/CanvasBuffer'),

我们知道pixi在浏览器不支持webgl的时候会切换成canvas渲染，这里封装好了一个canvas渲染器。

内容比较少，我们可以一窥这个封装的过程。

    function CanvasBuffer(width, height)
    {
        this.canvas = document.createElement('canvas'); //新建一个canvas
        this.context = this.canvas.getContext('2d'); //获取其渲染对象
        this.canvas.width = width; 
        this.canvas.height = height;
    }
    CanvasBuffer.prototype.constructor = CanvasBuffer;
    module.exports = CanvasBuffer;
    Object.defineProperties(CanvasBuffer.prototype, {
        width: {
            get: function ()
            {
                return this.canvas.width;
            },
            set: function (val)
            {
                this.canvas.width = val;
            }
        },
        height: {
            get: function ()
            {
                return this.canvas.height;
            },
            set: function (val)
            {
                this.canvas.height = val;
            }
        }
    });
    CanvasBuffer.prototype.clear = function () //封装器的清除方法本质是对其内置的canvas内容的清除
    {
        this.context.setTransform(1, 0, 0, 1, 0, 0);
        this.context.clearRect(0,0, this.canvas.width, this.canvas.height);
    };
    CanvasBuffer.prototype.resize = function (width, height)
    {
        this.canvas.width = width;
        this.canvas.height = height;
    };
    CanvasBuffer.prototype.destroy = function () //封装器销毁方法只需把原来的canvas元素的引用丢失就行了，这些丢失引用的dom对象和普通的js对象之后都会被javascript的垃圾回收机制给处理掉，这就是封装器销毁的原理，即引用丢失。
    {
        this.context = null;
        this.canvas = null;
    };

可见，pixi里面的构造函数，也就是类，是非常多的。

不过有很多都是装饰者模式，把其他对象封装作为自己的一个属性，然后赋予其更多的属性和方法。

我们在研习框架源码的时候，一定要知道某个类的用法，确定我们操作的对象到底是谁。

比如上面这个canvasBuffer类。

使用这个类，本质还是在操作一个canvas的dom对象，但是却为你方便的封装了一系列简单操作。

比如context已经为你制定好了就是getContext('2d')，比如调用clear就会直接清空canvas这样的。

canvas为我们暴露出的api其实都是相当底层和原始的，pixi的这一层封装就像是研磨不平滑的平面，是很有必要的。


我们说完了RenderTexture的依赖，终于可以看看他本身了。

### RenderTexture = require('../textures/RenderTexture'),

A RenderTexture is a special texture that allows any Pixi display object to be rendered to it.

注释是这样介绍他的，RenderTexture是一个特殊的材质，允许任何Pixi的displayObject用它来进行渲染。

所有的材质都要被预加载，否则就会呈现出一个黑色矩形。

同时RenderTexture还会为displayobject提供一个无关位置和旋转角度的快照。我们有理由相信这个功能一定来自TextureUvs类。


function RenderTexture(renderer, width, height, scaleMode, resolution){...}

其接受五个参数：

* renderer {PIXI.CanvasRenderer|PIXI.WebGLRenderer} 渲染器
* [width=100] {number} 要渲染的宽度
* [height=100] {number} 要渲染的高度
* [scaleMode] {number} 要渲染的拉伸模式
* [resolution=1] {number} 分辨率

对于不同渲染模式下会有不同的材质渲染方式。

     if (this.renderer.type === CONST.RENDERER_TYPE.WEBGL)
        {
            var gl = this.renderer.gl; //webgl模式下我们textureBuffer是renderTarget实例。
            this.textureBuffer = new RenderTarget(gl, this.width, this.height, baseTexture.scaleMode, this.resolution);//, this.baseTexture.scaleMode);
            this.baseTexture._glTextures[gl.id] =  this.textureBuffer.texture;
            this.filterManager = new FilterManager(this.renderer); //当然webgl下才有过滤模式
            this.filterManager.onContextChange();
            this.filterManager.resize(width, height);
            this.render = this.renderWebGL;
            this.renderer.currentRenderer.start();
            this.renderer.currentRenderTarget.activate();
        }
        else //canvas模式下没有filterManager，textureBuffer源自canvasBuffer。
        {
            this.render = this.renderCanvas;
            this.textureBuffer = new CanvasBuffer(this.width* this.resolution, this.height* this.resolution); 
            this.baseTexture.source = this.textureBuffer.canvas;
        }

毫无疑问，webGl模式的渲染会更强大，相对的，源码内部的实现方式也复杂点，不过用户并不要关心webgl与canvas的切换。他们的切换是自动且封装完毕的。除非有指定渲染模式的特殊需求。

RenderTexture提供了一个获取渲染结果的方法。

他是由该类连续三个方法实现的。

    RenderTexture.prototype.getImage = function ()
    {
        var image = new Image();
        image.src = this.getBase64();
        return image; //返回一个可用于直接展示的image对象。
    };
    RenderTexture.prototype.getBase64 = function ()
    {
        return this.getCanvas().toDataURL(); //返回canvas图像转换为DataURL的结果
    };
    RenderTexture.prototype.getCanvas = function ()
    {
        if (this.renderer.type === CONST.RENDERER_TYPE.WEBGL)
        {
            var gl = this.renderer.gl;
            var width = this.textureBuffer.size.width;
            var height = this.textureBuffer.size.height;
            var webGLPixels = new Uint8Array(4 * width * height);
            gl.bindFramebuffer(gl.FRAMEBUFFER, this.textureBuffer.frameBuffer);
            gl.readPixels(0, 0, width, height, gl.RGBA, gl.UNSIGNED_BYTE, webGLPixels);
            gl.bindFramebuffer(gl.FRAMEBUFFER, null);
            var tempCanvas = new CanvasBuffer(width, height);
            var canvasData = tempCanvas.context.getImageData(0, 0, width, height);
            canvasData.data.set(webGLPixels);
            tempCanvas.context.putImageData(canvasData, 0, 0);
            return tempCanvas.canvas;  //将webGL内容用一个临时canvas保存来返回
        }
        else
        {
            return this.textureBuffer.canvas; //返回canvas对象
        }
    };

这个例子可以很好的诠释面向对象编程中方法“职责”的边界依据。

从一个只想获取iamge对象的开发者的角度，getImage还要依次执行getBase64和getCanvas方法，一个getImage方法拆分成三段简直蠢得可以。

但是假如其他开发者只想获取这个canvas的base64信息或只是canvas本身呢？

按照那个“聪明”的开发者想法，我们还得重写个包含getCanvas的getBase64方法和裸的getCanvas方法。

原本A+B+C式的定义方法，牺牲了一部分可读性和连贯性。但是换来了结构的清晰和三个耦合度极低的API。

如果变成ABC+BC+C的模式，只为了每个方法的流畅，却增加了大量的冗余代码。得不偿失。

而且，后者还完全没有考虑框架的扩展性，每次在调用链上增加一个方法，都是N（N是之前调用链长度）倍于前者的代码冗余！

所以说，这三个方法的定义很规范也很巧妙。

他们之前分工明确，功能清晰，每个都可以独立成一个API。从任何一个方法切入都不会有功能差错。

这就是一个健壮的库提供的API的标准。也是面向对象编程中将一个大的方法拆分成块的依据。


RenderTexture类还有获取全部像素和单个像素的两个原型方法。我们已经知道PIXI会有两套渲染模式，如果对webGL模式不熟悉，我们直接看稍微熟悉点的canvas方法吧，这也是种快速理解源码的办法。

    RenderTexture.prototype.getPixels = function ()
    {
        var width, height;
        if (this.renderer.type === CONST.RENDERER_TYPE.WEBGL)
        {
           ...
        }
        else
        {
            width = this.textureBuffer.canvas.width;
            height = this.textureBuffer.canvas.height;
            return this.textureBuffer.canvas.getContext('2d').getImageData(0, 0, width, height).data; //获取全部像素数组的数据
        }
    };
    RenderTexture.prototype.getPixel = function (x, y)
    {
        if (this.renderer.type === CONST.RENDERER_TYPE.WEBGL)
        {
          ...
        }
        else //否则获取指定X,Y位置的单个像素的数据
        {
            return this.textureBuffer.canvas.getContext('2d').getImageData(x, y, 1, 1).data;
        }
    };

至此，我们用很长的篇幅看完了材质类。关于真正用到它的DsiplayObject，sprite等重要方法只能留到下一章啦。