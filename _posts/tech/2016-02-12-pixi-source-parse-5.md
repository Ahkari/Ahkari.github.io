---
layout: post
title: PIXI v3源码解析 其五
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第五章
---

### core.extras = require('./extras');
为PIXI增加extras子模块。

他所暴露出的三个内部对象是：

    module.exports = {
        MovieClip:      require('./MovieClip'),
        TilingSprite:   require('./TilingSprite'),
        BitmapText:     require('./BitmapText')
    };

让我们从依赖开始分析。

### require('./cacheAsBitmap');

采用类增强的方法。

引入DisplayObject并为其增加了一个主属性：
`cacheAsBitmap`, 从其defineProperties来看, 如果设为true, 我们就会把displayObject缓存为一个bitmap。
这样能提升复杂的静态物体渲染的性能。

### require('./getChildByName');

为Container增加getChildByName方法：

    core.DisplayObject.prototype.name = null;
    core.Container.prototype.getChildByName = function (name)
    {
        for (var i = 0; i < this.children.length; i++)
        {
            if (this.children[i].name === name)
            {
                return this.children[i];
            }
        }
        return null;
    };

这是一个新增的方法，用于直接通过对象的name来获取指定子元素。

为什么不直接在Container类定义的地方把这个方法和其属性写进去呢？我觉得应该有两个原因：

1. 按照Container的设计理念，里面的子元素并没有名字属性`name`这一概念的，因为Child一直都是用index来指向的。现在有需求希望能通过name来索引，但只有在搜寻container的子元素的时候才会需要name属性。这个name的应用范围就小的可怜。所以我们希望能和原本的属性方法作区分。
2. 这个通过name索引的需求一定是后来才出现的，为了和原来的Container区分，也就是为了表明这个方法是附加的，需要单独定义。

所以PIXI决定，用最小的代码代价给Container增加一个通过名字搜索子元素的方法。同时还需要给displayObject加一个最基础的原型name。

就有了上述代码单独定义的代码。

维护大型项目的时候，如何在维护既有代码的稳定性基础上实现功能扩充也是门学问。

### require('./getGlobalPosition');
DisplayObject新增方法`core.DisplayObject.prototype.getGlobalPosition`
用户获取相对于canavs的绝对定位。

### MovieClip: require('./MovieClip'),
逐帧动画功能。使用方法如下：

    var alienImages = ["image_sequence_01.png","image_sequence_02.png","image_sequence_03.png","image_sequence_04.png"];
    var textureArray = [];
    for (var i=0; i < 4; i++)
    {
        var texture = PIXI.Texture.fromImage(alienImages[i]);
        textureArray.push(texture);
    };
    var mc = new PIXI.MovieClip(textureArray);
    
这是一个功能完整的用于显示逐帧动画的API。相应的stop，play方法以及设计ticker时钟的都很有意思。
不再赘述。

### TilingSprite: require('./TilingSprite'),
纹理精灵，在Sprite基础上能力更专一。

可以大面积重复渲染。

### BitmapText: require('./BitmapText')
这个类能通过自定义的字体文件来生成文字，生成文字的前提是font已加载完毕。

    // in this case the font is in a file called 'desyrel.fnt'
    var bitmapText = new PIXI.extras.BitmapText("text using a fancy font!", {font: "35px Desyrel", align: "right"});
    // 这个例子里font就会指向Desyrel这个字体。
    
### core.filters = require('./filters');
真丶滤镜类。集合了所有的具体的滤镜实现方法。其依赖的滤镜有这么多。至于每个是啥功能就需要一点点去看了（虽然从英文名上差不多就能认出来了。）

    module.exports = {
        AsciiFilter:        require('./ascii/AsciiFilter'),
        BloomFilter:        require('./bloom/BloomFilter'),
        BlurFilter:         require('./blur/BlurFilter'),
        BlurXFilter:        require('./blur/BlurXFilter'),
        BlurYFilter:        require('./blur/BlurYFilter'),
        BlurDirFilter:      require('./blur/BlurDirFilter'),
        ColorMatrixFilter:  require('./color/ColorMatrixFilter'),
        ColorStepFilter:    require('./color/ColorStepFilter'),
        ConvolutionFilter:  require('./convolution/ConvolutionFilter'),
        CrossHatchFilter:   require('./crosshatch/CrossHatchFilter'),
        DisplacementFilter: require('./displacement/DisplacementFilter'),
        DotScreenFilter:    require('./dot/DotScreenFilter'),
        GrayFilter:         require('./gray/GrayFilter'),
        DropShadowFilter:   require('./dropshadow/DropShadowFilter'),
        InvertFilter:       require('./invert/InvertFilter'),
        NoiseFilter:        require('./noise/NoiseFilter'),
        PixelateFilter:     require('./pixelate/PixelateFilter'),
        RGBSplitFilter:     require('./rgb/RGBSplitFilter'),
        ShockwaveFilter:    require('./shockwave/ShockwaveFilter'),
        SepiaFilter:        require('./sepia/SepiaFilter'),
        SmartBlurFilter:    require('./blur/SmartBlurFilter'),
        TiltShiftFilter:    require('./tiltshift/TiltShiftFilter'),
        TiltShiftXFilter:   require('./tiltshift/TiltShiftXFilter'),
        TiltShiftYFilter:   require('./tiltshift/TiltShiftYFilter'),
        TwistFilter:        require('./twist/TwistFilter')
    };
    
每个滤镜都有对应的js处理和webgl程序驱动。我们不深入了。

### core.interaction = require('./interaction');
这个模块很重要，涉及输入输出的交互。

PIXI位处于浏览器的环境之下，所以一定会有和鼠标键盘的交互。这个模块就是把PIXI之前渲染出的Sprite等可展示对象和浏览器交互事件结合起来。

### InteractionData: require('./InteractionData'),
这个类把基础的事件封装一层。

其下属性`originalEvent`是原生事件。


### InteractionManager: require('./InteractionManager'),
他需要一些事件状态标记，用一个对象统一管理。

    var interactiveTarget = {
        interactive: false,
        buttonMode: false,
        interactiveChildren: true,
        defaultCursor: 'pointer',
        _over: false,
        _touchDown: false
    };
    module.exports = interactiveTarget;
    
并全部用于增强DsiplayObject。因为涉及全局设置，所以写进原型里。

    Object.assign(
        core.DisplayObject.prototype,
        require('./interactiveTarget')
    );

为canvas增加事件监听：

    InteractionManager.prototype.addEvents = function ()
    {
        if (!this.interactionDOMElement)
        {
            return;
        }
        core.ticker.shared.add(this.update, this);
        if (window.navigator.msPointerEnabled)
        {
            this.interactionDOMElement.style['-ms-content-zooming'] = 'none';
            this.interactionDOMElement.style['-ms-touch-action'] = 'none';
        }
        window.document.addEventListener('mousemove',    this.onMouseMove, true);
        this.interactionDOMElement.addEventListener('mousedown',    this.onMouseDown, true);
        this.interactionDOMElement.addEventListener('mouseout',     this.onMouseOut, true);
        this.interactionDOMElement.addEventListener('touchstart',   this.onTouchStart, true);
        this.interactionDOMElement.addEventListener('touchend',     this.onTouchEnd, true);
        this.interactionDOMElement.addEventListener('touchmove',    this.onTouchMove, true);
        window.addEventListener('mouseup',  this.onMouseUp, true);
        this.eventsAdded = true;
    };

假设我们触发了onMouseDown：

    InteractionManager.prototype.onMouseDown = function (event)
    {
        this.mouse.originalEvent = event; //封装事件
        this.eventData.data = this.mouse; 
        this.eventData.stopped = false;
        this.mapPositionToPoint( this.mouse.global, event.clientX, event.clientY); //定位！
        if (this.autoPreventDefault)
        {
            this.mouse.originalEvent.preventDefault();
        }
        this.processInteractive(this.mouse.global, this.renderer._lastObjectRendered, this.processMouseDown, true );
    };
    InteractionManager.prototype.mapPositionToPoint = function ( point, x, y ) //用到的定位方法
    {
        var rect = this.interactionDOMElement.getBoundingClientRect();
        point.x = ( ( x - rect.left ) * (this.interactionDOMElement.width  / rect.width  ) ) / this.resolution;
        point.y = ( ( y - rect.top  ) * (this.interactionDOMElement.height / rect.height ) ) / this.resolution;
    };

然后确定事件相关的所有参数后，进入事件的触发，这个方法涉及递归和许多if判断，所以最好在事件实际触发的时候debug来看比较容易懂：

    InteractionManager.prototype.processInteractive = function (point, displayObject, func, hitTest, interactive )
    {
        if(!displayObject || !displayObject.visible)
        {
            return false;
        }
        var children = displayObject.children;
        var hit = false;
        interactive = interactive || displayObject.interactive; 
        if(displayObject.interactiveChildren)
        {
            for (var i = children.length-1; i >= 0; i--)
            {
                if(! hit  && hitTest)
                {
                    hit = this.processInteractive(point, children[i], func, true, interactive ); //如果子元素有点击区域在其之上的hit就变成true。
                }
                else
                {
                    this.processInteractive(point, children[i], func, false, false );
                }
            }
        }
        if(interactive)
        {
            if(hitTest)
            {
                if(displayObject.hitArea)
                {
                    displayObject.worldTransform.applyInverse(point,  this._tempPoint);
                    hit = displayObject.hitArea.contains( this._tempPoint.x, this._tempPoint.y );//判断是否在触发区域内
                }
                else if(displayObject.containsPoint)
                {
                    hit = displayObject.containsPoint(point);//是否触发点是否在其内容区域上
                }
            }
            if(displayObject.interactive) 
            {
                func(displayObject, hit); //触发绑定的事件
            }
        }
        return hit;
    };

### core.loaders = require('./loaders');

`bitmapFontParser:require('./bitmapFontParser'),`是用来解析bitmapfont字体文件的工具模块。这个模块的架构相对于PIXI整体来说比较随意。注释风格也和总体框架不一致，目测和主编写者不是一个人233。

`spritesheetParser:require('./spritesheetParser'),`用来加载精灵动画。即有很多帧聚集在一张大图上的动画。

`textureParser:require('./textureParser'),`异步加载材质

`Resource:require('resource-loader').Resource`github地址[Resource-loader](https://github.com/englercj/resource-loader), 是用这个库拓展而来的。专为web游戏而生的资源加载器。

### core.mesh = require('./mesh');
其依赖关系如下:

    module.exports = {
        Mesh:           require('./Mesh'),
        Rope:           require('./Rope'),
        MeshRenderer:   require('./webgl/MeshRenderer'),
        MeshShader:     require('./webgl/MeshShader')
    };

用于实现网状材质和线性材质。可以实现一些扭曲的效果。

至此PIXI v3的源码已解析完毕。

只从源码的角度来看这个框架是十分晦涩难懂的。这也是我用了整整一个星期来看的原因。

在结束源码的阅读后，我们需要去熟悉PIXI的API，但是PIXI的API本质上和源码注释的内容是一样的。

所以我建议直接看demo。

[PIXI最容易懂得在线demo](http://pixijs.github.io/examples/)

遇到不是很理解的部分再回去查看源码，等demo全部弄懂后基本就完全掌握PIXI啦。

但是，需要认识到的是PIXI离一个真正可以使用的HTML5游戏框架还是有一段距离的。PIXI的核心是渲染。而一个游戏除了渲染还需要逻辑。这也是基于PIXI之上还有开源游戏框架phaser等的原因。

不过PIXI的代码组织非常漂亮，模块化程度极高，面向对象编程，是严谨的传统OO风格。

看了这么久，没点收益，我自己都不信呐。

