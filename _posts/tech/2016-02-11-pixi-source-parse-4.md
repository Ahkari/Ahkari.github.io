---
layout: post
title: PIXI v3源码解析 其四
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第四章
---

### Text: require('./text/Text'),
字体类可以创建一行或多行文字。创建自一个String和其样式style。就像下面这样使用：

` * var text = new PIXI.Text('This is a pixi text',{font : '24px Arial', fill : 0xff1010, align : 'center'});`

PIXI的text本身的一系列文本特有的如自动换行等特性都是自己实现的，我们挑选几个来看看Text和Sprite究竟有什么的不同。

来看wordWrap自动换行的实现方法：本质是文字的预处理，在可能导致换行的地方手动添加`\n`。
主要在用与判断当前text文本是否要换行的if判断和之后的操作很巧妙。(因为英文有空格的存在, 所以判断方式会更麻烦点。)

    Text.prototype.wordWrap = function (text)
    {
        var result = '';
        var lines = text.split('\n'); //原本文字中可能带\n换行的，我们需要分割出来一个个处理
        var wordWrapWidth = this._style.wordWrapWidth; //换行宽度
        for (var i = 0; i < lines.length; i++)
        {
            var spaceLeft = wordWrapWidth; //换行的宽度
            var words = lines[i].split(' '); //空格也视为可换行的标识
            for (var j = 0; j < words.length; j++)
            {
                var wordWidth = this.context.measureText(words[j]).width; //获取单个字的宽度
                var wordWidthWithSpace = wordWidth + this.context.measureText(' ').width; 
                if (j === 0 || wordWidthWithSpace > spaceLeft) //超过行宽
                {
                    if (j > 0)
                    {
                        result += '\n'; //在j不为0时，确认需要换行
                    }
                    result += words[j];
                    spaceLeft = wordWrapWidth - wordWidth; //如果换行或是第一个字符，下次判断的时候用于比较的宽度就从第一个字位置开始算！
                }
                else
                {
                    spaceLeft -= wordWidthWithSpace;
                    result += ' ' + words[j];
                }
            }
            if (i < lines.length-1)
            {
                result += '\n';
            }
        }
        return result;
    };

有个属性很有意思, 叫`dirty`。在width的getter函数上，我们发现`dirty=true`的时候我们需要先对他进行text更新。如果你有接触过数据绑定相关框架的知识，就应该隐约知道，这个`dirty`作用就是**脏检查**的判断标识。

    width: {
        get: function ()
        {
            if (this.dirty) //如果为true, 说明text的数据已更改过了, 也就是说数据"脏"了。
            {
                this.updateText(); //所以我们获取文本宽度前要先对他进行重绘。
            }
            return this.scale.x * this._texture._frame.width;
        },
        set: function (value)
        {
            this.scale.x = value / this._texture._frame.width;
            this._width = value;
        }
    },

那么就是说，初始的dirty是false的。只有影响其展现属性的数据发生变化的时候dirty才会true。

1. style属性的setter的最后`this.dirty = true`
2. text属性的setter的最后`this.text = true`

相对的, 更新完text后, 展现的和数据已一致, 那么dirty就会变成false, 表示数据已是最新而无需更新。

`Text.prototype.updateTexture = function (){}`的最后, `this.dirty = false;`

数据的脏检查是javascript框架数据绑定模块的一个重要概念。不同的框架其脏检查的方法也迥异。有空的时候可以把脏检查模块单独拿出来研究研究哈。


### Graphics: require('./graphics/Graphics'),
这里是对图形的处理。比如canvas的对图形的处理在这个js里面。
`CanvasGraphics = require('../renderers/canvas/utils/CanvasGraphics'),`
CanvasGraphics是一个普通对象，不是一个类，直接面向过程。
他下面的方法传递指定对象就可以使用。不需要实例化。

我们知道PIXI.SHAPES是一个对象, 含有各种图形和其对应的数字。看canvas是如何绘制不同的图像的。

    CanvasGraphics.renderGraphics = function (graphics, context)
    {
        var worldAlpha = graphics.worldAlpha;
        if (graphics.dirty)
        {
            this.updateGraphicsTint(graphics);
            graphics.dirty = false;
        }
        for (var i = 0; i < graphics.graphicsData.length; i++)
        {
            var data = graphics.graphicsData[i]; //图形的数据
            var shape = data.shape; 
            var fillColor = data._fillTint; //填充色
            var lineColor = data._lineTint; //描线色
            context.lineWidth = data.lineWidth; //描线宽度
            if (data.type === CONST.SHAPES.POLY) //如果是多边形
            {
                context.beginPath(); //开始路径
                var points = shape.points; //顶点数组
                context.moveTo(points[0], points[1]); //前两位是第一个顶点的XY值
                for (var j=1; j < points.length/2; j++)
                {
                    context.lineTo(points[j * 2], points[j * 2 + 1]); //移动到下一个顶点的XY位置上。
                }
                if (shape.closed) //如果图形需要封闭
                {
                    context.lineTo(points[0], points[1]); //最后绘制回起始点
                }
                if (points[0] === points[points.length-2] && points[1] === points[points.length-1])
                {
                    context.closePath(); //如果，第一个点和最后一个点是同一个点，直接封闭。
                }
                if (data.fill) //填充
                {
                    context.globalAlpha = data.fillAlpha * worldAlpha;
                    context.fillStyle = '#' + ('00000' + ( fillColor | 0).toString(16)).substr(-6);
                    context.fill();
                }
                if (data.lineWidth) //线宽
                {
                    context.globalAlpha = data.lineAlpha * worldAlpha;
                    context.strokeStyle = '#' + ('00000' + ( lineColor | 0).toString(16)).substr(-6);
                    context.stroke();
                }
            }
            else if (data.type === CONST.SHAPES.RECT) //如果是矩形
            {
                if (data.fillColor || data.fillColor === 0)
                {
                    context.globalAlpha = data.fillAlpha * worldAlpha;
                    context.fillStyle = '#' + ('00000' + ( fillColor | 0).toString(16)).substr(-6);
                    context.fillRect(shape.x, shape.y, shape.width, shape.height); //直接绘制矩形
                }
                if (data.lineWidth) //存在线宽
                {
                    context.globalAlpha = data.lineAlpha * worldAlpha;
                    context.strokeStyle = '#' + ('00000' + ( lineColor | 0).toString(16)).substr(-6);
                    context.strokeRect(shape.x, shape.y, shape.width, shape.height); //绘制线宽
                }
            }
            else if (data.type === CONST.SHAPES.CIRC) //绘制圆
            {
                context.beginPath();
                context.arc(shape.x, shape.y, shape.radius,0,2*Math.PI); //绘制圆
                context.closePath();
                if (data.fill)
                {
                    context.globalAlpha = data.fillAlpha * worldAlpha;
                    context.fillStyle = '#' + ('00000' + ( fillColor | 0).toString(16)).substr(-6);
                    context.fill();
                }
                if (data.lineWidth)
                {
                    context.globalAlpha = data.lineAlpha * worldAlpha;
                    context.strokeStyle = '#' + ('00000' + ( lineColor | 0).toString(16)).substr(-6);
                    context.stroke();
                }
            }
            else if (data.type === CONST.SHAPES.ELIP) //绘制椭圆
            {
                // ellipse code taken from: http://stackoverflow.com/questions/2172798/how-to-draw-an-oval-in-html5-canvas
                //canvas并不提供原生的绘制椭圆形的方法, PIXI的作者参考了stackoverflow这个问题。采用了把椭圆形分成四段贝赛尔曲线的方法来依次绘制。（这个答案中还提供了一个方法，就是绘制一个圆，然后把它按方向拉伸，不过这个方法会导致描线的失真所以未被采用）
                //可见，代码编写没有银弹，即使是一些众人皆知的类库也不可能有完美的解决方法，这里是汇集了其他社区的程序员的经验所探索得出的结果。这种集众人之所长的本领值得我们去学习。
                var w = shape.width * 2;
                var h = shape.height * 2;
                var x = shape.x - w/2;
                var y = shape.y - h/2;
                context.beginPath(); //开始绘制
                var kappa = 0.5522848,
                    ox = (w / 2) * kappa, // control point offset horizontal
                    oy = (h / 2) * kappa, // control point offset vertical
                    xe = x + w,           // x-end
                    ye = y + h,           // y-end
                    xm = x + w / 2,       // x-middle
                    ym = y + h / 2;       // y-middle
                context.moveTo(x, ym);
                context.bezierCurveTo(x, ym - oy, xm - ox, y, xm, y); //第一段
                context.bezierCurveTo(xm + ox, y, xe, ym - oy, xe, ym); //第二段
                context.bezierCurveTo(xe, ym + oy, xm + ox, ye, xm, ye); //第三段
                context.bezierCurveTo(xm - ox, ye, x, ym + oy, x, ym); //第四段
                context.closePath();
                if (data.fill)
                {
                    context.globalAlpha = data.fillAlpha * worldAlpha;
                    context.fillStyle = '#' + ('00000' + ( fillColor | 0).toString(16)).substr(-6);
                    context.fill();
                }
                if (data.lineWidth)
                {
                    context.globalAlpha = data.lineAlpha * worldAlpha;
                    context.strokeStyle = '#' + ('00000' + ( lineColor | 0).toString(16)).substr(-6);
                    context.stroke();
                }
            }
            else if (data.type === CONST.SHAPES.RREC) //绘制半圆
            {
                var rx = shape.x;
                var ry = shape.y;
                var width = shape.width;
                var height = shape.height;
                var radius = shape.radius;
                var maxRadius = Math.min(width, height) / 2 | 0;
                radius = radius > maxRadius ? maxRadius : radius;
                context.beginPath();
                context.moveTo(rx, ry + radius); //起始点
                context.lineTo(rx, ry + height - radius); 
                context.quadraticCurveTo(rx, ry + height, rx + radius, ry + height);
                context.lineTo(rx + width - radius, ry + height);
                context.quadraticCurveTo(rx + width, ry + height, rx + width, ry + height - radius);
                context.lineTo(rx + width, ry + radius);
                context.quadraticCurveTo(rx + width, ry, rx + width - radius, ry);
                context.lineTo(rx + radius, ry);
                context.quadraticCurveTo(rx, ry, rx, ry + radius);
                context.closePath();
                if (data.fillColor || data.fillColor === 0)
                {
                    context.globalAlpha = data.fillAlpha * worldAlpha;
                    context.fillStyle = '#' + ('00000' + ( fillColor | 0).toString(16)).substr(-6);
                    context.fill();
                }
                if (data.lineWidth)
                {
                    context.globalAlpha = data.lineAlpha * worldAlpha;
                    context.strokeStyle = '#' + ('00000' + ( lineColor | 0).toString(16)).substr(-6);
                    context.stroke();
                }
            }
        }
    };

接下来有个CanvasGraphics.renderGraphicsMask方法，用于绘制图形遮罩。这方法和上述图形绘制其实一样的，只不过少了填充颜色的步骤。

除了图形绘制，还依赖一个`GraphicsData = require('./GraphicsData'),`用于简单的存储图形数据。

回到`Graphics`类。

它同Sprite一样继承自Container类。也即是说，他是和Sprite一样能展现的物体。

如果说Sprite来自材质，那么Graphics就是来自几何。

所以他具有如下几何属性：

* fillAlpha 填充透明度
* lineWidth 线宽
* lineColor 描线颜色
* graphicsData 图形数据
* tint 填充色
* _prevTint 默认填充色
* blendMode 混合模式
* currentPath 当前路径
* _webGL webgl属性
* isMask 该几何形状是否用于遮罩
* boundsPadding 边界的内边距
* _localBounds 当前的边界
* dirty 数据是否已"脏"
* glDirty webgl渲染的东西是否已"脏", 即是否需要重新渲染
* boundsDirty 边界是否已"脏", 脏了就需要重新计算边界
*  cachedSpriteDirty 是否精灵对象的缓存已脏

他具有如下原型属性, 因为其也是个可展示可操作的对象, 所以原型方法理解起来很容易。

* clone() 完整克隆
* lineStyle() 设置路径的线的样式
* moveTo() 移动绘制点至某处
* lineTo() 移动到指定点并画下一条线
* quadraticCurveTo() 绘制二次曲线
* bezierCurveTo() 绘制贝塞尔曲线
* arcTo() 画圆
* arc() 画圆
* beiginFill() 填充
* endFill() 结束填充
* drawRect() 绘制矩形
* drawRoundRect() 绘制圆角矩形
* drawCircle() 绘制圆
* drawEllipse() 绘制椭圆
* drawPolygon() 绘制多边形
* clear() 清除
* generateTexture() 材质同步
* _renderWebGL() ...
* _renderCanvas() ...
* getBounds() 获取边界
* containsPoint() 指定点是否在其中
* updateLocalBounds() 更新边界, 然而和sprite一样, 他们的边界本质上还是一个矩形范围, 所以不能用于碰撞检测
* drawShape() ...
* destory() 销毁该类

### GraphicsRenderer: require('./graphics/webgl/GraphicsRenderer'),
PIXI里面webgl的渲染都是单独拿出来做一个类来解释的。

比如这里，我们用canvasAPI直接实现的形状绘制需要我们用webgl来重新实现一遍。

这里我们要依次对几个形状进行build。
再build多边形的时候我们用到了一个叫earcut的库。他是最快最轻量的js操作webgl多边形的库。
[Earcut-webgl操作多变形](https://github.com/mapbox/earcut)

接下来的依赖我们之前都一个个解析过了：

    // textures 材质类
    Texture:                require('./textures/Texture'),
    BaseTexture:            require('./textures/BaseTexture'),
    RenderTexture:          require('./textures/RenderTexture'),
    VideoBaseTexture:       require('./textures/VideoBaseTexture'),
    TextureUvs:             require('./textures/TextureUvs'),
    // renderers - canvas canvas渲染器
    CanvasRenderer:         require('./renderers/canvas/CanvasRenderer'),
    CanvasGraphics:         require('./renderers/canvas/utils/CanvasGraphics'),
    CanvasBuffer:           require('./renderers/canvas/utils/CanvasBuffer'),
    // renderers - webgl webgl渲染器
    WebGLRenderer:          require('./renderers/webgl/WebGLRenderer'),
    ShaderManager:          require('./renderers/webgl/managers/ShaderManager'),
    Shader:                 require('./renderers/webgl/shaders/Shader'),
    ObjectRenderer:         require('./renderers/webgl/utils/ObjectRenderer'),
    RenderTarget:           require('./renderers/webgl/utils/RenderTarget'),
    // filters - webgl webgl特有的滤镜类
    AbstractFilter:         require('./renderers/webgl/filters/AbstractFilter'),
    FXAAFilter:             require('./renderers/webgl/filters/FXAAFilter'),
    SpriteMaskFilter:       require('./renderers/webgl/filters/SpriteMaskFilter'),
    
回到第二章开头，我们一直在看core类。他的最后一个方法`autoDetectRenderer`用于主动切换渲染方式。

    autoDetectRenderer: function (width, height, options, noWebGL)
    {
        width = width || 800;
        height = height || 600;
        if (!noWebGL && core.utils.isWebGLSupported())
        {
            return new core.WebGLRenderer(width, height, options);
        }
        return new core.CanvasRenderer(width, height, options);
    }
    
至此核心类core已介绍完，这个类直接暴露出最有用的几个对象分别是:

1. `Container` 可以包含元素的最基础的类
2. `Sprite` 可以展示图片的精灵类
3. `Graphic` 可以展示形状的类, 和Sprite基本同级
4. `Text` 文本类, 用于展示文本
5. `CanvasRenderer` Canvas渲染器
6. `WebGLRenderer` webGL渲染器, 附带各种材质的管理器
7. 以及其他辅助类

可见core核心为我们提供了绘制图像的几乎所有的方法。并为我们准备了两套渲染引擎。

我们所理解的PIXIjs本质就是一个渲染引擎，为我们提供的可操作的物体最高也只到了Sprite这样的自由度。

底层基于PIXI，封装一层逻辑操作层让PIXI功能更为强大甚至做出酷炫的游戏。有这样的框架，最有名的应该就是phaser。而我司一直干的也是和phaser一样的事。

下一节，我们会分析除了core以外的辅助类。