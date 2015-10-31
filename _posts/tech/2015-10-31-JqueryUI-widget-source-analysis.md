---
layout: post
title: widget对象工厂源码解析
category: 技术
tags: Jquery JqueryUI widget 源码解析
description: 源码解析jqueryUI的核心方法widget，理解源码可以帮助我们构造出各种各样的复杂控件。
---

### $.widget是什么

`$.widget()`是`jquery.ui`的核心方法。所有`jquery.ui`的内置控件都是通过widget构造而来。
更重要的是，理解`widget`方法之后我们能随心所欲构造自定义控件。
谈及控件，我们不必都死死的联系到那种`button`式点击了然后触发操作的按钮式强交互控件。我们可以理解为这是一种`UI`类，用来管理`view`状态与行为的对象。这个对象强关联（绑定）在一个`jquery`包装过的`dom`元素上，并且接管了它的内部所有行为与数据操作。
当一个大型应用通篇使用`jqueryUI`的`widget`工厂模式设计的时候，我们应分解页面为一个个控件模块，通过控件模块之间的层层调用实现复杂的`web`应用，实现组件化开发。不放出任何内部操作到控件树的外面。
经由`widget`部件库构造而来的控件，都有`jqueryUI`控件的一系列优秀特性：
* 状态维持
* 控件继承
* 内置API一致性
* 命名空间管理

### 从widget使用方法探究其实现

widget使用方法有如下几个场景,我们以新建一个自定义控件为例
1 **定义控件**
        
    $.widget('ui.test','ui.superTest',{
        options:{},
        _create:function(){}
    }) ;
        
我们声明了一个`ui`命名空间下的名为`test`的控件，他继承自`ui.superTest`控件，最后一个参数是控件的原型对象，里面`options`是默认参数，`_create`是初次调用时执行的代码。如果要写自定义方法或属性，就在原型里写和`options`，`_create`同级的方法和属性就行。

对于这个使用方法我们猜想下其实现：
一定是有一个`$.widget()`工厂方法，根据参数不同构建出不同的控件对象。
这个工厂方法本身还可能有原型方法，因为我们所知构建出的控件具有很多内置方法。

2 **给控件传参**
    
    `$( selectorString ).test( options ) ;`

我们这里对一个`jquery`对象直接使用定义好的`test`方法，如果接受一个对象，那么理解为对控件进行**配置**，`options`里面的数据都会替换或增强步骤1定义时的默认`options`参数，此步会在`jquery`对象上**初始化控件**。

猜想其实现：
步骤1里工厂方法里所构建出的`widget`对象需要插件化，便于被`jquery`元素插件式调用。我们猜想这里和`$.fn`一定有着千丝万缕的联系。

3 **使用控件方法**

    `$( selectorString ).test( funcName , arguments ) ;`

这里调用了`test`控件的`funcName`的方法，后面是这个方法所接受的参数。

猜想其实现：
类似于步骤二，这里因参数不同而插件式调用`widget`定义好的方法。

这三个步骤我们之后会细说其实现。
先看源码总体架构，并验证我们的大体猜想。

### widget总体架构

    (function( factory ) {
        //模块化支持
    }(function( $ ) {
        $.cleanData = (function( orig ) {
            //增强cleanData方法,使其兼容jqueryUI
            //其相关的bug可查看http://bugs.jquery.com/ticket/8235和github上的pull历史https://github.com/jquery/jquery/pull/459
        })( $.cleanData );
        $.widget = function( name, base, prototype ) {
            //工厂方法，A处。
        };
        $.widget.extend = function( target ) {
            //类似jquery的深复制
        };
        $.widget.bridge = function( name, object ) {
            //将构造的对象转为插件模式，B处。
        };
        
        //构造函数,注意下面的基类相关都是第一个字母大写的Widget
        $.Widget = function( /* options, element */ ) {}; 
        $.Widget._childConstructors = [];
        $.Widget.prototype = {
            //widget原型方法，内置事件，私有事件都在此处。C处。
        };
        $.each( { show: "fadeIn", hide: "fadeOut" }, function( method, defaultEffect ) {
            //为widget添加show和hide方法
        });
        var widget = $.widget;
    }));

可见源码折叠之后架构一目了然，基本验证了我们的猜想。

### widget对象工厂解析
**A处代码如下**
功能：完成对控件的注册与定义。

    $.widget = function( name, base, prototype ) {
        var fullName, existingConstructor, constructor, basePrototype,
            proxiedPrototype = {},
        namespace = name.split( "." )[ 0 ];//命名空间'.'之前
        name = name.split( "." )[ 1 ];//控件名是'.'之后，所以我们由此知道只能有一层命名空间
        fullName = namespace + "-" + name;//插件全称更名为namespace-name，存储用
        if ( !prototype ) {//假设不存在第三个参数，那我们估计这个插件得继承自最原始的widget咯
            prototype = base;//那么第二个参数就是当前插件的原型
            base = $.Widget;//果不其然，继承自$.Widget，这一看就是原始的widget
        }
        //注一：添加插件选择器
        $.expr[ ":" ][ fullName.toLowerCase() ] = function( elem ) {
            return !!$.data( elem, fullName );
        };
        $[ namespace ] = $[ namespace ] || {};//判定$下命名空间对象是否存在，没有的话 则创建一个空对象
        existingConstructor = $[ namespace ][ name ];//先存储当前插件的旧版插件（假如有）
        //注二：这里是核心之一，我们定义类/控件类的最终目的就是创建其构造函数并关联原型。现在我们的构造函数命名为constructor，并放进插件方法里，在调用时被正式实例化。
        constructor = $[ namespace ][ name ] = function( options, element ) {
            //constructor存储了插件的实例，同时也创建了基于命名空间的对象
            if ( !this._createWidget ) {
                return new constructor( options, element );
            }
            if ( arguments.length ) {
                this._createWidget( options, element );
            }
        };
        //将旧版本插件合并到新版本，修改原来的版本，原型方法和插件实例。
        $.extend( constructor, existingConstructor, {
            version: prototype.version,
            _proto: $.extend( {}, prototype ),
            _childConstructors: []
        });
        basePrototype = new base();//实例化父类
        basePrototype.options = $.widget.extend( {}, basePrototype.options );//注三：深复制父类options
        //遍历原型方法，对方法和属性区分处理
        $.each( prototype, function( prop, value ) {
            if ( !$.isFunction( value ) ) {
                proxiedPrototype[ prop ] = value;//直接复制属性
                return;
            }
            proxiedPrototype[ prop ] = (function() {
                //用闭包将父类中方法保存,子类通过`_super()`和`_superAply()`调用父类同名方法；
                //_super是call式调用
                var _super = function() {
                        return base.prototype[ prop ].apply( this, arguments );
                    },
                    //_superApply是apply式调用
                    _superApply = function( args ) {
                        return base.prototype[ prop ].apply( this, args );
                    };
                return function() {
                    //执行强化后的子类方法时，现有的_super会被屏蔽
                    var __super = this._super,
                        __superApply = this._superApply,
                        returnValue;
                    //this._super指向闭包存储的父类方法
                    this._super = _super;
                    this._superApply = _superApply;
                    //传递this，子类方法中this._super最终指向父类方法
                    returnValue = value.apply( this, arguments );
                    //最后将屏蔽的原_super还原，_super本身也是没起作用
                    this._super = __super;
                    this._superApply = __superApply;
                    //函数结果返回
                    return returnValue;
                };
            })();
        });
        //整合构造函数原型
        constructor.prototype = $.widget.extend( basePrototype, {
            widgetEventPrefix: existingConstructor ? (basePrototype.widgetEventPrefix || name) : name
        }, proxiedPrototype, {
            constructor: constructor, //构造器
            namespace: namespace, //命名空间
            widgetName: name, //控件名
            widgetFullName: fullName //控件全名
        });
        if ( existingConstructor ) { 
            //如果控件已定义过，执行如下处理。
            $.each( existingConstructor._childConstructors, function( i, child ) {
                var childPrototype = child.prototype;
                $.widget( childPrototype.namespace + "." + childPrototype.widgetName, constructor, child._proto );
            });
            delete existingConstructor._childConstructors;
        } else {
            base._childConstructors.push( constructor ); //注四：父类的实例再增加一个，因为base始终挂载在全局的$上，所以可以有效统计页面里面的插件数量。
        }
        //widget插件桥
        $.widget.bridge( name, constructor );
        return constructor;
    };

注一：**这里添加了一个插件名命名的伪类选择器。**
现在选择器字符里只要追加上`:fullName`（例如：`$(':ui-test')`）就会选中页面中所有定义`ui.test`控件的元素。
其代码实现是通过jquery内部的`$.expr`方法实现，我们从`return !!$.data( elem, fullName )`推断其实现方式：
检测`elem`中`data`是否有`fullName`对象（我们已经知道`widget`对象就命名为`fullName`，并用`jquery`的`data`方法存在绑定的元素上），并返回结果为true的元素集合。
注二：**类实例化的核心流程。见下方B处代码的注一。**
注三：**深复制`options`的必要性。**
`basePrototype.options = $.widget.extend( {}, basePrototype.options )`
为了让父类配置的改动不会影响子类。

    var super = {foo:1} ;
    var constructor = { proto:super } ;
    //此时 constructor.proto 是 foo:1 
    super.foo = 2 ;
    //此时 constructor.proto 因父类原型属性改变而变成了 foo:2
    //但是我们用jquery深复制赋值一次就不一样了
    constructor.proto = $.extend( {}, constructor.proto ) ; //新的constructor.proto已断开了和super的联系，成为一个新对象。
    super.foo = 3 ；
    //此时 constructor.proto 仍然为foo:2

注四：**我们通过统计页面`widget`实例数量来看控件类继承。**
这里我们知道所有实例化过的控件不仅挂载在$上了，并且一层层套在父类的实例属性里。
比如如下操作：
`$.widget('ui.superWidget',{options:{name:'我是一个父类,没有指定继承,所以父类是$.Widget'}}) ;`
先定义一个继承自$.Widget的父控件
`$.widget('ui.subWidget',$.ui.superWidget,{options:{name:'我是子类,指定继承自superWidget'}}) ;`
再定义一个继承自父控件的子类控件
我们查看`$.Widget._childConstructors`属性，这是记载Widget根控件其所有实例的数组，我们查看最后一个实例的内容：

![父类控件](http://7xny7k.com1.z0.glb.clouddn.com/superWidget.png)

可见这是我们刚刚定义的`superWidget`，显示他又有一个子类实例，点开：

![子类控件](http://7xny7k.com1.z0.glb.clouddn.com/subClass.png)

可见这是我们刚定义的`subWidget`。


**B处代码如下**
通过被jquery元素调用，实例化注册好的widget类。

    $.widget.bridge = function( name, object ) {
        var fullName = object.prototype.widgetFullName || name;
        //将控件方法挂载到$.fn上，使其能被Jquery元素插件式调用。
        //经总结做了如下几个工作：
        //1，参数处理：不同参数执行不同操作，实现多态。（第一个参数是对象时配置，是字符串时执行同名方法。）
        //2，单例模式：只在第一次调用时存储实例，以后都直接调用存储的实例。
        //3，自定义配置：接收参数并交由内部option方法合并处理。
        //4，访问控制：未查找到的方法或以_开头的方法都不能调用。
        $.fn[ name ] = function( options ) {
            var isMethodCall = typeof options === "string",
                args = widget_slice.call( arguments, 1 ),
                returnValue = this;
            if ( isMethodCall ) { //调用时，第一个参数是string，则是方法调用
                this.each(function() {
                    var methodValue,
                        instance = $.data( this, fullName );
                    if ( options === "instance" ) {
                        returnValue = instance;//传入方法名为instance，返回实例。
                        return false;
                    }
                    if ( !instance ) {//这个元素上此控件未曾实例化过，报错。
                        return $.error( "cannot call methods on " + name + " prior to initialization; " +
                            "attempted to call method '" + options + "'" );
                    }
                    if ( !$.isFunction( instance[options] ) || options.charAt( 0 ) === "_" ) { //阻止对不存在的方法和以_开头的方法的访问。
                        return $.error( "no such method '" + options + "' for " + name + " widget instance" );
                    }
                    methodValue = instance[ options ].apply( instance, args );//执行该方法
                    if ( methodValue !== instance && methodValue !== undefined ) {
                        returnValue = methodValue && methodValue.jquery ?
                            returnValue.pushStack( methodValue.get() ) :
                            methodValue;
                        return false;
                    }
                });
            } else {
                if ( args.length ) {
                    options = $.widget.extend.apply( null, [ options ].concat(args) );
                }
                this.each(function() {
                    var instance = $.data( this, fullName );
                    if ( instance ) { 
                        //该元素上控件实例已存在，调用实例option方法，这个我们看内置原型方法的时候再看。猜想这里是做了options合并操作。
                        instance.option( options || {} );
                        if ( instance._init ) {
                            instance._init(); //每次配置参数的时候都会调用_init()方法。
                        }
                    } else {
                        //控件实例不存在，则将控件实例存储在元素的data里。
                        $.data( this, fullName, new object( options, this ) );//注一：实例化
                    }
                });
            }
            return returnValue;
        };
    };

注一：**由A处注二部分的定义的构造函数将在这里被实例化。**
我们看A处注二的代码：

    constructor = $[ namespace ][ name ] = function( options, element ) {
        if ( !this._createWidget ) {
            return new constructor( options, element );
        }
        if ( arguments.length ) {
            this._createWidget( options, element );
        }
    };

这里的`constructor`支持了两种调用方法，一种是直接`$[namespace][name]({},element)`调用方式,这种方式因为没有用`new`调用，其原型中的`_createWidget`并不能访问得到，所以再次执行new调用，来对外实现“无new”式实例化。
第二种就是`new`式实例化，她劫持该函数并用构造对象的形式调用。可见，每次`new`调用时`widget`工厂内部都会执行`_createWidget`方法。

**C处代码如下**
定义`$.Widget`控件通用原型。

    $.Widget = function( /* options, element */ ) {};
    $.Widget._childConstructors = [];
    $.Widget.prototype = {
        widgetName: "widget",
        widgetEventPrefix: "", //事件代理，注一
        defaultElement: "<div>",
        options: {
            disabled: false,
            // callbacks
            create: null
        },
        //_createWidget是初次实例化必然被调用的方法。
        _createWidget: function( options, element ) {
            element = $( element || this.defaultElement || this )[ 0 ];
            this.element = $( element );//this.element已是jquery包装过的元素，可以对其使用一系列jquery方法。
            this.uuid = widget_uuid++;
            this.eventNamespace = "." + this.widgetName + this.uuid;//事件命名空间，保证唯一
            this.bindings = $();
            this.hoverable = $();
            this.focusable = $();
            if ( element !== this ) {
                $.data( element, this.widgetFullName, this );
                this._on( true, this.element, {
                    remove: function( event ) {
                        if ( event.target === element ) {
                            this.destroy();//this.element上监控了remove方法，如果调用remove()则执行控件的destroy()
                        }
                    }
                });
                //确认document和window指向。
                this.document = $( element.style ?
                    element.ownerDocument :
                    element.document || element );
                this.window = $( this.document[0].defaultView || this.document[0].parentWindow );
            }
            this.options = $.widget.extend( {},
                this.options,
                this._getCreateOptions(),
                options );//深合并options
            this._create();//执行_create()初始化事件，注一
            this._trigger( "create", null, this._getCreateEventData() );
            this._init();
        },
        _getCreateOptions: $.noop,
        _getCreateEventData: $.noop,
        _create: $.noop,
        _init: $.noop,
        //内置控件销毁方法：去除绑定事件、去除数据、去除样式、属性
        destroy: function() {
            this._destroy();//支持自定义方法，将在destroy前执行
            this.element
                .unbind( this.eventNamespace )
                .removeData( this.widgetFullName )//
                .removeData( $.camelCase( this.widgetFullName ) );
            this.widget()
                .unbind( this.eventNamespace )
                .removeAttr( "aria-disabled" )
                .removeClass(
                    this.widgetFullName + "-disabled " +
                    "ui-state-disabled" );
            this.bindings.unbind( this.eventNamespace );
            this.hoverable.removeClass( "ui-state-hover" );
            this.focusable.removeClass( "ui-state-focus" );
        },
        _destroy: $.noop,
        widget: function() {
            return this.element;
        },
        //内置参数设置方法：set与get合一。无参时get，有参时set。
        option: function( key, value ) {
            var options = key,
                parts,
                curOption,
                i;
            //注二：options参数探究。
            if ( arguments.length === 0 ) {
                return $.widget.extend( {}, this.options );
            }
            if ( typeof key === "string" ) {
                // handle nested keys, e.g., "foo.bar" => { foo: { bar: ___ } }
                options = {};
                parts = key.split( "." );
                key = parts.shift();
                if ( parts.length ) {
                    curOption = options[ key ] = $.widget.extend( {}, this.options[ key ] );
                    for ( i = 0; i < parts.length - 1; i++ ) {
                        curOption[ parts[ i ] ] = curOption[ parts[ i ] ] || {};
                        curOption = curOption[ parts[ i ] ];
                    }
                    key = parts.pop();
                    if ( arguments.length === 1 ) {
                        return curOption[ key ] === undefined ? null : curOption[ key ];
                    }
                    curOption[ key ] = value;
                } else {
                    if ( arguments.length === 1 ) {
                        return this.options[ key ] === undefined ? null : this.options[ key ];
                    }
                    options[ key ] = value;
                }
            }
            this._setOptions( options );
            return this;
        },
        _setOptions: function( options ) {
            var key;
            for ( key in options ) {
                this._setOption( key, options[ key ] );
            }
            return this;
        },
        _setOption: function( key, value ) {
            this.options[ key ] = value;
            if ( key === "disabled" ) {
                this.widget()
                    .toggleClass( this.widgetFullName + "-disabled", !!value );
                if ( value ) {
                    this.hoverable.removeClass( "ui-state-hover" );
                    this.focusable.removeClass( "ui-state-focus" );
                }
            }
            return this;
        },
        //注三：伪造的enable与disable
        enable: function() {
            return this._setOptions({ disabled: false });
        },
        disable: function() {
            return this._setOptions({ disabled: true });
        },
        //事件绑定
        _on: function( suppressDisabledCheck, element, handlers ) {
            var delegateElement,
                instance = this;
            if ( typeof suppressDisabledCheck !== "boolean" ) {
                handlers = element;
                element = suppressDisabledCheck;
                suppressDisabledCheck = false;
            }
            if ( !handlers ) {
                handlers = element;
                element = this.element;
                delegateElement = this.widget();
            } else {
                element = delegateElement = $( element );
                this.bindings = this.bindings.add( element );
            }
            $.each( handlers, function( event, handler ) {
                function handlerProxy() {
                    if ( !suppressDisabledCheck &&
                            ( instance.options.disabled === true ||
                                $( this ).hasClass( "ui-state-disabled" ) ) ) {
                        return;
                    }
                    return ( typeof handler === "string" ? instance[ handler ] : handler )
                        .apply( instance, arguments );
                }
                if ( typeof handler !== "string" ) {
                    handlerProxy.guid = handler.guid =
                        handler.guid || handlerProxy.guid || $.guid++;
                }
                var match = event.match( /^([\w:-]*)\s*(.*)$/ ),
                    eventName = match[1] + instance.eventNamespace,
                    selector = match[2];
                if ( selector ) {
                    delegateElement.delegate( selector, eventName, handlerProxy );
                } else {
                    element.bind( eventName, handlerProxy );
                }
            });
        },
        //事件解绑
        _off: function( element, eventName ) {
            eventName = (eventName || "").split( " " ).join( this.eventNamespace + " " ) +
                this.eventNamespace;
            element.unbind( eventName ).undelegate( eventName );
    
            // Clear the stack to avoid memory leaks (#10056)
            this.bindings = $( this.bindings.not( element ).get() );
            this.focusable = $( this.focusable.not( element ).get() );
            this.hoverable = $( this.hoverable.not( element ).get() );
        },
        //事件置于队列尾
        _delay: function( handler, delay ) {
            function handlerProxy() {
                return ( typeof handler === "string" ? instance[ handler ] : handler )
                    .apply( instance, arguments );
            }
            var instance = this;
            return setTimeout( handlerProxy, delay || 0 );
        },
        //hover方法
        _hoverable: function( element ) {
            this.hoverable = this.hoverable.add( element );
            this._on( element, {
                mouseenter: function( event ) {
                    $( event.currentTarget ).addClass( "ui-state-hover" );
                },
                mouseleave: function( event ) {
                    $( event.currentTarget ).removeClass( "ui-state-hover" );
                }
            });
        },
        //focus方法
        _focusable: function( element ) {
            this.focusable = this.focusable.add( element );
            this._on( element, {
                focusin: function( event ) {
                    $( event.currentTarget ).addClass( "ui-state-focus" );
                },
                focusout: function( event ) {
                    $( event.currentTarget ).removeClass( "ui-state-focus" );
                }
            });
        },
        //事件触发
        _trigger: function( type, event, data ) {
            var prop, orig,
                callback = this.options[ type ];
    
            data = data || {};
            event = $.Event( event );
            event.type = ( type === this.widgetEventPrefix ?
                type :
                this.widgetEventPrefix + type ).toLowerCase();
            event.target = this.element[ 0 ];
            orig = event.originalEvent;
            if ( orig ) {
                for ( prop in orig ) {
                    if ( !( prop in event ) ) {
                        event[ prop ] = orig[ prop ];
                    }
                }
            }
            this.element.trigger( event, data );
            return !( $.isFunction( callback ) &&
                callback.apply( this.element[0], [ event ].concat( data ) ) === false ||
                event.isDefaultPrevented() );
        }
    };


注一：**`_create`入口方法。**
我们在最开始举例用来定义控件时除了配置`options`还写了一个`_create`方法，这个方法就是控件初始化方法。
初始化过程：
jquery元素第一次调用控件（必须是配置方式）——实例化指定控件——执行内置`_createWidget`——执行`_create`——存储该控件对象于jquery元素之上。

注二：**内置的option方法根据调用方式有多种配置数值的的方式。**
1. `option()`：无参：执行`[[getter]]`，返回当前的`options`
2. `option(keyString)`：单参且参数为`string`型：**查询**`options`中`key`属性的值，如果key是`obj.obj2.obj3.attr`式字符串，则按照对象属性查询来返回`options`深层属性的值。
3. `option(keyString,value)`：双参：同单参。**设定**`key`字符串所指向的属性的值。
4. `option(keyObject)`：参数第一个是object就行。数量无所谓：将这个object深合并进当前的`options`中。

注三：**提供样式控制与方法增强。**
某些控件，比如最原生的`button`，都是有实实在在的`disabled`属性的。我们用`jqueryUI`的`button`方法时通过设定`options`或是直接使用`disable()`,`enable()`方法自然可以将其状态改变。
但是大部分自定义控件并没有什么`disabled`状态，我们只能通过简单的`css`类控制来表示我们做过了“disable”或“enable”操作。
我们翻开`jqueryUI`的`button`控件的源码。其`_setOption`方法中关于设定`disabled`参数的部分是如此实现的：
    
    _setOption: function( key, value ) {
        this._super( key, value ); //执行父类_setOption方法
        if ( key === "disabled" ) {
            this._toggleClass( null, "ui-state-disabled", value );
            this.element[ 0 ].disabled = value; //设定原dom元素disable状态
            if ( value ) {
                this.element.blur(); //并触发失焦
            }
        }
    }

没错，`button`控件在widget基础控件的基础上增强了`_setOption`的方法，使其能真正实现控件的disable状态切换。
其他未修改基础方法的控件涉及“disabled”属性的操作，都是简单的执行css样式控制而已。


### 小结

看完源码后我们再次想下widget特性和使用方法，是否已经解决了我们所有的疑问。
1. **状态维持**：本质是因为`options`和方法实例化之后都储存在了jq对象的data里面，所以能一直维持在页面之中，只有触发其`destory()`方法才能确实移除（dom元素`remove`时会调用`destory()`）。
2. **控件继承**：每个控件都是从父控件继承而来，没有指定则继承自`$.Wedget`。继承时每个类的原型都是深复制出一个新对象。
3. **内置API一致性**：比较多的公共方法有`option`,`enable`,`disable`,`destory`,`widget`,这些都是可以直接在外面插件调用的，而内置API如`_create`，`_init`都是暴露出来让开发者自定义插件时使用的，当然，定义控件时可以所以所欲的修改几乎所有widget特性。
4. **命名空间管理**：通过对定义时的参数分割来划分命名空间。查看`$[ namespace ]`，可以轻松看到其下定义的所有控件。

`JqueryUI`近年发展并不甚理想，新的UI系框架层不出穷。
但她确实是无比强大，**基于其你甚至只需要在jquery的基础上引入仅几百行的widget源码就能构造复杂UI系统**。

理解核心的widget的实现方法能帮助你提升**js水平**和**对jquery框架体系的认知**，
有助于js开发者**框架思想的快速成型**。

写这篇解析本意是和大家共同进步。以上编写匆忙，如有错误请斧正。