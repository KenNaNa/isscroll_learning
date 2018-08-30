# isscroll_learning
# iscroll.js的学习

# better-scroll 源码学习

https://www.imooc.com/article/20502?block_id=tuijian_wz


<p>
  iscroll.js源码分析
<pre>
/*! iScroll v5.1.2 ~ (c) 2008-2014 Matteo Spinelli ~ http://cubiq.org/license */
(function (window, document, Math) {
    //请求动画帧
var rAF = window.requestAnimationFrame    ||
    window.webkitRequestAnimationFrame    ||
    window.mozRequestAnimationFrame        ||
    window.oRequestAnimationFrame        ||
    window.msRequestAnimationFrame        ||
    function (callback) { window.setTimeout(callback, 1000 / 60); };
//显示器16.7ms刷新间隔之前发生了其他绘制请求(setTimeout)，导致所有帧丢失，setTimeout的定时器值推荐最小使用16.7ms的原因（16.7 = 1000 / 60, 即每秒60帧）
//requestAnimationFrame与setTimeout的不同在于，前者跟着浏览器重绘时间走，不存在后者丢帧的问题
var utils = (function () {
    var me = {};

    var _elementStyle = document.createElement('div').style;
    //获取浏览器特性前缀
    var _vendor = (function () {
        var vendors = ['t', 'webkitT', 'MozT', 'msT', 'OT'],
            transform,
            i = 0,
            l = vendors.length;

        for ( ; i < l; i++ ) {
            transform = vendors[i] + 'ransform';
            if ( transform in _elementStyle ) return vendors[i].substr(0, vendors[i].length-1);
        }

        return false;
    })();
    //拼装样式属性名
    function _prefixStyle (style) {
        if ( _vendor === false ) return false;
        if ( _vendor === '' ) return style;
        return _vendor + style.charAt(0).toUpperCase() + style.substr(1);
    }
    //获取当前时间
    me.getTime = Date.now || function getTime () { return new Date().getTime(); };
    //对象复制
    me.extend = function (target, obj) {
        for ( var i in obj ) {
            target[i] = obj[i];
        }
    };
    //事件
    me.addEvent = function (el, type, fn, capture) {
        el.addEventListener(type, fn, !!capture);
    };

    me.removeEvent = function (el, type, fn, capture) {
        el.removeEventListener(type, fn, !!capture);
    };
    //MSPointerEvent 指针事件是一些事件和相关接口，用于处理来自鼠标、手写笔或触摸屏等设备的硬件不可知的指针输入，IE10引入，IE11去掉前缀，其他浏览器都不支持指针事件。
    me.prefixPointerEvent = function (pointerEvent) {
        return window.MSPointerEvent ? 
            'MSPointer' + pointerEvent.charAt(9).toUpperCase() + pointerEvent.substr(10):
            pointerEvent;
    };
    //deceleration减速
    /**
     * 根据我们的拖动返回运动的长度与耗时，用于惯性拖动判断
     * @param current 当前鼠标位置
     * @param start touchStart时候记录的Y（可能是X）的开始位置，但是在touchmove时候可能被重写
     * @param time touchstart到手指离开时候经历的时间，同样可能被touchmove重写
     * @param lowerMargin y可移动的最大距离，这个一般为计算得出 this.wrapperHeight - this.scrollerHeight
     * @param wrapperSize 如果有边界距离的话就是可拖动，不然碰到0的时候便停止
     * @param deceleration 匀减速
     * @returns {{destination: number, duration: number}}
     */
    me.momentum = function (current, start, time, lowerMargin, wrapperSize, deceleration) {
        var distance = current - start,
            speed = Math.abs(distance) / time,
            destination,
            duration;
        //减速变量
        deceleration = deceleration === undefined ? 0.0006 : deceleration;
        //减速路程
        destination = current + ( speed * speed ) / ( 2 * deceleration ) * ( distance < 0 ? -1 : 1 );
        //持续时间
        duration = speed / deceleration;

        if ( destination < lowerMargin ) {
            destination = wrapperSize ? lowerMargin - ( wrapperSize / 2.5 * ( speed / 8 ) ) : lowerMargin;
            distance = Math.abs(destination - current);
            duration = distance / speed;
        } else if ( destination > 0 ) {
            destination = wrapperSize ? wrapperSize / 2.5 * ( speed / 8 ) : 0;
            distance = Math.abs(current) + destination;
            duration = distance / speed;
        }

        return {
            destination: Math.round(destination),
            duration: duration
        };
    };

    var _transform = _prefixStyle('transform');

    me.extend(me, {
        hasTransform: _transform !== false,
        hasPerspective: _prefixStyle('perspective') in _elementStyle, //透视
        hasTouch: 'ontouchstart' in window,
        hasPointer: window.PointerEvent || window.MSPointerEvent, // IE10 is prefixed
        hasTransition: _prefixStyle('transition') in _elementStyle
    });

    // This should find all Android browsers lower than build 535.19 (both stock browser and webview) 低版本安卓机
    me.isBadAndroid = /Android /.test(window.navigator.appVersion) && !(/Chrome\/\d/.test(window.navigator.appVersion));

    me.extend(me.style = {}, {
        transform: _transform,
        transitionTimingFunction: _prefixStyle('transitionTimingFunction'),
        transitionDuration: _prefixStyle('transitionDuration'),
        transitionDelay: _prefixStyle('transitionDelay'),
        transformOrigin: _prefixStyle('transformOrigin')
    });

    me.hasClass = function (e, c) {
        var re = new RegExp("(^|\\s)" + c + "(\\s|$)");
        return re.test(e.className);
    };

    me.addClass = function (e, c) {
        if ( me.hasClass(e, c) ) {
            return;
        }

        var newclass = e.className.split(' ');
        newclass.push(c);
        e.className = newclass.join(' ');
    };

    me.removeClass = function (e, c) {
        if ( !me.hasClass(e, c) ) {
            return;
        }

        var re = new RegExp("(^|\\s)" + c + "(\\s|$)", 'g');
        e.className = e.className.replace(re, ' ');
    };

    me.offset = function (el) {
        var left = -el.offsetLeft,
            top = -el.offsetTop;

        // jshint -W084
        while (el = el.offsetParent) {
            left -= el.offsetLeft;
            top -= el.offsetTop;
        }
        // jshint +W084

        return {
            left: left,
            top: top
        };
    };
    // 配合config里面的preventDefaultException属性，不对匹配到的element使用e.preventDefault()。
    me.preventDefaultException = function (el, exceptions) {
        for ( var i in exceptions ) {
            if ( exceptions[i].test(el[i]) ) {
                return true;
            }
        }

        return false;
    };
    // 事件类型
    me.extend(me.eventType = {}, {
        touchstart: 1,
        touchmove: 1,
        touchend: 1,

        mousedown: 2,
        mousemove: 2,
        mouseup: 2,

        pointerdown: 3,
        pointermove: 3,
        pointerup: 3,

        MSPointerDown: 3,
        MSPointerMove: 3,
        MSPointerUp: 3
    });
    // 缓动函数
    me.extend(me.ease = {}, {
        quadratic: {
            style: 'cubic-bezier(0.25, 0.46, 0.45, 0.94)',
            fn: function (k) {
                return k * ( 2 - k );
            }
        },
        circular: {
            style: 'cubic-bezier(0.1, 0.57, 0.1, 1)',    // Not properly "circular" but this looks better, it should be (0.075, 0.82, 0.165, 1)
            fn: function (k) {
                return Math.sqrt( 1 - ( --k * k ) );
            }
        },
        back: {
            style: 'cubic-bezier(0.175, 0.885, 0.32, 1.275)',
            fn: function (k) {
                var b = 4;
                return ( k = k - 1 ) * k * ( ( b + 1 ) * k + b ) + 1;
            }
        },
        bounce: {
            style: '',
            fn: function (k) {
                if ( ( k /= 1 ) < ( 1 / 2.75 ) ) {
                    return 7.5625 * k * k;
                } else if ( k < ( 2 / 2.75 ) ) {
                    return 7.5625 * ( k -= ( 1.5 / 2.75 ) ) * k + 0.75;
                } else if ( k < ( 2.5 / 2.75 ) ) {
                    return 7.5625 * ( k -= ( 2.25 / 2.75 ) ) * k + 0.9375;
                } else {
                    return 7.5625 * ( k -= ( 2.625 / 2.75 ) ) * k + 0.984375;
                }
            }
        },
        elastic: {
            style: '',
            fn: function (k) {
                var f = 0.22,
                    e = 0.4;

                if ( k === 0 ) { return 0; }
                if ( k == 1 ) { return 1; }

                return ( e * Math.pow( 2, - 10 * k ) * Math.sin( ( k - f / 4 ) * ( 2 * Math.PI ) / f ) + 1 );
            }
        }
    });

    me.tap = function (e, eventName) {
        var ev = document.createEvent('Event');
        /**
        只有在新创建的 Event 对象被 Document 对象或 Element 对象的 dispatchEvent() 方法分派之前，才能调用 Event.initEvent() 方法        
        初始化新事件对象的属性
        */
        ev.initEvent(eventName, true, true);
        ev.pageX = e.pageX;
        ev.pageY = e.pageY;
        //事件触发器就是用来触发某个元素下的某个事件
        e.target.dispatchEvent(ev);
    };

    me.click = function (e) {
        var target = e.target,
            ev;
        //不是select/input/textarea就创建事件
        if ( !(/(SELECT|INPUT|TEXTAREA)/i).test(target.tagName) ) {
            ev = document.createEvent('MouseEvents');
            ev.initMouseEvent('click', true, true, e.view, 1,
                target.screenX, target.screenY, target.clientX, target.clientY,
                e.ctrlKey, e.altKey, e.shiftKey, e.metaKey,
                0, null);

            ev._constructed = true;
            target.dispatchEvent(ev);
        }
    };

    return me;
})();

function IScroll (el, options) {
    //传入的元素 必须要有子元素
    this.wrapper = typeof el == 'string' ? document.querySelector(el) : el;
    //子元素 实际滑动元素
    this.scroller = this.wrapper.children[0];
    this.scrollerStyle = this.scroller.style;        // cache style for better performance

    this.options = {

// INSERT POINT: OPTIONS 

        startX: 0,
        startY: 0,
        //默认是Y轴上下滚动
                scrollY: true,
                //方向锁定阈值，比如用户点击屏幕后，x与y之间差距大于5px，判断用户的拖动意图，是x方向拖动还是y方向
                directionLockThreshold: 5,
                //是否有惯性缓冲动画
        momentum: true,
        //超出边界时候是否还能拖动
                bounce: true,
                //超出边界还原时间点
                bounceTime: 600,
                //超出边界返回的动画
                bounceEasing: '',
                //是否阻止默认滚动事件
                preventDefault: true,
                //当遇到表单元素则不阻止冒泡，而是弹出系统自带相应的输入控件
                preventDefaultException: { tagName: /^(INPUT|TEXTAREA|BUTTON|SELECT)$/ },
        //硬件合成 通过GPU做图形渲染
        HWCompositing: true,
        //使用transition
        useTransition: true,
        //使用transform
        useTransform: true
    };

    for ( var i in options ) {
        this.options[i] = options[i];
    }

    // Normalize options
    this.translateZ = this.options.HWCompositing && utils.hasPerspective ? ' translateZ(0)' : '';

    this.options.useTransition = utils.hasTransition && this.options.useTransition;
    this.options.useTransform = utils.hasTransform && this.options.useTransform;
    
    // 默认false，当你想保留原生垂直滚动但能够添加水平iscroll时候，设置为true，iscroll区域水平滑动时候的touch垂直移动，不会触发原生的垂直滑动
    // 通常设置为：{eventPassthrough:true,scrollX:true,scrollY:false}
    // 
    this.options.eventPassthrough = this.options.eventPassthrough === true ? 'vertical' : this.options.eventPassthrough;
    //当设置了eventPassthrough时候，不阻止默认的滚动事件
    this.options.preventDefault = !this.options.eventPassthrough && this.options.preventDefault;

    // If you want eventPassthrough I have to lock one of the axes
    // eventPassthrough可以传入具体名字(vertical/horizontal)，这里更好说明了eventPassthrough的作用，对于传入的值来阻止相应轴的滑动事件
    // eventPassthrough -> true -> vertical -> scrollY : false 垂直滚动忽略
    this.options.scrollY = this.options.eventPassthrough == 'vertical' ? false : this.options.scrollY;
    this.options.scrollX = this.options.eventPassthrough == 'horizontal' ? false : this.options.scrollX;

    // With eventPassthrough we also need lockDirection mechanism
    // freeScroll只能与scrollX一起使用 {scrollX:true,freeScroll:true}
    this.options.freeScroll = this.options.freeScroll && !this.options.eventPassthrough;
    this.options.directionLockThreshold = this.options.eventPassthrough ? 0 : this.options.directionLockThreshold;

    // 缓动函数
    this.options.bounceEasing = typeof this.options.bounceEasing == 'string' ? utils.ease[this.options.bounceEasing] || utils.ease.circular : this.options.bounceEasing;
    
    //window resize时重新获取位置
    this.options.resizePolling = this.options.resizePolling === undefined ? 60 : this.options.resizePolling;

    if ( this.options.tap === true ) {
        this.options.tap = 'tap';
    }

// INSERT POINT: NORMALIZATION

    // Some defaults    
    this.x = 0;
    this.y = 0;
    this.directionX = 0;
    this.directionY = 0;
    this._events = {};

// INSERT POINT: DEFAULTS
    // IScroll初始化
    this._init();
    // 
    this.refresh();

    // 定位到最顶最左
    this.scrollTo(this.options.startX, this.options.startY);
    // 代表“启用”的一个标志
    this.enable();
}

IScroll.prototype = {
    version: '5.1.2',

    _init: function () {
        this._initEvents();

// INSERT POINT: _init

    },

    destroy: function () {
        this._initEvents(true);

        this._execEvent('destroy');
    },

    _transitionEnd: function (e) {
        if ( e.target != this.scroller || !this.isInTransition ) {
            return;
        }

        this._transitionTime();
        if ( !this.resetPosition(this.options.bounceTime) ) {
            this.isInTransition = false;
            this._execEvent('scrollEnd');
        }
    },

    _start: function (e) {
        // React to left mouse button only
        // 不是已经注册的事件，而且按下的不是左键，就返回不处理。
        if ( utils.eventType[e.type] != 1 ) {
            if ( e.button !== 0 ) {
                return;
            }
        }
        // 没有启用，返回不处理。
        if ( !this.enabled || (this.initiated && utils.eventType[e.type] !== this.initiated) ) {
            return;
        }
        // 如果 preventDefault === true 且 不是落后的安卓版本 且 不是需要过滤的target 就 阻止默认的行为
        if ( this.options.preventDefault && !utils.isBadAndroid && !utils.preventDefaultException(e.target, this.options.preventDefaultException) ) {
            e.preventDefault();
        }
        //e.touches表示的在屏幕上所有的触摸点，但事实上，绝大数手机浏览器并不支持多点触摸，所有用e.touchees[0]捕获一个触点就知足吧，
        //不要再奢望获取e.touches[>0]了，这个触点的位置可以有e.touches[0].pageX获取页面x坐标（像素）；
        var point = e.touches ? e.touches[0] : e,
            pos;
        // 事件类型
        this.initiated    = utils.eventType[e.type];
        this.moved    = false;    //是否移动的标志
        this.distX    = 0;        //
        this.distY    = 0;        //
        this.directionX = 0;    //x方向移动数
        this.directionY = 0;    //y方向移动数
        this.directionLocked = 0;    //方向锁
        // 设置运动时间
        this._transitionTime();
        // 开始时间
        this.startTime = utils.getTime();

        // 如果还在运动中
        if ( this.options.useTransition && this.isInTransition ) {
            this.isInTransition = false;
            // 获取当前位置
            pos = this.getComputedPosition();
            // 滑动到当前位置 相当于停止于此处
            this._translate(Math.round(pos.x), Math.round(pos.y));
            // 触发滑动结束回调函数
            this._execEvent('scrollEnd');
        } else if ( !this.options.useTransition && this.isAnimating ) {
            this.isAnimating = false;
            this._execEvent('scrollEnd');
        }

        this.startX    = this.x;        // scroller开始位置x
        this.startY    = this.y;        // scroller开始位置y
        this.absStartX = this.x;        //
        this.absStartY = this.y;        //
        this.pointX    = point.pageX;    // 触点x
        this.pointY    = point.pageY;    // 触点y
        // 触发滑动前回调函数
        this._execEvent('beforeScrollStart');
    },

    _move: function (e) {
        // 禁止 or 不存在此eventType 则返回
        if ( !this.enabled || utils.eventType[e.type] !== this.initiated ) {
            return;
        }

        if ( this.options.preventDefault ) {    // increases performance on Android? TODO: check!
            e.preventDefault();
        }
        // point 触点
        var point        = e.touches ? e.touches[0] : e,
            deltaX        = point.pageX - this.pointX,    // 当前触点的pagex - 开始时的pagex = 触点档次增量x
            deltaY        = point.pageY - this.pointY,    // 触点增量y
            timestamp    = utils.getTime(),
            newX, newY,
            absDistX, absDistY;
        // 最近上一次的触点位置
        this.pointX        = point.pageX;
        this.pointY        = point.pageY;

        // 触点移动的距离
        this.distX        += deltaX;
        this.distY        += deltaY;
        absDistX        = Math.abs(this.distX);
        absDistY        = Math.abs(this.distY);

        // We need to move at least 10 pixels for the scrolling to initiate
        // 触点至少移动10px才会触发scroll的move 并且 移动大于300ms
        if ( timestamp - this.endTime > 300 && (absDistX < 10 && absDistY < 10) ) {
            return;
        }

        // If you are scrolling in one direction lock the other
        // 除非设置了freeScroll，否则将只允许同一时间一个方向滑动
        if ( !this.directionLocked && !this.options.freeScroll ) {
            if ( absDistX > absDistY + this.options.directionLockThreshold ) {
                this.directionLocked = 'h';        // lock horizontally
            } else if ( absDistY >= absDistX + this.options.directionLockThreshold ) {
                this.directionLocked = 'v';        // lock vertically
            } else {
                this.directionLocked = 'n';        // no lock
            }
        }

        if ( this.directionLocked == 'h' ) {
            if ( this.options.eventPassthrough == 'vertical' ) {
                e.preventDefault();
            } else if ( this.options.eventPassthrough == 'horizontal' ) {
                this.initiated = false;
                return;
            }

            deltaY = 0;
        } else if ( this.directionLocked == 'v' ) {
            if ( this.options.eventPassthrough == 'horizontal' ) {
                e.preventDefault();
            } else if ( this.options.eventPassthrough == 'vertical' ) {
                this.initiated = false;
                return;
            }

            deltaX = 0;
        }

        deltaX = this.hasHorizontalScroll ? deltaX : 0;
        deltaY = this.hasVerticalScroll ? deltaY : 0;
        // this.x this.y 是最近上一次的scroller位置
        newX = this.x + deltaX;
        newY = this.y + deltaY;

        // Slow down if outside of the boundaries
        if ( newX > 0 || newX < this.maxScrollX ) {
            newX = this.options.bounce ? this.x + deltaX / 3 : newX > 0 ? 0 : this.maxScrollX;
        }
        if ( newY > 0 || newY < this.maxScrollY ) {
            newY = this.options.bounce ? this.y + deltaY / 3 : newY > 0 ? 0 : this.maxScrollY;
        }

        this.directionX = deltaX > 0 ? -1 : deltaX < 0 ? 1 : 0;    // -1 手势向左   1 手势向右
        this.directionY = deltaY > 0 ? -1 : deltaY < 0 ? 1 : 0; // -1 手势向上   1 手势向下

        if ( !this.moved ) {
            this._execEvent('scrollStart');
        }

        this.moved = true;

        this._translate(newX, newY);

/* REPLACE START: _move */

        if ( timestamp - this.startTime > 300 ) {
            //300ms更新一次
            this.startTime = timestamp;
            this.startX = this.x;
            this.startY = this.y;
        }

/* REPLACE END: _move */

    },
    // touchEnd时候处理缓动
    _end: function (e) {
        if ( !this.enabled || utils.eventType[e.type] !== this.initiated ) {
            return;
        }

        if ( this.options.preventDefault && !utils.preventDefaultException(e.target, this.options.preventDefaultException) ) {
            e.preventDefault();
        }

        var point = e.changedTouches ? e.changedTouches[0] : e,
            momentumX,
            momentumY,
            duration = utils.getTime() - this.startTime,
            newX = Math.round(this.x),
            newY = Math.round(this.y),
            distanceX = Math.abs(newX - this.startX),    //单次移动过程的x轴移动距离
            distanceY = Math.abs(newY - this.startY),
            time = 0,
            easing = '';

        this.isInTransition = 0;
        this.initiated = 0;
        this.endTime = utils.getTime();

        // reset if we are outside of the boundaries
        // 超过了边界就重新回到边界位
        if ( this.resetPosition(this.options.bounceTime) ) {
            return;
        }
        // 坚持滑动到
        this.scrollTo(newX, newY);    // ensures that the last position is rounded

        // we scrolled less than 10 pixels
        if ( !this.moved ) {
            if ( this.options.tap ) {
                utils.tap(e, this.options.tap);
            }

            if ( this.options.click ) {
                utils.click(e);
            }

            this._execEvent('scrollCancel');
            return;
        }

        if ( this._events.flick && duration < 200 && distanceX < 100 && distanceY < 100 ) {
            this._execEvent('flick');
            return;
        }

        // start momentum animation if needed
        // 获取缓动的距离 和 时间 { destination ， duration }
        if ( this.options.momentum && duration < 300 ) {
            momentumX = this.hasHorizontalScroll ? utils.momentum(this.x, this.startX, duration, this.maxScrollX, this.options.bounce ? this.wrapperWidth : 0, this.options.deceleration) : { destination: newX, duration: 0 };
            momentumY = this.hasVerticalScroll ? utils.momentum(this.y, this.startY, duration, this.maxScrollY, this.options.bounce ? this.wrapperHeight : 0, this.options.deceleration) : { destination: newY, duration: 0 };
            newX = momentumX.destination;
            newY = momentumY.destination;
            time = Math.max(momentumX.duration, momentumY.duration);
            this.isInTransition = 1;
        }

// INSERT POINT: _end

        if ( newX != this.x || newY != this.y ) {
            // change easing function when scroller goes out of the boundaries
            if ( newX > 0 || newX < this.maxScrollX || newY > 0 || newY < this.maxScrollY ) {
                easing = utils.ease.quadratic;
            }

            this.scrollTo(newX, newY, time, easing);
            return;
        }

        this._execEvent('scrollEnd');
    },

    _resize: function () {
        var that = this;

        clearTimeout(this.resizeTimeout);

        this.resizeTimeout = setTimeout(function () {
            that.refresh();
        }, this.options.resizePolling);
    },
    /**
     * 重新定位
     */
    resetPosition: function (time) {
        var x = this.x,
            y = this.y;

        time = time || 0;
        
        /**
        * 如果禁止水平滑动或者x大于0，x=0。如果可以滑动的情况，x应该是小于0的。
        * 否则（可水平滑动），x为maxScrollX即最左
        */
        if ( !this.hasHorizontalScroll || this.x > 0 ) {
            x = 0;
        } else if ( this.x < this.maxScrollX ) {
            x = this.maxScrollX;
        }
        /**
        * 如果禁止垂直滑动或者y大于0，y=0。如果可以滑动的情况，y应该是小于0的。
        * 否则（可垂直滑动），y为maxScrollY即最上
        */
        if ( !this.hasVerticalScroll || this.y > 0 ) {
            y = 0;
        } else if ( this.y < this.maxScrollY ) {
            y = this.maxScrollY;
        }

        // 初始化时候x y 都是0 返回。
        if ( x == this.x && y == this.y ) {
            return false;
        }

        this.scrollTo(x, y, time, this.options.bounceEasing);

        return true;
    },
    /**
     * scroll关闭的标志
     */
    disable: function () {
        this.enabled = false;
    },

    /**
     * scroll启用的标志
     */
    enable: function () {
        this.enabled = true;
    },

    /**
     * 重新获取位置等参数信息
     */
    refresh: function () {
        
        var rf = this.wrapper.offsetHeight;        // Force reflow
        // 获取包含块的宽度/高度
        this.wrapperWidth    = this.wrapper.clientWidth;
        this.wrapperHeight    = this.wrapper.clientHeight;

/* REPLACE START: refresh */
        // 滑动区的高度/宽度
        this.scrollerWidth    = this.scroller.offsetWidth;
        this.scrollerHeight    = this.scroller.offsetHeight;
        // 最多允许滑动的x/y
        this.maxScrollX        = this.wrapperWidth - this.scrollerWidth;
        this.maxScrollY        = this.wrapperHeight - this.scrollerHeight;

/* REPLACE END: refresh */
        // 是否可以垂直/水平滑动
        this.hasHorizontalScroll    = this.options.scrollX && this.maxScrollX < 0;
        this.hasVerticalScroll        = this.options.scrollY && this.maxScrollY < 0;

        if ( !this.hasHorizontalScroll ) {
            this.maxScrollX = 0;
            this.scrollerWidth = this.wrapperWidth;
        }

        if ( !this.hasVerticalScroll ) {
            this.maxScrollY = 0;
            this.scrollerHeight = this.wrapperHeight;
        }

        this.endTime = 0;
        this.directionX = 0;
        this.directionY = 0;
        
        // 获取包含块的offsetLeft offsetTop
        this.wrapperOffset = utils.offset(this.wrapper);

        this._execEvent('refresh');

        this.resetPosition();

// INSERT POINT: _refresh

    },
    /**
     * 自定义事件三件套，不解释。
     */
    on: function (type, fn) {
        if ( !this._events[type] ) {
            this._events[type] = [];
        }

        this._events[type].push(fn);
    },

    off: function (type, fn) {
        if ( !this._events[type] ) {
            return;
        }

        var index = this._events[type].indexOf(fn);

        if ( index > -1 ) {
            this._events[type].splice(index, 1);
        }
    },

    _execEvent: function (type) {
        if ( !this._events[type] ) {
            return;
        }

        var i = 0,
            l = this._events[type].length;

        if ( !l ) {
            return;
        }

        for ( ; i < l; i++ ) {
            this._events[type][i].apply(this, [].slice.call(arguments, 1));
        }
    },

    scrollBy: function (x, y, time, easing) {
        x = this.x + x;
        y = this.y + y;
        time = time || 0;

        this.scrollTo(x, y, time, easing);
    },

    /**
     *
     * @param  x 为移动的x轴坐标
     * @param y 为移动的y轴坐标
     * @param time 为移动时间
     * @param easing 为移动的动画效果
     */
    scrollTo: function (x, y, time, easing) {
        easing = easing || utils.ease.circular;

        this.isInTransition = this.options.useTransition && time > 0;

        if ( !time || (this.options.useTransition && easing.style) ) {
            this._transitionTimingFunction(easing.style);
            this._transitionTime(time);
            this._translate(x, y);
        } else {
            this._animate(x, y, time, easing.fn);
        }
    },

    /**
     * 这个方法实际上是对scrollTo的进一步封装,滚动到相应的元素区域。
     * @param el 为需要滚动到的元素引用
     * @param time 为滚动时间
     * @param offsetX 为X轴偏移量
     * @param offsetY 为Y轴偏移量
     * @param easing 动画效果
     */
    scrollToElement: function (el, time, offsetX, offsetY, easing) {
        el = el.nodeType ? el : this.scroller.querySelector(el);

        if ( !el ) {
            return;
        }

        var pos = utils.offset(el);

        pos.left -= this.wrapperOffset.left;
        pos.top  -= this.wrapperOffset.top;

        // if offsetX/Y are true we center the element to the screen
        if ( offsetX === true ) {
            offsetX = Math.round(el.offsetWidth / 2 - this.wrapper.offsetWidth / 2);
        }
        if ( offsetY === true ) {
            offsetY = Math.round(el.offsetHeight / 2 - this.wrapper.offsetHeight / 2);
        }

        pos.left -= offsetX || 0;
        pos.top  -= offsetY || 0;

        pos.left = pos.left > 0 ? 0 : pos.left < this.maxScrollX ? this.maxScrollX : pos.left;
        pos.top  = pos.top  > 0 ? 0 : pos.top  < this.maxScrollY ? this.maxScrollY : pos.top;

        time = time === undefined || time === null || time === 'auto' ? Math.max(Math.abs(this.x-pos.left), Math.abs(this.y-pos.top)) : time;

        this.scrollTo(pos.left, pos.top, time, easing);
    },
    // 动画运动时间
    _transitionTime: function (time) {
        time = time || 0;

        this.scrollerStyle[utils.style.transitionDuration] = time + 'ms';
        // 落后的安卓机运动時間需要把0转0.001s
        if ( !time && utils.isBadAndroid ) {
            this.scrollerStyle[utils.style.transitionDuration] = '0.001s';
        }

// INSERT POINT: _transitionTime

    },

    _transitionTimingFunction: function (easing) {
        this.scrollerStyle[utils.style.transitionTimingFunction] = easing;

// INSERT POINT: _transitionTimingFunction

    },

    /**
     * @param  x 为移动的x轴坐标
     * @param  y 为移动的y轴坐标
     */
    _translate: function (x, y) {
        if ( this.options.useTransform ) {

/* REPLACE START: _translate */

            this.scrollerStyle[utils.style.transform] = 'translate(' + x + 'px,' + y + 'px)' + this.translateZ;

/* REPLACE END: _translate */

        } else {
            x = Math.round(x);
            y = Math.round(y);
            this.scrollerStyle.left = x + 'px';
            this.scrollerStyle.top = y + 'px';
        }

        this.x = x;
        this.y = y;

// INSERT POINT: _translate

    },

    /**
     * @param  remove 是否解绑事件
     */
    _initEvents: function (remove) {
        var eventType = remove ? utils.removeEvent : utils.addEvent,
            // 是否绑定到wrapper元素上
            target = this.options.bindToWrapper ? this.wrapper : window;
        //旋转屏幕事件
        eventType(window, 'orientationchange', this);
        eventType(window, 'resize', this);

        if ( this.options.click ) {
            eventType(this.wrapper, 'click', this, true);
        }
        // 如果没禁用鼠标，绑定鼠标事件，针对PC。
        if ( !this.options.disableMouse ) {
            eventType(this.wrapper, 'mousedown', this);
            eventType(target, 'mousemove', this);
            eventType(target, 'mousecancel', this);
            eventType(target, 'mouseup', this);
        }
        // 针对win phone的touch事件
        if ( utils.hasPointer && !this.options.disablePointer ) {
            eventType(this.wrapper, utils.prefixPointerEvent('pointerdown'), this);
            eventType(target, utils.prefixPointerEvent('pointermove'), this);
            eventType(target, utils.prefixPointerEvent('pointercancel'), this);
            eventType(target, utils.prefixPointerEvent('pointerup'), this);
        }
        //针对ios & Android的touch事件
        if ( utils.hasTouch && !this.options.disableTouch ) {
            eventType(this.wrapper, 'touchstart', this);
            eventType(target, 'touchmove', this);
            eventType(target, 'touchcancel', this);
            eventType(target, 'touchend', this);
        }
        // css3动画结束后的回调事件
        eventType(this.scroller, 'transitionend', this);
        eventType(this.scroller, 'webkitTransitionEnd', this);
        eventType(this.scroller, 'oTransitionEnd', this);
        eventType(this.scroller, 'MSTransitionEnd', this);
    },

    getComputedPosition: function () {
        //获取scroller的样式
        var matrix = window.getComputedStyle(this.scroller, null),
            x, y;
        //两种方式 transform 和 top/left ( transform在ios下,文本输入的光标fixed )
        if ( this.options.useTransform ) {
            matrix = matrix[utils.style.transform].split(')')[0].split(', ');
            x = +(matrix[12] || matrix[4]);
            y = +(matrix[13] || matrix[5]);
        } else {
            x = +matrix.left.replace(/[^-\d.]/g, '');
            y = +matrix.top.replace(/[^-\d.]/g, '');
        }
        //返回 (x ,y)  or (top ,left)
        return { x: x, y: y };
    },

    /**
     * @param  destx 为移动目的地的x轴坐标
     * @param  desty 为移动目的地的y轴坐标
     * @param  duration 为移动的时间
     * @param  easingFn 为移动缓动函数
     */
    _animate: function (destX, destY, duration, easingFn) {
        var that = this,
            startX = this.x,
            startY = this.y,
            startTime = utils.getTime(),
            destTime = startTime + duration;

        function step () {
            var now = utils.getTime(),
                newX, newY,
                easing;

            if ( now >= destTime ) {
                that.isAnimating = false;
                that._translate(destX, destY);

                if ( !that.resetPosition(that.options.bounceTime) ) {
                    that._execEvent('scrollEnd');
                }

                return;
            }

            now = ( now - startTime ) / duration;
            easing = easingFn(now);
            newX = ( destX - startX ) * easing + startX;
            newY = ( destY - startY ) * easing + startY;
            that._translate(newX, newY);

            if ( that.isAnimating ) {
                rAF(step);
            }
        }

        this.isAnimating = true;
        step();
    },
    handleEvent: function (e) {
        switch ( e.type ) {
            case 'touchstart':
            case 'pointerdown':
            case 'MSPointerDown':
            case 'mousedown':
                this._start(e);
                break;
            case 'touchmove':
            case 'pointermove':
            case 'MSPointerMove':
            case 'mousemove':
                this._move(e);
                break;
            case 'touchend':
            case 'pointerup':
            case 'MSPointerUp':
            case 'mouseup':
            case 'touchcancel':
            case 'pointercancel':
            case 'MSPointerCancel':
            case 'mousecancel':
                this._end(e);
                break;
            case 'orientationchange':
            case 'resize':
                this._resize();
                break;
            case 'transitionend':
            case 'webkitTransitionEnd':
            case 'oTransitionEnd':
            case 'MSTransitionEnd':
                this._transitionEnd(e);
                break;
            case 'wheel':
            case 'DOMMouseScroll':
            case 'mousewheel':
                this._wheel(e);
                break;
            case 'keydown':
                this._key(e);
                break;
            case 'click':
                if ( !e._constructed ) {
                    e.preventDefault();
                    e.stopPropagation();
                }
                break;
        }
    }
};
IScroll.utils = utils;

if ( typeof module != 'undefined' && module.exports ) {
    module.exports = IScroll;
} else {
    window.IScroll = IScroll;
}

})(window, document, Math);

</pre>
</p>
