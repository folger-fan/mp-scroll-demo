[复杂点效果视频](http://url.cn/4EaSBuo)

# 微信小程序动画能力研究（踩坑、iscroll源码解读）
在小程序提供的基础组件外，我们还需要更丰富的动画效果

## 小程序开发痛点
- 不能直接操作dom
- 没有window等页面常用基本对象
- 部分css不支持或跟页面展示有异
- 组件不能细分，生命周期不完善
- data改变后不知道何时渲染完成
- 传统页面开发者不习惯数据驱动展示模式
- 解决了部分移动端兼容性问题但引入了新的兼容性问题


**因为小程序不能操作dom，大量的前端库没办法在小程序中使用。小程序将view和逻辑隔离开了，也不方便封装库，没办法在库内部形成闭环。**

小程序提供了一些基本组件，**但满足不了很多定制化的需求**，比如局部下拉刷新、弹性滚动。小程序中的swiper在手机上感觉有点卡顿（手指拨动的时候还好，放开手指swiper自己滑动的时候能感觉到一点卡顿）。

定制化的需求大多还得自己手写，很少有第三方库支持。如何在小程序上做定制化的动画，想到了研究iscroll，仿照其原理在小程序中实现。最终效果挺好、挺流畅。

## iscroll-lite源码研究
iscroll-lite是iscroll的简版，代码量只有后者的一半，是比较纯粹的弹性滚动的库。比较适合上手研究。

iscroll文档上有这个属性

```
options.useTransition
iScroll uses CSS transition to perform animations (momentum and bounce). By setting this to false, requestAnimationFrame is used instead.
On modern browsers the difference is barely noticeable. On older devices transitions perform better.
Default: true
```
这句话介绍了iscroll的核心原理。意思是**iscroll使用css的transition实现动画（惯性滚动和弹性滚动）**。如果useTransition设为false的话会用requestAnimationFrame替代。在现代浏览器上两者差不多的性能，在旧设备上transitions性能更优。

本人试过，在桌面浏览器上两者体验差不多的，但在笔者的两千块**安卓机上requestAnimationFrame实现动画会有卡顿，使用transition则媲美原生体验很流畅**。小程序同理。

结合iscroll的使用方法，通读源码会发现iscroll做弹性滚动的原理：
**外层wrapper固定尺寸，wrapper下scroller是滚动元素，监听scroller的touch事件，利用transform控制scroller的移动。**
在控制scroller的过程中可以定制化的实现一些效果，如弹性滚动、轮播图、定点滚动snap(iScroll can snap to fixed positions and elements.)等。

源码最上边

```
var rAF = window.requestAnimationFrame	||
	window.webkitRequestAnimationFrame	||
	window.mozRequestAnimationFrame		||
	window.oRequestAnimationFrame		||
	window.msRequestAnimationFrame		||
	function (callback) { window.setTimeout(callback, 1000 / 60); };
```
**动画一秒达到60帧才不会感到卡顿**，小程序中没有 requestAnimationFrame 可以用 window.setTimeout(callback, 1000 / 60)替代

```
    me.momentum = function (current, start, time, lowerMargin, wrapperSize, deceleration) {
		var distance = current - start,
			speed = Math.abs(distance) / time,
			destination,
			duration;

		deceleration = deceleration === undefined ? 0.0006 : deceleration;

		destination = current + ( speed * speed ) / ( 2 * deceleration ) * ( distance < 0 ? -1 : 1 );
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
```
这一段是惯性滑动的算法，手指拨动元素，放开手指元素还应该滑多远多久。

参数:
- current 当前坐标（触发move事件的坐标）
- start 起始坐标（上次move事件的坐标或start的坐标）
- time 事件间隔（此次move事件距离上次move事件或start事件的事件间隔）
- lowerMargin 最大能滚动的距离（以横向滚动为例，容器宽度（小）减去滚动元素的宽度（大），往左滑动translateX是负数且越来越小）
- wrapperSize 容器宽度
- deceleration 阻尼系数 越小越快越远

```
    me.extend(me.ease = {}, {
		quadratic: {
			style: 'cubic-bezier(0.25, 0.46, 0.45, 0.94)',
			fn: function (k) {
				return k * ( 2 - k );
			}
		},
		circular: {
			style: 'cubic-bezier(0.1, 0.57, 0.1, 1)',	// Not properly "circular" but this looks better, it should be (0.075, 0.82, 0.165, 1)
			fn: function (k) {
				return Math.sqrt( 1 - ( --k * k ) );
			}
		},
		back: {
			style: 'cubic-bezier(0.175, 0.885, 0.32, 1.275)',
			fn: function (k) {

			}
		},
		bounce: {
			style: '',
			fn: function (k) {

			}
		},
		elastic: {
			style: '',
			fn: function (k) {

			}
		}
	});
```

这一段是惯性滑动动画时间曲线的配置和算法，style是css3中transition-timing-function的属性值 (cubic-bezie 公式)。fn指的如果用 requestAnimationFrame 对应的时间曲线算法。k是时间占动画时长的比例(过了多久)。

```
    _transitionEnd: function (e) {

	},

	case 'transitionend':
			case 'webkitTransitionEnd':
				this._transitionEnd(e);
				break;
```
动画结束会有 transitionend 事件，小程序没有transitionend，替代方案是在动画时长的时间后用setTimeout进行下一步操作，如:
```
    setTimeout(function () {
        that.resetPosition(bounceTime)
    }, time)
```


下面是touch相关事件处理，代码有点多，这里不贴，可以在源码中寻找。
```
    _start(e){

    },

```
绑定滚动元素的touchstart事件，主要逻辑是取消滚动态的锁定（开始新的滚动），停止requestAnimationFrame动画，记录起始偏移距离，准备记录后续滑动距离。

```
    _move(e){

    },
```
绑定滚动元素的touchmove事件，主要逻辑是计算此次move相对上次move/start滑动距离，累加得到滚动元素偏移的距离distX并修改元素translate偏移distX（delay:0、duration:0），判断滑动方向（横向x还是竖向y）。如果超过了边界且设置了弹性滚动bounce，那么减慢偏移速度，代码如下：
```
// Slow down if outside of the boundaries
		if ( newX > 0 || newX < this.maxScrollX ) {
			newX = this.options.bounce ? this.x + deltaX / 3 : newX > 0 ? 0 : this.maxScrollX;
		}
```


```
_end: function (e) {}
```
绑定滚动元素touchend事件，主要逻辑是计算此次时间同上次move事件的间隔duration，判断是否超过边界了，没有达到边界计算惯性滚动距离和时间。

```
_translate: function (x, y) {}
```
设置滚动元素的偏移量
```
_transitionTimingFunction: function (easing) {
		this.scrollerStyle[utils.style.transitionTimingFunction] = easing;

	},
```
设置transition的动画效果（时间曲线）
```
_transitionTime: function (time) {})
```
设置transition的时长duration
```
resetPosition: function (time) {})
```
滚动结束检测是否超出边界，超出则调用 scrollTo 回滚
```
_translate: function (x, y) {})
```
设置滚动元素的偏移量，时长和动画由 _transitionTimingFunction、_transitionTime方法设置

```
_animate: function (destX, destY, duration, easingFn) {})
```
使用requestAnimationFrame调用 _translate 设置偏移量
```
scrollTo: function (x, y, time, easing) {
    easing = easing || utils.ease.circular;

		this.isInTransition = this.options.useTransition && time > 0;
		var transitionType = this.options.useTransition && easing.style;
		if ( !time || transitionType ) {
				if(transitionType) {
					this._transitionTimingFunction(easing.style);
					this._transitionTime(time);
				}
			this._translate(x, y);
		} else {
			this._animate(x, y, time, easing.fn);
		}
})
```
scrollTo是对_translate和_animate的封装，在touchend后会有惯性滚动或边界回滚的动画，这时候根据用户的设置采用_translate或_animate。_translate利用的css3的过渡动画，pc和移动端效果都不错，媲美原生。_animate则在requestAnimationFrame中计算滚动元素在指定时刻的偏移量，在移动端有点卡顿。一个是css动画一个是js实现的动画。

## 小程序动画能力
小程序提供了jsapi [wx.createAnimation](https://mp.weixin.qq.com/debug/wxadoc/dev/api/api-animation.html)控制动画。笔者根据iscroll的原理在小程序中实现弹性滚动、惯性滚动的时候一开始采用这个api控制动画。在pc上没啥问题，但在笔者的安卓机上拨动的时候略显卡顿。笔者在腾讯视频中观察拨动的动画感觉挺流畅的，遂明白wx.createAnimation做连续的动画有性能问题。后来尝试直接控制元素style中的transition、transform等样式属性控制动画，流畅度提升了一大截，媲美原生体验。

因为小程序中不能直接操作dom，只能在wxml中绑定事件，在js中setData更新页面。所以需要手动将事件绑定在元素标签上。
```
<view class="wrapper">
            <view class="scroller"  style="transition:transform {{duration/1000}}s;transform:translateZ(0) translateX({{distX}}px)"
                  bindtouchmove="_move" bindtouchstart="_start" bindtouchend="_end">



_translate: function (x, time) {
        x = Math.round(x);
        this.setData({
            distX: x,
            duration: time || 0
        });
        this.x = x;
    },
```

小程序提供了 [wx.createSelectorQuery()](https://mp.weixin.qq.com/debug/wxadoc/dev/api/wxml-nodes-info.html#nodesreffieldsfieldscallback)
可以查询元素位置、尺寸信息。因为iscroll需要知道容器尺寸、滚动元素可偏移量等信息，可以手动设置或在代码中利用wx.createSelectorQuery()计算。

在touchend后在处理惯性滚动、边界弹性滚动的时候一开始想效果更真实，采用了iscroll的_animation方法在setTimeout中利用iscroll源码的算法计算滚动元素偏移量，在pc上没问题但在笔者安卓机上感到卡顿，遂直接设置样式中的translateX/Y、duration等值将动画交给小程序（小程序上wx.createAnimation同css3过渡样式不能同时使用会有冲突）。




## demo
如下是一个横向滚动的简单demo。对代码做些改动定制化处理事件，可以做到很多iscroll能实现的效果

wxml
```
<view class="wrapper">
    <view class="scroller" bindtouchstart="_start" bindtouchmove="_move" bindtouchend="_end"
          style="width:{{scrollWidth}}px;transition: transform {{duration/1000}}s;transform:translateZ(0) translateX({{distX}}px)">
        <view wx:key="item" wx:for="{{slideBgColors}}" class="slide" style="width:{{100+index*10}}px;background:{{item}}"></view>
    </view>
</view>
```

js
```
/**
 * Created by folgerfan on 2017/8/3.
 */
var slideBgColors = 'green,gray,blueviolet,blue,rebeccapurple,yellowgreen,gold,darkslategray,brown,darkslategray'.split(',');
var deceleration = 0.005;//阻尼系数,越小越快
function momentum(current, start, time, lowerMargin, wrapperSize) {
    var distance = current - start,
        speed = Math.abs(distance) / time,
        destination,
        duration;
    destination = current + speed / deceleration * ( distance < 0 ? -1 : 1 );
    duration = speed / deceleration;

    if (destination < lowerMargin) {
        destination = wrapperSize ? lowerMargin - ( wrapperSize / 2.5 * ( speed / 8 ) ) : lowerMargin;
        distance = Math.abs(destination - current);
        duration = distance / speed;
    } else if (destination > 0) {
        destination = wrapperSize ? wrapperSize / 2.5 * ( speed / 8 ) : 0;
        distance = Math.abs(current) + destination;
        duration = distance / speed;
    }
    return {
        destination: Math.round(destination),
        duration: duration
    };
}
function getTime() {
    return +new Date()
}
var bounceTime = 500,sysInfo = wx.getSystemInfoSync();
var windowWidth = sysInfo.windowWidth;
Page({
    data: {
        slideBgColors,
        distX: 0,
        duration: 0,
        scrollWidth:100000
    },
    x: 0,
    maxScrollX: 0,
    wrapperWidth: windowWidth,

    _start(e){
        var point = e.touches[0];
        this.moved = false;
        this.distX = 0;

        this.startTime = getTime();
        this.lastMoveTime = getTime();
        this.startX = this.x;
        this.absStartX = this.x;
        this.pointX = point.pageX;
    },
    _move(e){
        var point = e.touches[0],
            deltaX = point.pageX - this.pointX,
            timestamp = +new Date(),
            newX, absDistX;
        this.pointX = point.pageX;
        this.distX += deltaX;
        absDistX = Math.abs(this.distX);
        // We need to move at least 10 pixels for the scrolling to initiate
        if (timestamp - this.endTime > 300 && absDistX < 10) {
            return;
        }
        newX = this.x + deltaX;
        // Slow down if outside of the boundaries
        if (newX > 0 || newX < this.maxScrollX) {
            newX = this.x + deltaX / 3;
        }
        this.directionX = deltaX > 0 ? -1 : deltaX < 0 ? 1 : 0;
        this.moved = true;
        this._translate(newX);
        if (timestamp - this.startTime > 300) {
            this.startTime = timestamp;
            this.startX = this.x;
            this.startY = this.y;
        }
    },
    _end(e){
        var momentumX,
            duration = getTime() - this.startTime,
            newX = Math.round(this.x),
            time = 0,
            that = this;
        this.endTime = getTime();

        if (!this.moved) {
            return;
        }
        if (this.resetPosition(bounceTime)) {
            return;
        }
        // start momentum animation if needed
        if (duration < 300 && Math.abs(this.startX - this.x) > 10) {
            momentumX = momentum(this.x, this.startX, duration, this.maxScrollX, this.wrapperWidth);
            newX = momentumX.destination;
            time = momentumX.duration;
            this._translate(newX, time);
            setTimeout(function () {
                that.resetPosition(bounceTime)
            }, time)
        }
    },
    _translate: function (x, time) {
        x = Math.round(x);
        this.setData({
            distX: x,
            duration: time || 0
        });
        this.x = x;
    },
    resetPosition: function (time) {
        var x = this.x,
            time = time || 0;
        if (this.x > 0) {
            x = 0;
        } else if (this.x < this.maxScrollX) {
            x = this.maxScrollX;
        }
        if (x == this.x) {
            return false;
        }
        this._translate(x, time);
        return true;
    },
    onReady(){
        var sum = 0,that = this;

        wx.createSelectorQuery().selectAll('.slide').boundingClientRect(function (rects) {
            //计算scroller宽度和可滚动距离
            rects.forEach(rect=>sum+=rect.width);
            that.maxScrollX = that.wrapperWidth-sum;
            that.setData({
                scrollWidth:sum
            })
        }).exec();
    }
});
```
wxss

```
.wrapper{
    width:100%;
    height:300px;
    overflow: hidden;
}
.scroller{
    height:300px;
}
.scroller .slide{
    height:300px;
    float:left;
}

```


