---
layout: post
title: PIXI v3源码解析 其五
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第五章
---

### core.extras = require('./extras');
为PIXI增加extras子命名空间。

他所暴露出的三个内部对象是：

    module.exports = {
        MovieClip:      require('./MovieClip'),
        TilingSprite:   require('./TilingSprite'),
        BitmapText:     require('./BitmapText')
    };

让我们从依赖开始分析。

### require('./cacheAsBitmap');

采用对象增强模式。

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

这是一个新增的方法，为什么不直接在Container的属性里面增加呢？因为按照Container的设计理念，里面的子元素并没有名字属性`name`这一说。

Child一直都是用index来指向的。

所以如果我们想增加一个getChildByName的方法。如果为了整体框架的统一，需要从displayObject开始为所有可展示对象增加其name属性。

但是现实是，只有在搜寻container的子元素的时候才会需要name索引。所以这个name的应用范围就变得小的可怜。

最后，PIXI决定，用最小的代码代价给Container增加一个通过名字搜索子元素的方法。就有了上述代码：name不是必须属性，并和用到它的原型方法共存。

这就是库用于拓展功能时的一种方法。维护大型系统的时候，既有系统的稳定也是需要头等考虑的事。

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

