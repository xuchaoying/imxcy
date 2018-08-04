---
uuid: 1508df0a-4282-4e0d-8ab9-454093952f9c
title: WebP接入方案探究
date: 2017-02-05 15:32:25
categories:
 - frontend
thumbnail: /explore-webp/caniuse-webp.png

---

WebP(读作weppy)是谷歌提出的一种新的web图片格式.它能很出色地完成对图片的有损或者无损压缩,且支持图片透明度(alpha通道).

根据[官方文档](https://developers.google.com/speed/webp/#top_of_page),在同等[SSIM](https://en.wikipedia.org/wiki/Structural_similarity)指标下,WebP无损图片大小较PNG图片缩小了26%,
WebP有损图片较JPEG图片缩小了25-34%.

目前谷歌官方提供了[预编译工具](https://developers.google.com/speed/webp/docs/precompiled)将图片转换成WebP格式.

## WebP原理

### 有损图片

对于有损图片, WebP使用了VP8视频编解码器中的压缩单个帧的方法,即预测编码(predictive coding)技术来压缩图片.

首先将图片分割成不同的宏块(MacroBlocking),对于宏块中的每个4*4大小的block,预测编码使用以下几种帧内预测的方法进行色值预测,得到最接近原块的预测块:

- H_PRED(horizontal prediction): 使用block左边的一列L来填充block中的每一列

- V_PRED(vertical prediction): 使用block上边的一行A来填充block中的每一行

- DC_PRED(DC prediction): 使用L和A中所有像素的平均值作为唯一的值填充block

- TM_PRED(TrueMotion prediction): 使用渐进的方式,记录上面一行的渐进差,以同样的差值,以L为基准拓展每一行

得到最佳的预测块后,导出过滤结果(剩余误差),然后对其执行DCT过滤,使得变换后的数据低频部分分布在数据块的左上方,高频部分在右下方.

进一步进行量化处理,这使得数据高频部分的频率系数被大大衰减甚至清零,低频部分则被很好地保留(人眼对低频更敏感),从而使数据得到有效压缩.

转成量化矩阵后重新排序，然后送到一个静态压缩器里进行压缩(WebP使用的是[算术压缩器](https://www.youtube.com/watch?v=FdMoL3PzmSA&index=7&list=PLOU2XLYxmsIJGErt5rrCqaSGTMyyqNt2H)).

从结果来看WebP很像JPEG,但prediction coding是WebP之所以强于JPEG的重要原因,量化也起到了一些优化作用,算数压缩器的压缩率也较JPEG的哈夫曼压缩器高了5%-10%.


### 无损图片

无损图片的压缩使用了多种不同的图片转换技术,包括预测像素空间变换,色彩空间转换,使用调色板,将多像素打包到一个像素中,以及alpha值替换等.转换系数以及转换后的图片数据则采用了熵编码(entropy coding),它使用了改进版LZ77-Huffman编码,通过对距离值的2D编码来紧凑稀疏值.

## 浏览器支持

目前google chrome浏览器,Opera浏览器等都内置了对WebP的支持,根据[caniuse](http://caniuse.com/#search=webp)的数据:

![WebP浏览器支持度](caniuse-webp.png)

目前Safari和Firefox也对WebP进行了实验性支持.也就是说,在项目中使用WebP图片,大约72%的用户可以直接体验到.

## WebP接入技术

### server-side内容转换

如果浏览器支持WebP,它会在请求头Accept中带上image/webp,来告诉服务端它可以接收WebP格式图片.

所以可以由服务端直接转换图片并返回.

### PageSpeed自动转换模块

Google开发的[PageSpeed模块](https://modpagespeed.com/doc/)可以自动将图像转换成WebP格式或者是浏览器所支持的其它格式。

PageSpeed的设置很简单,比如在nginx中:

首先在http模块开启pagespeed属性:

```nginx
pagespeed on;
pagespeed FileCachePath "/var/cache/ngx_pagespeed/";
```
然后在主机配置添加如下一行代码，就能启用这个特性:

```nginx
pagespeed EnableFilters convert_png_to_jpeg,convert_jpeg_to_webp;
```

结果对比:
```html
<!--原始代码-->
<img src="./574ceeb8N73b24dc2.jpg" />

<!--在支持WebP的浏览器中,它会被转换成:-->
<img src="x574ceeb8N73b24dc2.jpg.pagespeed.ic.YcCPjxQL4t.webp"/>

<!--在不支持WebP的浏览器中,它会被转换成:-->
<img src="x574ceeb8N73b24dc2.jpg.pagespeed.ic.3TXX_PUg99.jpg"/>
```

### HTML5 picture标签

picture标签允许列出多个图片源,并按照顺序加载第一张它可以展示的图片.

所以可以把WebP url写在原格式url的前面,做好WebP加载失败的fallback.

### 在JS中

在浏览器中,可以使用javascript检测浏览器是否支持WebP.如果浏览器支持WebP则展示WebP格式图片,否则展示正常格式图片.

谷歌官方的判断方法,加载一张图片看能否加载成功,需要异步回调:
```js
// check_webp_feature:
// 'feature' can be one of 'lossy', 'lossless', 'alpha' or 'animation'.
// 'callback(feature, result)' will be passed back the detection result (in an asynchronous way!)
function check_webp_feature(feature, callback) {
    var kTestImages = {
        lossy: "UklGRiIAAABXRUJQVlA4IBYAAAAwAQCdASoBAAEADsD+JaQAA3AAAAAA",
        lossless: "UklGRhoAAABXRUJQVlA4TA0AAAAvAAAAEAcQERGIiP4HAA==",
        alpha: "UklGRkoAAABXRUJQVlA4WAoAAAAQAAAAAAAAAAAAQUxQSAwAAAARBxAR/Q9ERP8DAABWUDggGAAAABQBAJ0BKgEAAQAAAP4AAA3AAP7mtQAAAA==",
        animation: "UklGRlIAAABXRUJQVlA4WAoAAAASAAAAAAAAAAAAQU5JTQYAAAD/////AABBTk1GJgAAAAAAAAAAAAAAAAAAAGQAAABWUDhMDQAAAC8AAAAQBxAREYiI/gcA"
    };
    var img = new Image();
    img.onload = function () {
        var result = (img.width > 0) && (img.height > 0);
        callback(feature, result);
    };
    img.onerror = function () {
        callback(feature, false);
    };
    img.src = "data:image/webp;base64," + kTestImages[feature];
}
```

在stackoverflow上看到的[另一种方法](http://stackoverflow.com/a/27232658),通过canvas.toDataUrl()方法来代替图片加载,避免了异步回调:
```js
function canUseWebP() {
    var elem = document.createElement('canvas');

    if (!!(elem.getContext && elem.getContext('2d'))) {
        // was able or not to get WebP representation
        return elem.toDataURL('image/webp').indexOf('data:image/webp') == 0;
    }
    else {
        // very old browser like IE 8, canvas not supported
        return false;
    }
}
```

## 参考
[WebP谷歌官方文档](https://developers.google.com/speed/webp/)

[探究WebP一些事儿](https://aotu.io/notes/2016/06/23/explore-something-of-webp/)

[WebP原理和Android支持现状介绍](https://dev.qq.com/topic/582939577ef9c5b708556b0d)




