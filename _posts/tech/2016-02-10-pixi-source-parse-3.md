---
layout: post
title: PIXI v3源码解析 其三
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第三章
---

同样的开头，不过我们这次真的来看DisplayObject类了！

### DisplayObject: require('./display/DisplayObject'),
我们研究完了相对抽象的材质类们，DisplayObject类终于以一种“看得见摸得着”的可描述二维物体状态站在了我们面前。

一个DisplayObject是PIXI中用来展示的最基础的实体对象，基础到直接继承自EventEmitter类。

但是他却是一个抽象类，作为所有待展示对象的基类。他提供了很多待复写的方法，这些方法本身并不能起到预期的作用，只有被继承之后复写了才有意义。

他有如下属性：

* position 位置
* scale 伸缩
* pivot 中心点
* rotation 旋转角度
* alpha 透明度
* visible 是否可见
* renderable 是否可渲染, 和visible区别只在updateTransform方法仍然会调用
* parent 重要, 这个元素的父元素
* worldAlpha 整体的透明度, 即当前透明度乘以这个值才是实际需要展示的透明度
* worldTransform 同理, 应用在其上的外界的矩阵效果
* filterArea 需要引用滤镜的区域
* _sr 用来缓存sin.rotation计算结果, 频繁的数学计算对性能是有影响的
* _cr 同理
* _bounds 缓存这个物体的边界
* _currentBounds 获取最新的物体边界
* _mask 缓存物体的遮罩

很眼熟的二维属性, 除此之外还有defineProperties方式定义的稍微高级点的属性 。

* x 其实就是position.x
* y 同理
* worldVisible 很有趣的属性, 我们从其getter方法了解到, 如果他自身或任何一个父元素不可见, 那他就是false, 否则就是true。仅凭这点信息我们就可以猜测出PIXI是如何为物体扩展出一个visible属性了。

即每个元素自身有个visible和worldVisible属性，两个任何一个为false时，PIXI在渲染就会忽略他。同时也宣告了visible的所有子元素都不可见，因为他们的worldVisible都会是false。

        worldVisible: {
            get: function ()
            {
                var item = this;
                do {
                    if (!item.visible)
                    {
                        return false;
                    }
                    item = item.parent;
                } while (item);
                return true;
            }
        },

* mask 遮罩
* filter 滤镜

然后是原型方法，大部分是对上述二维属性的操作。需要注意的几个方法如下：

* DisplayObject.prototype.updateTransform 用于更新当前元素变换数值
一句话描述就是：计算父元素的worldTransform对当前元素的影响，更新当前元素的worldTransform。（顺便把透明度也重新计算了）。
* DisplayObject.prototype.getBounds 用来获取元素的展示边界

        DisplayObject.prototype.getBounds = function (matrix) // jshint unused:false
        {
            return math.Rectangle.EMPTY; //返回空的矩形距离
        };

    这个原型方法是用来获取元素展示的边界的，可是却意外的返回一个定值。
没错，如开始所说，displayObject是一个抽象类，他本身不能用于展示物体，他等待着被子类继承重写内部方法。

如果是java的话，会很清晰明确的告知这就是个抽象类，而且会在尝试实例化的时候报错。

可惜javascript语言天生和传统OO编程分道扬镳，都是靠着一代代开发者的自觉来强制遵守OO编程的规矩的。这里的displayObject自然可以被new实例化，不过使用时一定会有不按预期的情况发生。

比如调用getBound就始终不会有预期结果。js自然也不会报错，只能靠使用者遵循API使用规范才可以得到预想结果。这也是动态语言的天然短板之一。
* DisplayObject.prototype.renderWebGL与DisplayObject.prototype.renderCanvas这两个方法源码里已经大摇大摆的表明自己就是要被子类重写的了。。。

        DisplayObject.prototype.renderWebGL = function (renderer) // jshint unused:false
        {
            // OVERWRITE;
        };

* DisplayObject.prototype.setParent 从源码我们又看出了端倪。
如果我们要把给一个对象设置父元素，需要这个父元素存在且有addChild方法。难道只是有个addChild方法就能添加子元素了吗，理论上是的，但是从Error的内容来看，这个父元素又必须是一个`Container`。

我们就可以推测出：PIXI必然有专门用来包裹子元素的Container容器类，这里没有用intansceOf来判断父元素实例化自哪里，说明有多个子类继承自Container，同时也避免了我们子类继承Container时报错。这里通过判断addChild方法的是否存在这种这种弱检测，缓和了判断条件也提高了框架的容错性。当然，代价是严谨度。

        DisplayObject.prototype.setParent = function (container)
        {
            if (!container || !container.addChild)
            {
                throw new Error('setParent: Argument must be a Container');
            }
            container.addChild(this);
            return container; //返回的是容器父元素
        };


真是说曹操曹操到，我们刚预测了Container类的存在，core接下来就用到了。

### Container: require('./display/Container'),

Container是PIXI中非常重要的一个概念, 我们在第一章中阅读旧版兼容时了解到了PIXI.Stage已经被Container取代, 而DisplayObjectContainer直接更名为Container。

也就是说，PIXI v3将概念都统一了，原本有一个用于展示所有内容的Stage，现在也概括为一个“包含所有内容的容器”。

他继承自DsiplayObject，在具有一些二维物体属性的基础上多了一个：
`this.children = [];`属性。用来存储自己的子元素。

对于child的操作也成了我们关注的重点，比如下面的原型方法：

    Container.prototype.addChild = function (child)
    {
        return this.addChildAt(child, this.children.length); //这个方法默认添加到子元素数组的末尾
    };
    Container.prototype.addChildAt = function (child, index)
    {
        if (child === this)
        {
            return child; //child是自身，不进行操作
        }
        if (index >= 0 && index <= this.children.length)
        {
            if (child.parent)
            {
                child.parent.removeChild(child); //如果目标子元素含有父元素，先需要进行移除操作
            }
            child.parent = this;
            this.children.splice(index, 0, child); //目标元素插入进来
            this.onChildrenChange(index); //触发ChildrenChange
            child.emit('added', this); //触发容器的added事件
            return child;
        }
        else //否则。提示index越界
        {
            throw new Error(child + 'addChildAt: The index '+ index +' supplied is out of bounds ' + this.children.length);
        }
    };

简单易懂，不拖泥带水。

对一个最简答的容器来说，这个add方法绰绰有余且360度全方位无死角。

but，这并不能代表所有的容器这个add方法都适用。

假设后期我们需要增加一个能切换标签页的容器，这个容器具有N个标签，每个标签中都能放东西，点击哪个标签，就会显示哪个标签里的内容。

我们继承自Container之后，要如何构造这个Tabcontainer的其他属性方法呢？

动用面向对象思维，即，目标是什么样子的，我们就同样的方法来描述他。

这个标签页容器具有标签，我们给容器增加一个this.tabs数组来管理。

同时child属性里的每个对象都需要有能指明自己所属标签的属性，这一步就很多方法了。我们至少能有以下形式的child数据存储结构.

1. [ { tabname : 'tab1' ， objName : ... } , { tabname : 'tab2' , objName : ... } , ... ] 
2. [ tab1 : [ {} , {} , {} ] , tab2 : [ {} , {} ,{} ] , ... ]

等等.

采用哪种方式是根据数据风格来定的。

在此之后我们需要重写addChild方法，现在不仅需要index，还要指定所属标签。

总之到这一步后，子类的功能和实现方式就已经完全受编码者意愿控制了。

重写父类方法是面向对象继承模式中重要一环, 我们知道, displayObject的getBounds并没有给出计算边界的具体方法。我们看看Container的getBounds的实现方法。

    Container.prototype.getBounds = function ()
    {
        if(!this._currentBounds) //存在缓存，就不需要重新计算了。
        {
            if (this.children.length === 0)
            {
                return math.Rectangle.EMPTY; //为空
            }
            var minX = Infinity; //表示正无穷的数，可随时覆盖。
            var minY = Infinity;
            var maxX = -Infinity;
            var maxY = -Infinity;
            var childBounds;
            var childMaxX;
            var childMaxY;
            var hildVisible = false;
            for (var i = 0, j = this.children.length; i < j; ++i)
            {
                var child = this.children[i];
                if (!child.visible)
                {
                    continue;
                }
                childVisible = true;
                childBounds = this.children[i].getBounds(); //获取子元素的边界值
                minX = minX < childBounds.x ? minX : childBounds.x; //最小X
                minY = minY < childBounds.y ? minY : childBounds.y; //最小Y
                childMaxX = childBounds.width + childBounds.x;
                childMaxY = childBounds.height + childBounds.y;
                maxX = maxX > childMaxX ? maxX : childMaxX; //最大X
                maxY = maxY > childMaxY ? maxY : childMaxY; //最大Y
            }
            if (!childVisible)
            {
                return math.Rectangle.EMPTY; //虽然计算了，但是如果visible是false还是得返回空
            }
            var bounds = this._bounds;
            bounds.x = minX;
            bounds.y = minY;
            bounds.width = maxX - minX;
            bounds.height = maxY - minY;
            this._currentBounds = bounds;
        }
        return this._currentBounds; //返回的是这样一个玩意，“最左上角的坐标”和总共的宽高。
    };
    
可见，容器的实际边界其实是由其内部的子元素的位置宽高决定的。而并不是本身自定义的。

displayObject中还有个抽象方法是renderWebgl与renderCanvas。在这些方法中是这样使用他的。

    if (this._filters && this._filters.length)
    {
        renderer.filterManager.pushFilter(this, this._filters); //正确使用滤镜的方法
    }
    if (this._mask)
    {
        renderer.maskManager.pushMask(this, this._mask); //正确使用着找的方法
    }

回溯之前的代码可是阅读源码重要的一步，我们看之前的滤镜和遮罩管理的类。

`FilterManager.prototype.pushFilter = function (target, filters){}`

接受两个参数，一个是应用滤镜的对象，一个是滤镜名。这里将滤镜效果加进渲染器里之后，renderWebgl就会在这个渲染效果下renderer.currentRenderer.start();

开启渲染后，我们调用其下的所有子元素的渲染方法，让他们按照设定好的渲染规则展示出来。

一次循环完毕后，清空状态，我们至此就完成了递归渲染(假设其下还有容器的话)。

    renderer.currentRenderer.flush();
    if (this._mask)
    {
        renderer.maskManager.popMask(this, this._mask);
    }
    if (this._filters)
    {
        renderer.filterManager.popFilter();
    }
    renderer.currentRenderer.start();
    
总结下Container，他是个用于包含子元素的容器，本身没有什么可渲染的效果。他的新特性主要都在子元素操作上。

由于PIXI的Stage舞台概念已经被根Container容器所取代，所以说Container当之无愧是当前PIXI最强大的基础类了。（整个舞台都是通过他渲染的，能不强大？）

下面我们看精灵Sprite类，在此之前我们先看看他涉及的一个普通对象。CanvasTinter。

### CanvasTinter = require('../renderers/canvas/utils/CanvasTinter'),
为什么说他是个普通对象呢。

因为他不需要构造函数，所以也就不是个类，只是个工具方法的集合而已。

用处是操作canvas的着色或材质。

提供了如下几个方法：
* CanvasTinter.getTintedTexture 获取目标着色材质
* CanvasTinter.tintWithMultiply 用复合模式着色
* CanvasTinter.tintWithOverlay 用覆盖模式着色
* CanvasTinter.tintWithPerPixel 用逐像素模式着色
...

### Sprite: require('./sprites/Sprite'),

Container.call(this); 夭寿啦，Sprite继承自Container！

也就是说PIXI里面的精灵都可以有子元素，并不一定是个狭义上的只能包含物体的容器。

到Sprite这一步时，PIXI终于有一个确实能展示自身的类了。

拿宽高这个属性来说：我们知道Sprite继承自Container，Container继承自DisplayObject

DisplayObject根本没有width和height属性，它只有二维变换透明度拉伸滤镜遮罩等用于展现材质的属性。

Container的宽高通过definedProperty定义，是其中所有全部子元素内容所占的宽高，它只在父类上新增了一个“含有子元素”这个特征，他本身就像是一具空壳。

而Sprite的宽高`return Math.abs(this.scale.x) * this.texture._frame.width;`是确确实实的当前帧的宽高，这时，我们才有了个真正意义上的可以呈现的对象。

另一个Sprite终于是“可展现”的对象的根据是他有个判断某二维点是否在其内的方法。

    Sprite.prototype.containsPoint = function( point )
    {
        this.worldTransform.applyInverse(point,  tempPoint);
        var width = this._texture._frame.width;
        var height = this._texture._frame.height;
        var x1 = -width * this.anchor.x;
        var y1;
        if ( tempPoint.x > x1 && tempPoint.x < x1 + width )
        {
            y1 = -height * this.anchor.y;
            if ( tempPoint.y > y1 && tempPoint.y < y1 + height )
            {
                return true;
            }
        }
        return false;
    };

因为他可见，所以我们才会有判断有某点在其之内的必要，也就是研究其“物理特性”的必要。

我们还发现，Sprite半重写了destory方法。

    Sprite.prototype.destroy = function (destroyTexture, destroyBaseTexture)
    {
        Container.prototype.destroy.call(this); //先调用父类同名方法
        this.anchor = null;
        if (destroyTexture) //方法增强
        {
            this._texture.destroy(destroyBaseTexture);
        }
        this._texture = null;
        this.shader = null;
    };

即使是覆写父类方法，我们当然也是怎么偷懒怎么来。call父类同名方法自然是节省代码的最简单方法了。

也许有人会对PIXI里面的那些同名方法感到迷惑，为什么没有继承关系的两个类也可能有功能相近的方法呢？

其实很可能他们其实调用的都是同一个东西：

    Sprite.fromImage = function (imageId, crossorigin, scaleMode)
    {
        return new Sprite(Texture.fromImage(imageId, crossorigin, scaleMode));
    };

Sprite可以直接从图片生成，因为这个方法已经悄悄为你新建了一个可应用的材质。


### ParticleContainer: require('./particles/ParticleContainer'),

一个因垂丝汀的类，粒子容器。

从其自身的介绍就能了解其特性了：

* 功能明确且唯一，用于生成大量的Sprite或粒子。
* 所以其上并不能应用一些高级方法，受容器本身限制，他们将无效化
* 他只能有位置，拉伸和旋转角度的变化

示例代码已经说明一切：

    var container = new ParticleContainer();
    for (var i = 0; i < 100; ++i)
    {
        var sprite = new PIXI.Sprite.fromImage("myImage.png");
        container.addChild(sprite);
    }

这个容器和普通容器的使用没什么区别, 但是我们会向里面置入大量的粒子图片。拿到这些粒子图片的处理都在粒子容器渲染方法里有特殊的处理。

webgl实现的方式我们稍后看。下面是canvas的实现方法，作为粒子容器，他直接接管了子元素的渲染，并不会递归调用子元素各自的渲染方法。这就是粒子容器为什么是粒子容器的特殊之处。

    ParticleContainer.prototype.renderCanvas = function (renderer)
    {
        if (!this.visible || this.worldAlpha <= 0 || !this.children.length || !this.renderable)
        {
            return;
        }
        var context = renderer.context;
        var transform = this.worldTransform;
        var isRotated = true;
        var positionX = 0;
        var positionY = 0;
        var finalWidth = 0;
        var finalHeight = 0;
        context.globalAlpha = this.worldAlpha;
        this.displayObjectUpdateTransform();
        for (var i = 0; i < this.children.length; ++i) //直接接管子元素的渲染方式
        {
            var child = this.children[i];
            if (!child.visible)
            {
                continue;
            }
            var frame = child.texture.frame;
            context.globalAlpha = this.worldAlpha * child.alpha;
            if (child.rotation % (Math.PI * 2) === 0) //如果被2PI整除，即没有旋转
            {
                if (isRotated) //是否旋转了
                {
                    context.setTransform(
                    transform.a,
                    transform.b,
                    transform.c,
                    transform.d,
                    transform.tx,
                    transform.ty
                );
                isRotated = false; //矩阵变换完毕，设为未旋转的状态
            }
            positionX = ((child.anchor.x) * (-frame.width * child.scale.x) + child.position.x  + 0.5);
            positionY = ((child.anchor.y) * (-frame.height * child.scale.y) + child.position.y  + 0.5);
            finalWidth = frame.width * child.scale.x;
            finalHeight = frame.height * child.scale.y;
        }
        else
        {
            if (!isRotated)
            {
                isRotated = true;
            }
            child.displayObjectUpdateTransform();
            var childTransform = child.worldTransform;
            if (renderer.roundPixels)
            {
                context.setTransform(
                    childTransform.a,
                    childTransform.b,
                    childTransform.c,
                    childTransform.d,
                    childTransform.tx | 0,
                    childTransform.ty | 0
                );
            }
            else
            {
                context.setTransform(
                    childTransform.a,
                    childTransform.b,
                    childTransform.c,
                    childTransform.d,
                    childTransform.tx,
                    childTransform.ty
                );
            }
            positionX = ((child.anchor.x) * (-frame.width) + 0.5);
            positionY = ((child.anchor.y) * (-frame.height) + 0.5);
            finalWidth = frame.width;
            finalHeight = frame.height;
        }
        context.drawImage(
            child.texture.baseTexture.source,
            frame.x,
            frame.y,
            frame.width,
            frame.height,
            positionX,
            positionY,
            finalWidth,
            finalHeight
        );
    }

下面来看webgl渲染方式，他是一个“插件式”调用的过程。

    ParticleContainer.prototype.renderWebGL = function (renderer)
    {
        if (!this.visible || this.worldAlpha <= 0 || !this.children.length || !this.renderable)
        {
            return;
        }
        renderer.setObjectRenderer( renderer.plugins.particle ); //设置渲染方式
        renderer.plugins.particle.render( this ); //执行渲染
    };

我们知道这个renderer是一个`PIXI.WebGLRenderer`渲染器，我们查找其下的plugins.particle究竟是在哪里定义的。

我们在WebGLRenderer.js的166行找到了依据：
`utils.pluginTarget.mixin(WebGLRenderer);`

在这里对WebGLRenderer进行了对象插件式增强, 让其拥有了`registerPlugin`, `initPlugins`, `destroyPlugins`
等方法。

在ParticleRenderer.js的68行, 我们给粒子渲染器注册了插件"particle", 使其能给plugin访问到。
`WebGLRenderer.registerPlugin('particle', ParticleRenderer);`

所以回过头看`renderer.plugins.particle.render( this )`, 这里会执行webgl对应的渲染模式，这行代码就不难理解了。

接下来我们就要看Renderer方法。分别是Sprite和ParticleContainer的webGL式渲染方法。

根据Sprite的依赖，我们要先去看renderer模块。

### ObjectRenderer = require('../../renderers/webgl/utils/ObjectRenderer'),
其依赖WebGLManager, 这个我们之前已经了解过，是用来管理传入的PIXI.WebGLRenderer的。

ObjectRender本身也是个抽象类，其定义了start，stop，flush，render等方法等待覆写。

### WebGLRenderer = require('../../renderers/webgl/WebGLRenderer'),
来看渲染器。
其依赖的玩意有点多，感觉被吓到了：

    var SystemRenderer = require('../SystemRenderer'), 
        ShaderManager = require('./managers/ShaderManager'),
        MaskManager = require('./managers/MaskManager'),
        StencilManager = require('./managers/StencilManager'),
        FilterManager = require('./managers/FilterManager'),
        BlendModeManager = require('./managers/BlendModeManager'),
        RenderTarget = require('./utils/RenderTarget'),
        ObjectRenderer = require('./utils/ObjectRenderer'),
        FXAAFilter = require('./filters/FXAAFilter'),
        utils = require('../../utils'),
        CONST = require('../../const');

在研究他们到底是什么之前, 我们需要先入门下webGL绘图。

没错，我们到现在为止遇到webGL相关的基本就一头雾水，我们来自主绘制下图：
原教程地址：[welGL入门-绘制多边形](http://blog.csdn.net/lufy_legend/article/details/38446243)

![webGL绘图demo](http://7xny7k.com1.z0.glb.clouddn.com/webGLdemo.jpg)

大致上的步骤是这样的：

从HTML中获取canvas对象，从canvas中获取WebGL的context

编译着色器

准备模型数据

顶点缓存（VBO）的生成和通知

坐标变换矩阵的生成和通知

发出绘图命令

更新canvas并渲染

第一步是获取canvas元素, 让我从dom开始进入webGL

        // canvas对象获取  
        var c = document.getElementById('canvas');  
        c.width = 300;  
        c.height = 300;  
        // webgl的context获取  
        var gl = c.getContext('webgl') || c.getContext('experimental-webgl');  
        // 设定canvas初始化的颜色  
        gl.clearColor(0.0, 0.0, 0.0, 1.0);  
        // 设定canvas初始化时候的深度  
        gl.clearDepth(1.0);  
        // canvas的初始化  
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);  

第二步生成顶点着色器和片段着色器, 使用程序对象来链接.

我们可以这么理解：顶点着色器就是处理顶点相关的信息，比如坐标变换（像是模型变幻，视图变换，投影变换这些）。而片段着色器就是处理画面上的颜色信息。

程序对象是管理顶点着色器和片段着色器，或者WebGL程序和各个着色器之间进行数据的互相通信的重要的对象。生成程序对象，并把着色器传给程序对象，然后连接着色器，将这些处理函数化。就是我们要做的。

    // 顶点着色器和片段着色器的生成  
    var v_shader = create_shader('vs');  //我们自己写，用途很明确，就是根据参数返回着色器对象。
    var f_shader = create_shader('fs');  //同样是返回着色器对象。
    // 程序对象的生成和连接  
    var prg = create_program(v_shader, f_shader);  //返回程序对象。
    // attributeLocation的获取  
    var attLocation = gl.getAttribLocation(prg, 'position');  
    // attribute的元素数量(这次只使用 xyz ，所以是3)  
    var attStride = 3; 

第三步定义模型数据，并生成VBO，然后为了将VBO传给顶点着色器，进行绑定并传入数据。在这一步实际上已经脱离我们当前正在了解着的shader着色器部分了。但了解一个完整的渲染流程是很有必要的。我们继续看。
在这里我们定义了基础数据，生成了VBO让他和上一步的attribute的属性相链接。

    // 模型（顶点）数据  
    var vertex_position = [  
         0.0, 1.0, 0.0,  
         1.0, 0.0, 0.0,  
        -1.0, 0.0, 0.0  
    ];  
    // 生成VBO  
    var vbo = create_vbo(vertex_position);  
    // 绑定VBO  
    gl.bindBuffer(gl.ARRAY_BUFFER, vbo);  
    // 设定attribute属性有效
    gl.enableVertexAttribArray(attLocation);  
    // 添加attribute属性  
    gl.vertexAttribPointer(attLocation, attStride, gl.FLOAT, false, 0, 0); 

第四步是矩阵变换，在PIXIjs里面由矩阵类来进行计算。

    // 使用minMatrix.js对矩阵的相关处理  
    // matIV对象生成  
    var m = new matIV();  
    // 各种矩阵的生成和初始化  
    var mMatrix = m.identity(m.create());  
    var vMatrix = m.identity(m.create());  
    var pMatrix = m.identity(m.create());  
    var mvpMatrix = m.identity(m.create());  
    // 视图变换坐标矩阵  
    m.lookAt([0.0, 1.0, 3.0], [0, 0, 0], [0, 1, 0], vMatrix);  
    // 投影坐标变换矩阵  
    m.perspective(90, c.width / c.height, 0.1, 100, pMatrix);  
    // 各矩阵想成，得到最终的坐标变换矩阵  
    m.multiply(pMatrix, vMatrix, mvpMatrix);  
    m.multiply(mvpMatrix, mMatrix, mvpMatrix);  
    // uniformLocation的获取  
    var uniLocation = gl.getUniformLocation(prg, 'mvpMatrix');  
    // 向uniformLocation中传入坐标变换矩阵  
    gl.uniformMatrix4fv(uniLocation, false, mvpMatrix); 

第五步当着色器，顶点数据，坐标变换矩阵等，各种准备工作都完工了，终于该写绘制命令了。这里的drawArrays的第一个参数是使用顶点绘图的方式。gl.TRIANGLES意为纯三角形。

    // 绘制模型  
    gl.drawArrays(gl.TRIANGLES, 0, 3);  
    // context的刷新  
    gl.flush(); 

那么我回来继续看渲染器的依赖，有了上述demo的经验，我们现在对webGL稍稍不是太陌生了。

渲染的依赖分别是什么呢? 如下： 

* SystemRenderer = require('../SystemRenderer'), 
系统渲染器, 不要忘记往dom中添加canavs元素, 不然可能什么也看不见。
* ShaderManager = require('./managers/ShaderManager'),
着色器管理。

`TextureShader = require('../shaders/TextureShader'),`

`var Shader = require('./Shader');`

着色器Shader会有一个初始化编译的过程。

`this.init()`执行初始化, 其中关键一步是

`this.compile()`执行编译, 其中又将着色方法分为两种

`var glVertShader = this._glCompile(gl.VERTEX_SHADER, this.vertexSrc);var glFragShader = this._glCompile(gl.FRAGMENT_SHADER, this.fragmentSrc);`在其中这里构造最初始着色器，也就是顶点着色器和片段着色器。

`var shader = this.gl.createShader(type);`最后还是会用getContext('webGL')来获取webGL绘制对象。

`var program = gl.createProgram();`还有少不了的程序对象的生成。

按照javascript面向对象范式，一个类的构造函数里一般我们都期望只定义私有属性。
然而有些复杂的属性比如这次的shader的编译构造过程相对很繁琐，我们更希望用init()来初始化，这样隔绝出来的代码显得职责分明。

我们的shader相当接近底层，比如`Shader.prototype.syncUniform`方法, 根据uniform.type的不同, 他会调用webGL绘制对象的`uniform1i`,`uniform1f`,`uniform2f`,`uniform3f`,`uniform1iv`,`niform1fv`,`uniformMatrix2fv`等不同方法, 这些方法源自webGL的独有的api，这里主要是对各种参数进行适配。

材质着色器TextureShader在Shader的基础上具体了一点，并且提供了顶点着色器和片段着色器的默认方法：
这个语法是GLSL（OpenGL着色语言）代码。

    TextureShader.defaultVertexSrc = [
        'precision lowp float;',
        'attribute vec2 aVertexPosition;',
        'attribute vec2 aTextureCoord;',
        'attribute vec4 aColor;',
        'uniform mat3 projectionMatrix;',
        'varying vec2 vTextureCoord;',
        'varying vec4 vColor;',
        'void main(void){',
        '   gl_Position = vec4((projectionMatrix * vec3(aVertexPosition, 1.0)).xy, 0.0, 1.0);',
        '   vTextureCoord = aTextureCoord;',
        '   vColor = vec4(aColor.rgb * aColor.a, aColor.a);',
        '}'
    ].join('\n');

类似这样的字符串拼接成的代码段，在TextureShader构造函数里面：

    vertexSrc = vertexSrc || TextureShader.defaultVertexSrc;
    ...
    Shader.call(this, shaderManager, vertexSrc, fragmentSrc, uniforms, attributes);

传递给着色器, 在着色器Shader.js的552行, 也就是之前提及的编译函数里面使用:

    this.gl.shaderSource(shader, src); //这里的src就是刚才的源码

这个shaderSource的API就是webGL提供的链接底层的代码, 通过他我们将直接通过webGL操作显卡。 在这里能实现浏览器环境下的跨语言的程序代码执行。

“初始复杂着色器”ComplexPrimitiveShader自然只需要指定不同的着色器初始化参数，就能和其他着色器区分开来。比如下面的就是顶点着色器的GLSL代码。

    [
        'attribute vec2 aVertexPosition;',
        'uniform mat3 translationMatrix;',
        'uniform mat3 projectionMatrix;',
        'uniform vec3 tint;',
        'uniform float alpha;',
        'uniform vec3 color;',
        'varying vec4 vColor;',
        'void main(void){',
        '   gl_Position = vec4((projectionMatrix * translationMatrix * vec3(aVertexPosition, 1.0)).xy, 0.0, 1.0);',
        '   vColor = vec4(color * alpha * tint, alpha);',//" * vec4(tint * alpha, alpha);',
        '}'
    ].join('\n'),
    
“初始着色器”PrimitiveShader也是如此。可以通过看其顶点着色器代码来稍微推测此语言的用法。

    [
        'attribute vec2 aVertexPosition;',
        'attribute vec4 aColor;',
        'uniform mat3 translationMatrix;',
        'uniform mat3 projectionMatrix;',
        'uniform float alpha;',
        'uniform float flipY;',
        'uniform vec3 tint;',
        'varying vec4 vColor;',
        'void main(void){',
        '   gl_Position = vec4((projectionMatrix * translationMatrix * vec3(aVertexPosition, 1.0)).xy, 0.0, 1.0);',
        '   vColor = aColor * vec4(tint * alpha, alpha);',
        '}'
    ].join('\n'),

集齐了三个龙珠（误）。
加载了TextureShader，ComplexPrimitiveShader，PrimitiveShader三个模块后，我们就可以用ShaderManager来对他们进行管理了。
管理器Manager这些类，全部继承自WebGLManager类，几乎覆写了全部方法。所以每个特定的管理器类各自的特性就尤为重要。
着色器管理类`ShaderManager`特性主要是：onContextChange方法调用时初始化全部的三个着色器模块，他有自己的设置着色器方法，这样就可以实现对着色器的管理。

* MaskManager = require('./managers/MaskManager'),
遮罩管理需要`var AbstractFilter = require('./AbstractFilter'),`,这个类没有父类。有一个滤镜应用的方法需要关注。

        AbstractFilter.prototype.applyFilter = function (renderer, input, output, clear)
        {
            var shader = this.getShader(renderer);
            renderer.filterManager.applyFilter(shader, input, output, clear);
        };

同时还需要`var fs = require('fs');`, 这是nodejs用来进行文件管理的模块, 一般用来文件读取, 文件输出等操作的。这里为了不用像之前那样用字符串拼接方式写GLSL代码所以采用文件读取的方式。

    AbstractFilter.call(this,
        fs.readFileSync(__dirname + '/spriteMaskFilter.vert', 'utf8'),
        fs.readFileSync(__dirname + '/spriteMaskFilter.frag', 'utf8'),
        {
            mask:           { type: 'sampler2D', value: sprite._texture },
            alpha:          { type: 'f', value: 1},
            otherMatrix:    { type: 'mat3', value: maskMatrix.toArray(true) }
        }
    );

在spriteMaskFilter.vert文件里，就可以正常的写GLSL程序了。

分析完依赖，遮罩只要提供极简的API就行了：

`MaskManager.prototype.pushMask`和`MaskManager.prototype.popMask`, 通过这两个API实现对遮罩的管理。

* StencilManager = require('./managers/StencilManager'),
模板管理StencilManager, 和遮罩类类似。核心方法`pushStencil`和`popStencil`。

* FilterManager = require('./managers/FilterManager'),
滤镜效果管理FilterManager，核心方法`pushFilter`，`popFilter`，`applyFilter`

* BlendModeManager = require('./managers/BlendModeManager'),
混合模式管理BlendModeManager, 核心方法`setBlendMode`。其比较简单，所以只在WebglManager基础上稍微扩充下就行了。

        BlendModeManager.prototype.setBlendMode = function (blendMode)
        {
            if (this.currentBlendMode === blendMode)
            {
                return false;
            }
            this.currentBlendMode = blendMode;
            var mode = this.renderer.blendModes[this.currentBlendMode];
            this.renderer.gl.blendFunc(mode[0], mode[1]);
            return true;
        };

* RenderTarget = require('./utils/RenderTarget'),
获取渲染context对象。

* ObjectRenderer = require('./utils/ObjectRenderer'),
抽象类。

* FXAAFilter = require('./filters/FXAAFilter'),
滤镜的顶点与碎片着色器程序。

* utils = require('../../utils'),
工具类。

* CONST = require('../../const');
常量。

分析完依赖，回到WebGLRenderer.js里面。

WebGLRender会把场景和其中所有的内容都渲染在一个开启了webgl的canvas。当前浏览器也需要支持webgl。
这个渲染器会自动处理webGL相关的内容，我们之前分析的滤镜，遮罩，着色器，模板等管理类就是用来自动处理这些玩意的。

我们来看这个集大成者有哪些功能：

获取webgl的context对象的方法如下：

    WebGLRenderer.prototype._createContext = function () {
        var gl = this.view.getContext('webgl', this._contextOptions) || this.view.getContext('experimental-webgl', this._contextOptions);
        this.gl = gl;
        if (!gl)
        {
            throw new Error('This browser does not support webGL. Try using the canvas renderer');
        }
        this.glContextId = WebGLRenderer.glContextId++;
        gl.id = this.glContextId;
        gl.renderer = this;
    };

`WebGLRenderer.prototype._initContext`提供的是初始化context的方法。比如canvas渲染所不支持的filter模式需要在这里初始化。涉及到这部分的代码如下。

    if(!this._useFXAA)
    {
        this._useFXAA = (this._contextOptions.antialias && ! gl.getContextAttributes().antialias);
    }
    if(this._useFXAA)
    {
        window.console.warn('FXAA antialiasing being used instead of native antialiasing');
        this._FXAAFilter = [new FXAAFilter()];
    }

`WebGLRenderer.prototype._mapGlModes`是canvas和webgl混合模式之间的转换。blendModes和drawModes都会有相应的映射。

绕了一大圈我们弄清WebGLRenderer是什么之后，还记得它是作为谁的依赖来着么？

### SpriteRenderer: require('./sprites/webgl/SpriteRenderer'),
对啊，就是这个。sprite的webGL渲染器。

在看了那么多的渲染相关的方法之后，对于这个sprit渲染器的功能已经不需要赘述了。

### ParticleRenderer: require('./particles/webgl/ParticleRenderer'),
粒子的webgl渲染方法因为渲染对象的差异性，还需要额外引入两个文件。

一个是ParticleBuffer.js用来管理粒子缓存，一个是ParticleShader.js用来修改着色器程序和参数的。

其实现和SpriteRenderer也不尽相同。可以参考canvas渲染的差异。

总之，webgl渲染方式实现起来是相当底层的。canvas在浏览器帮助下多封装了一层。webGL只能用js调用底层代码一点点实现。理论上，直接操作显卡的webGL渲染方式必然比canvas快，唯一的性能瓶颈可能只在操作webGL的js代码上。
