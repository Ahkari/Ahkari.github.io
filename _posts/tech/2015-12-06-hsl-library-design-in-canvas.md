---
layout: post
title: HSL色彩模式库设计
category: 技术
tags: canvas hsl smartHSL
description: 如何从hsl基础理论开始撸一个hsl处理库
---

### 地址

先把地址放了吧, 大家有兴趣可以先去看看。

[smartHSL-源码-github](https://github.com/Ahkari/smartHSL)

[smartHSL-极速操作hsl-demo](http://ahkari.github.io/smartHSL/)


### 何来的需求?

最近(其实就是下周啦)项目里要加一个控件。 
要求能设定图片的色相, 饱和度和明度, 按理说只是配置之后给引擎用的, 可是临时增加了需求， 要求能预览。

于是用一天时间从HSL色彩模式理论开始，撸了个基于canvas的HSL模式操作库。
按照我的想法，这个库自然是能多简单要多简单，来看看使用方式就知道了。

`var smartCanvas = new SmartHSL( canvas ) ; //实例化操作库` 
`smartCanvas.hsl(h,s,l) ; //一次配置三个参数`
`smartCanavs.h(h) ;`
`smartCanavs.s(s) ;`
`smartCanavs.l(l) ; //单独配置`
`smartCanvas.reset() ; //重置图像`

### HSL与RGB理论

最好的知识储备来自维基百科
[HSL与HSV色彩空间-维基百科](http://zh.wikipedia.net.ru/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4)

我们熟知的RGB色彩空间和HSL色彩空间都是用来表示颜色的。一般来说设计师用HSL模式的更多, 因为他直观简便。
如果我们构思着需要一个`浅浅的`,`明亮的`,`紫色`。
对于RGB模式,  我们就要考虑红蓝混合, 考虑分别给多少的量, 甚至还要注意绿通道的影响。

但是对于hsl就简单很多了，我们通过h色相找到所期望的紫色，s饱和度调低，l明度提高就得到了结果。
可以说，hsl模式是符合人类对颜色的直观印象，设计师喜欢用hsl模式也合情合理。

hsl空间可以视为一个圆柱, 色相是可以用角度表示, 饱和度可以用与圆心距离表示, 明度可以用圆柱高度表示。
![直观的颜色空间](http://7xny7k.com1.z0.glb.clouddn.com/hslandhsv.png)

我们参照ps的色相/饱和度操作方式来定义我们库操作的取值：
![ps的色相/饱和度调整](http://7xny7k.com1.z0.glb.clouddn.com/hsl.png)

色相取值范围是-180~+180，他本质是一个色轮，在-180和+180时是一样的。
饱和度取值范围是-100~+100，为-100的时候图片将呈现出黑白。这时候色相将影响不了图像，色相此时理应为“未定义”。
明度取值范围是-100~+100，越低画面越暗。

### canvas基础

html5的canvas元素将web表现能力整体提升了一个档次，能帮助开发者在浏览器里实现不输于flash的图像表现能力。近年来基于html5 canvas的web富应用，游戏发展也愈发蓬勃。

我们将要利用canvas像素处理能力来调整图像hsl，

1. 加载图片
    
        var canvas = document.querySelector('#smartCanvas'),
            context = canvas.getContext('2d') ;
        var canvasImg = new Image() ;
        canvasImg.src = src ; //这个src需要自己去设置, 可以是本域下的图片url, 或FileReader接口readAsDataURL所获取的result
        canvasImg.onload = function(e){ //src指定之后, 图像加载完毕时触发
            var zoom = Math.min( canvas.width/this.width , canvas.height/this.height ) ; //适配
            context.clearRect( 0 , 0 , canvas.width , canvas.height ) ; //清除原canvas像素
            context.drawImage( canvasImg , 0, 0 , this.width*zoom , this.height*zoom ) ; //图像绘制
        } ;

    在我库的demo中, 提供了注释所述两种图片加载方式, 但他们都是设定src的方式`canvasImg.src = src`,
    理论上能通过设定src展示图片的都能通过`drawImage`来绘制到canvas上, 不过非同源的图像却不能实现像素处理。

2. 像素操作
    
    至此就要用到我们的库了，我们如下调用`SmartCanvas`的`hsl`方法
        
        if ( !smartCanvas ){
            smartCanvas = new SmartHSL(canvas) ; //单例模式，确保对于这个canvas只存在一个操作实例
        }
        console.log('h:'+h+',s:'+s+',l:'+l) ;
        smartCanvas.hsl(h,s,l) ; //对操作实例执行hsl变换
        
    库里的`hsl`方法如下：
    
        HSL : function(){
            for ( var i=0 ; i < arguments.length ; i++ ){
                this[ this.map[ i+1 ] ] = arguments[i] ;//这里是按照参数顺序把HSL设定好
            }
            this.setHSL( this.Hue , this.Saturation , this.Lightness ) ;//执行统一的setHSL方法
        },

    HSL配置好`smartCanvas`的hsl参数后，调用`setHSL`方法就行。
    类似的，对h，s，l单独设置的方法其实就是改下单参数的值，之后仍然统一调用`setHSL` ；
        
        H : function(){
            var h = parseInt( arguments[0] ) ;
            if ( isNaN(h) ){
                return ;
            }
            this.Hue = h ;
            this.setHSL( this.Hue , this.Saturation , this.Lightness ) ;
        },
    
    那么来看关键的canvas设置hsl方法`setHSL`，
    
        setHSL : function( H, S, L ){ 
            this.imageOperate( this.imageDate , this._imageDate , H ,S ,L ) ;
            this.context.putImageData( this._imageDate , 0 , 0 ) ;
        },
    
    这里的`imageData`是实例化canvas之时存储好的最初的图片所有的像素信息，每次的操作都是对初始图片处理，避免了在调整过的图片上二次操作的可能。
    `_imageDate`是和原canvas等大的图片信息存储。从`SmartHSL`构造函数上可以看到他们分别的生成方法。
    
        function SmartHSL(canvas){
            this.canvas = canvas ;
            this.context = this.canvas.getContext('2d') ;
            this.imageDate = this.context.getImageData( 0, 0, this.canvas.width , this.canvas.height ) ;
            this._imageDate = this.context.createImageData( this.canvas.width , this.canvas.height ) ;
            this.map = {
                1 : 'Hue' ,
                2 : 'Saturation' ,
                3 : 'Lightness' 
            } ; //这个是用来设定hsl顺序的。其实可以独立成一个API。
            this.Hue = 0 ;
            this.Saturation = 0 ;
            this.Lightness = 0 ;
        }
        
    setHSL中的`putImageData`方法将canvas备份`_imageData`绘制回去。之后的每次设置都是`_imageData`信息的重复绘制。


### HSL处理

那么问题来了, `setHSL`中的核心操作究竟做了什么?
`this.imageOperate( this.imageDate , this._imageDate , H ,S ,L ) ;`
在这里, 我们根据H ,S ,L 的数据, 把`iamgeData`数据修改进`_imageData`中。
问题其一就是：`imageData`是一个像素信息数组，他是这样个结构：
![canvas的像素信息](http://7xny7k.com1.z0.glb.clouddn.com/RGBA.png)
可见canvas的像素描述方式却是RGBA颜色模式。
每四个决定一个像素，分别对应R,G,B,A。由于A（透明度）比较特殊，并不对应HSL，所以在处理的过程中我们不对Alpha作处理。

做调整操作我们需要以下三步：
1.  当前的RGB转换为HSL。
2.  当前像素的HSL，根据配置信息H,S,L，生成目标HSL
3.  将目标HSL转换回RGB，填充给`_imageData` 

### 第一步:RGB转HSL
公式来自维基百科：
![RGB转HSL公式](http://7xny7k.com1.z0.glb.clouddn.com/rgb2hsl.png)
原样实现即可：不过要对输入参数适配下，这里都除以255确保RGB位于[0,1]区间

    rgb2hsl : function( rgba ){ 
        var r = rgba.red / 255 ,
            g = rgba.green / 255 ,
            b = rgba.blue / 255 ;
        var max = Math.max(r,g,b) ,
            min = Math.min(r,g,b) ;
            hsl = {} ;
        if ( max === min ){
            hsl.h = 0 ;
        }else if( max === r && g>=b ){
            hsl.h = 60*( (g-b)/(max-min) ) ;
        }else if( max === r && g<b ){
            hsl.h = 60*( (g-b)/(max-min) ) + 360 ;
        }else if( max === g ){
            hsl.h = 60*( (b-r)/(max-min) ) + 120 ;
        }else if( max === b ){
            hsl.h = 60*( (r-g)/(max-min) ) + 240 ;
        }
        hsl.l = (max+min)/2 ;
        if ( hsl.l===0 || max===min ){
            hsl.s = 0 ;
        }else if( 0<hsl.l && hsl.l<=0.5 ){
            hsl.s = (max-min)/(2*hsl.l) ;
        }else if( hsl.l>0.5 ){
            hsl.s = (max-min)/(2-2*hsl.l) ;
        }
        rgba.alpha && (hsl.a = rgba.alpha)  
        return hsl ;
    },
    
### 第二步:HSL调整

我们这里有一个有趣的思考: 我们要把像素调整成配置所述的h,s,l值吗?

如果你回答**是**, 那再思考下, 这样我们把图像处理完之后, 这个图片是不是就变成一个纯色整体了?

如果你意识到症结所在的话, 你就会重新审视"图像处理"的这个命题了。

图像处理，不是把像素按照配置设定。
是对图片每个像素都按照配置执行一套预定操作，这个操作必须是基于原有像素的。不同的像素处理完之后会有不同的结果。
所以除了一些特殊操作，基础图像处理都需要保持原图像精度。

对于任何一个初始图，我们调整hsl的时候都是从0,0,0开始的，他们可调整值范围分别是-360~+360，-100~+100，-100~+100。
就是说，我们获取到一个经由RGB转换来的HSL，和配置比较的时候都要把它视为0,0,0，然后用这个比值关系把实际hsl转换成目标hsl。直接看代码吧：

    hslConvert : function( hslDate , options ){
        var h = hslDate.h ,
            s = hslDate.s ,
            l = hslDate.l ;
        var hOpe ;
        hOpe = parseInt(options[0])+180 ; //[-180,180]区间转为[0,360]区间
        var sOpe ;
        sOpe = (parseInt(options[1])+100)/200 ; //[-100,100]区间转为[0,1]区间
        var lOpe ;
        lOpe = (parseInt(options[2])+100)/200 ; //[-100,100]区间转为[0,1]区间
        //色相比值计算
        function hAdapt(old,ope){ 
            return old+ope-180 ;
        } 
        //饱和度比值计算
        function sAdapt(old,ope){
            var newS ; 
            if ( ope === 0.5 ){
                newS = old ;
            }else if ( ope < 0.5 ){
                newS = old*ope/0.5 ;
            }else if ( ope > 0.5 ){
                newS = 2*old + 2*ope - (ope*old/0.5) - 1 ;
            }
            return newS ;
        } 
        //明度比值计算
        function lAdapt(old,ope){
            var newL ; 
            if ( ope === 0.5 ){
                newL = old ;
            }else if ( ope < 0.5 ){
                newL = old*ope/0.5 ;
            }else if ( ope > 0.5 ){
                newL = 2*old + 2*ope - (ope*old/0.5) - 1 ;
            }
            return newL ;
        } 
        var ret = {
            h : hAdapt( h , hOpe ) ,
            s : sAdapt( s , sOpe ) ,
            l : lAdapt( l , lOpe ) 
        } ;
        hslDate.a && ( ret.a = hslDate.a ) ;
        return ret ;
    }, 

### 第三步:HSL转回RGB

公式来自维基百科
不过其中有个那个困扰我半天的地方需要注意下, 这个我还是查了半天, 最后在一个菊苣用excel编写HSL转RGB拓展时弄明白的。

![HSL转RGB](http://7xny7k.com1.z0.glb.clouddn.com/HSL2RGB.png)

代码如下：
    
    hsl2rgb : function( hsl ){
        var h  = hsl.h , //0-360
            s = hsl.s ,
            l = hsl.l ;
        var rgba = {} ;
        if ( s === 0 ){
            rgba.red = rgba.green = rgba.blue = Math.round( l*255 ) ;
        }else{
            var q ,
                p ,
                hK ,
                tR ,
                tG ,
                tB ;
            if ( l<0.5 ){
                q = l * ( 1 + s ) ;
            }else if( l>=0.5 ){
                q = l + s - ( l * s ) ;
            }      
            p = 2*l-q ;
            hK = h/360 ;
            tR = hK + 1/3 ;
            tG = hK ;
            tB = hK - 1/3 ;
            var correctRGB = function(t){
                if( t<0 ){
                    return t + 1.0 ;
                }
                if( t>1 ){
                    return t - 1.0 ;
                }
                return t ;
            } ;
            var createRGB = function(t){
                if ( t<(1/6) ){
                    return p+((q-p)*6*t) ;
                }else if( t>=(1/6) && t<(1/2) ){
                    return q ;
                }else if( t>=(1/2) && t<(2/3) ){
                    return p+((q-p)*6*((2/3)-t)) ;
                }
                return p ;
            } ;
            rgba.red = tR = Math.round( createRGB( correctRGB( tR ) )*255 ) ;
            rgba.green = tG = Math.round( createRGB( correctRGB( tG ) )*255 ) ;
            rgba.blue = tB = Math.round( createRGB( correctRGB( tB ) )*255 ) ;
        }
        hsl.a && ( rgba.alpha = hsl.a ) ;  
        return rgba ;
    },
    
    
至此操作库已介绍完，下面是演示地址，大家有兴趣可以玩玩，因为canvas是可以直接复制保存图片的，所以用来做在线HSL处理也是可以的233。

[smartHSL-极速操作hsl](http://ahkari.github.io/smartHSL/)