---
uuid: 1ff6e520-db11-11e6-b1b1-91818851584c
title: 移动端开发中的坑
date: 2016-11-04 11:33:27
updated: 2017-01-07 16:14
categories: 
   - frontend 
thumbnail: /traps-in-webview-dev/banner.jpg
---

在我目前有限的几次webview开发中,遇到过一些坑,现在写下来,也不知道有没有用,实际上这些坑一般遇到过一次就应该不会再跳进去了.

### 爬虫抓取文字

页面后面添加个**隐藏的p标签**，补充些正常文字，让微博等爬链接的时候抓取（它们默认抓取页面最后的文字），避免把链接文字搞得莫名其妙 

爬虫取到的页尾文字有时候会非常搞siao

### 去掉默认点击效果

在css中可以用`-webkit-tap-highlight-color: transparent`来去除手机上a元素等点击产生的阴影、长方形边框效果

最好给body加上,否则监听body有可能导致ios点击闪动问题

开发中遇到过一个实际问题是,监听整个body的点击事件,导致在ios上点击非可点区域时,该区域会产生灰色遮罩效果,加上`-webkit-tap-highlight-color: transparent`后解决

### 淡腾的输入法和fixed元素

有position:fixed元素时，input onFocus弹出输入法时需要将fixed改为absolute，否则会定位偏移；输入法收起后再改回fixed。             

如果策划想让input框fix在页面顶部，请拒绝这个需求吧，因为这样在ios上focus时会无法避免输入框抖动

### body上overflow:hidden失效

移动端在body上设置`overflow:hidden`无效，似乎是因为浏览器在解析`<meta name="viewport">`标签的时候会忽略html和body上的overflow属性。

可以加个wrapper将需要禁止滚动的内容**包裹**起来，在wrapper上设置`overflow:hidden`

### 伪元素 :before, :after并不是所有元素都支持!

毕竟指的是在元素中的内容前后添加，以下元素本身不能包裹内容，遇到需要添加伪元素的情况时只能在外层包括div                  
    
> IE 不支持的元素有：img，input，select，textarea      
> FireFox 不支持的元素有：input，select，textarea     
> Chrome 不支持的元素有：input[type=text]，textarea      

*chrome的input[type=checkbox]等支持伪元素简直就是个bug啊！！！*

### 在移动端retina屏幕实现真·1px边框

对于retina屏幕，dpr=2，代码中的像素数都会以*2的结果显示在屏幕上，也就是说1px会被显示成2px。
那我们暂时还不太能写成0.5px，于是要实现1px边框只能这么做：

```css          
.border-1px {
    position: relative;
}
.border-1px:after {
    border: 1px solid #c8c7cc;
    position: absolute; 
    content: ''; 
    top: 0; 
    left: 0; 
    width: 100%; 
    height: 100%; 
}

/* if border-radius is needed */
.border-1px, .border-1px:after {
    border-radius: 4px;
}

/* for dpr >= 2 利用缩放 */
@media screen and (-webkit-min-device-pixel-ratio: 2){
    .border-1px:after {
        width: 200%; 
        height: 200%;
        -webkit-transform: scale(0.5); 
        -ms-transform: scale(0.5); 
        transform: scale(0.5);
        -webkit-transform-origin: top left; 
        -ms-transform-origin: top left; 
        transform-origin:top left;
    }
    /* if border-radius is needed */
    .border-1px:after {
        border-radius: 8px;
    }
}           
```

### 微信分享事件

自定义微信分享的图片和标题等信息：

```js
var weChatShare = function() {
  document.addEventListener('WeixinJSBridgeReady', function () {
      // 发送给好友;
      WeixinJSBridge.on('menu:share:appmessage', function (argv) {
          WeixinJSBridge.invoke('sendAppMessage', {
              "appid":      '',
              "img_url":    '', // 分享图片的地址
              "img_width":  '', 
              "img_height": '',
              "link":       '', // 分享链接的地址
              "desc":       '', // 描述
              "title":      ''  // 标题 
          }, _.noop);
      });
      // 分享到朋友圈;
      WeixinJSBridge.on('menu:share:timeline', function (argv) {
          WeixinJSBridge.invoke('shareTimeline', {
              "img_url":    '', // 分享图片的地址
              "img_width":  '', 
              "img_height": '',
              "link":       '', // 分享链接的地址
              "desc":       '', // 描述
              "title":      ''  // 标题 
          }, _.noop);
      });
  }, false);
};
```
  
### textarea的placeholder不自动换行问题

textarea的placeholder文字如果过长，在安卓4.3以下(not very sure)不会自动换行，即会显示不全。
最好的解决办法是在textarea后添加span标签来显示placeholder，然后用js控制placeholder的显隐。

### IOS上滑动不顺畅

IOS上滑动不顺畅，卡，甚至导致页面无法滚动，可以在**滚动元素**上添加`-webkit-overflow-scrolling: touch` 丝般顺滑，搞定！ 

但是在滚动元素上加`-webkit-overflow-scrolling: touch`还有个bug：
当该元素中有`position: relative`元素时，可能会出现页面内容消失的情况。
解决的办法是给该元素加上`-webkit-transform: translate3d(0, 0, 0)`，强制开启硬件加速。

所以解决滑动不顺畅问题一般得在滚动元素上写两个：`-webkit-overflow-scrolling: touch; -webkit-transform: translate3d(0, 0, 0)`

### 父级元素中的transform 导致子元素position:fixed失效

要是一个fixed元素的父级元素中有使用transform,那么这个元素的fixed会失效。

这是一个浏览器bug，浏览器目前不能同时render这两种属性。只能尽量避免出现这种情况。

### scroll事件触发

scroll事件在Android中会触发多次（在滚动时），而在IOS只触发一次（最后滚动停止时）

* android: 手指接触屏幕(touchstart)->滚动(touchmove,同时scroll,频率比move高)->手指离开屏幕(touchend)->惯性滚动(scroll*N)->惯性滚动停止(scroll)

* ios: 手指接触屏幕(touchstart)->滚动(touchmove)->手指离开屏幕(touchend)->惯性滚动(真空,无法监听)->惯性滚动停止(scroll*1)


那么要如何兼容监听**滚动停止**事件：

```js
var timer = null;
$(window).addEventListener('scroll', function() {
    if(timer !== null) {
        clearTimeout(timer);        
    }
    timer = setTimeout(function() {
        // do something
    }, 150);
}, false);
```

### flexbox最小内容宽高问题

[flex的官方描述](https://www.w3.org/TR/css-flexbox/#flex-common)里有一段话：

> By default, flex items won’t shrink below their minimum content size (the length of the longest word or fixed-size element). 
To change this, set the min-width or min-height property.

[min-width文档](https://developer.mozilla.org/en-US/docs/Web/CSS/min-width)说:

> 对于flex items，min-width默认值是auto的，
`min-width:auto`对于flex items的值是min-content，对于其他元素的值是0.

但实际上，几乎所有的主流浏览器都没有遵循以上约定。在Chrome, Opera, 和Safari上，浏览器允许flex items的宽高缩小到0。
也就是说在flex shrink的时候这些items可能会被压缩，造成overlapping样式显示错误。

解决办法就是给flex items加上具体的min-width、min-height值。

### flex-wrap在ios8不被支持

不要使用flex-wrap. 在ios8(更低没测过)上,元素没有按预期地换行显示
