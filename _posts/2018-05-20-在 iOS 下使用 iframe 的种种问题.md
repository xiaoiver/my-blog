---
layout: post
title: "在 iOS 下使用 iframe 的种种问题"
subtitle: "fixed 元素，宽度被撑开..."
cover: "/assets/img/iframe.jpg"
date:   2018-05-20
category: coding
tags: CSS javascript
author: xiaOp
comments: true
index: 48
---

最近在项目中使用 iframe，在 iOS Safari 浏览器的测试中遇到了不少问题。
查阅资料发现 AMP 提供了很多针对这类问题的 Hack 方法，可能是因为 AMP 需要考虑站长站点嵌入在 iframe 的场景吧。

## iframe 内滚动

我们都知道在 body 上滚动时会触发一些浏览器的默认特性，比如隐藏顶部地址栏和底部导航栏。
而换成容器内滚动会丢失这部分特性，iframe 也是一种容器，一样面临这样的问题。

那么有没有办法在 body 上滚动呢？

有一种尝试是在 iframe 内触发 `touch` 系列事件时，实时将计算的滑动距离设置到 iframe 高度上，以触发 body 的滚动。
但是实际操作时，会有明显的延迟卡顿。

也有开发者提出新增 API [root-scroller](https://github.com/bokand/root-scroller/blob/master/howto.md)，让任意滚动容器都可以成为类似 BODY 这样的根容器，以具有浏览器默认行为。

所以权衡之下，通常还是会选择 iframe 绝对定位，在容器内滚动：
{% prism html linenos %}
<html>
<head>
    <title>I’m a Web App and I show AMP documents</title>
    <style>
        iframe {
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
        }
    </style>
</head>
<body>
    <iframe width="100%" height="100%"
        scrolling="yes"
        src="https://cdn.ampproject.org/c/pub1.com/doc1"></iframe>
</body>
</html>
{% endprism %}

注意我们没有应用 CSS `overflow` 属性，而是直接使用 iframe 的 `scrolling` 属性。这个属性已经被 HTML5 规范[废弃](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)，但是由于历史原因，很多浏览器还是支持。

但是 iOS 不支持这个属性，详见 [Bug Report](https://bugs.webkit.org/show_bug.cgi?id=149264) 以及 [Online Demo](http://output.jsbin.com/dedega)。

AMP 使用了这么一种解决方法：虽然 iframe 不能滚动，但是可以把 HTML 和 BODY 作为滚动容器，让其中的内容滚动。[Online Demo](http://output.jsbin.com/deyibinehu)
{% prism html linenos %}
<html style="overflow-y: auto; -webkit-overflow-scrolling: touch;">
<head></head>
<body
    style="
        overflow-y: auto;
        -webkit-overflow-scrolling: touch;
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
    ">
</body>
</html>
{% endprism %}

这种方法存在以下问题：
1. 在 AMP 中，用户定义在 body 上的部分 CSS 规则会失效，例如 `margin`
2. 由于在容器内滚动，`body.scrollTop` 会始终为 0

于是 AMP 很快提出了一种**改进方案**。

### 改进方案

既然直接在 body 上滚动会有损失，在 body 外再套一个 wrapper。之所以选择 `<html>` 是为了保证 `html > body` 这样的规则能继续生效。
{% prism html linenos %}
<html AMP
    style="overflow-y: auto; -webkit-overflow-scrolling: touch;">
    <head></head>
    <html id="i-amp-html-wrapper"
        style="
            display: block;
            overflow-y: auto;
            -webkit-overflow-scrolling: touch;
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
        ">
        <body style="position: relative;">
        </body>
    </html>
</html>
{% endprism %}

为了生成上述结构，AMP 中使用 JS 对于原站点 HTML 进行改造。
值得注意的是，由于 body 不再是根元素的子节点，需要通过 `defineProperty` 保证 `document.body` 依然可访问。
{% prism javascript linenos %}
// Create wrapper.
const wrapper = document.createElement('html');
wrapper.id = 'i-amp-html-wrapper';

// Setup classes and styles.
wrapper.className = document.documentElement.className;
document.documentElement.className = '';
document.documentElement.style = '...';
wrapper.style = '...';

// Attach wrapper straight inside the document root.
document.documentElement.appendChild(wrapper);

// Reparent the body.
const body = document.body;
wrapper.appendChild(body);

Object.defineProperty(document, 'body', {
    get: () => body,
});
{% endprism %}

如果说不支持非标准的 `scrolling` 属性还可以理解，下面这个问题就很莫名了。

## fixed 元素滚动问题

iframe 中的 fixed 元素在 iframe 滚动时会上下跳动，似乎一直无法确定自己的位置。[Video](https://drive.google.com/file/d/0B_v8thsbiGyDMXZMZkRFZGFRbjA/view) [Online Demo](http://output.jsbin.com/fusijum/quiet)

这也是一个 iOS 的 Bug，可以看到 2016 年就已经存在 [Bug Report](https://bugs.webkit.org/show_bug.cgi?id=154399)了。

AMP 使用了这样一个 Hack 方法：创建一个 `body` 的兄弟节点，将原有页面中所有的 fixed 元素移动到这个节点中，要注意设置正确的 `z-index`。
{% prism html linenos %}
<html>
<body>...</body>
<div id="fixed-layer"
    style="
      position: absolute;
      top: 0;
      left: 0;
      width: 0;
      height: 0;
      pointer-events: none;
    ">
    <div id="fixed-element"
        style="pointer-events: initial; z-index: 11;">
    </div>
</div>
</html>
{% endprism %}

这种方法的缺点很明显，改变了原有页面结构，会导致部分 CSS 规则匹配不上。不过对于 AMP 这样可以限制开发者使用组件的场景，倒是可以接受。

## 宽度被撑开

当我们给 iframe 设置宽度 `100%` 时，例如 `<iframe width="100%"></iframe>` 或者通过 CSS，我们希望其中的内容是响应式的，不应该出现滚动条。

但是 iOS 下存在[问题](https://stackoverflow.com/questions/23083462/how-to-get-an-iframe-to-be-responsive-in-ios-safari)，`width: 100%` 似乎被浏览器的默认设置覆盖了，无法得到应用。

有一种 HACK 方式如下，在 [AMP ISSUE](https://github.com/ampproject/amphtml/issues/11133) 中也采用了类似思路，使用 `min-width` 覆盖掉 iOS Safari 对于 `width` 的默认设置：
{% prism css linenos %}
iframe {
    width: 1px;
    min-width: 100%;
}
{% endprism %}

## 滚动穿透

这个问题倒不是 iframe 独有，而是 iOS 下普遍存在的问题。

在常见的对话框浮层场景下，在 `fixed` 定位的浮层上滚动时，很容易滚动到下层的 `body` 上，造成穿透现象。

![](/assets/img/1_default_bottom_overscroll.gif)

首先想到的方案是在下层容器上禁用滚动：
{% prism css linenos %}
html, body {
    overflow: hidden;
}
{% endprism %}

但存在两个问题：
1. 在安卓上可行，iOS 下无效
2. 关闭浮层时需要恢复 `body` 上的滚动距离，因为禁用滚动的瞬间会丢失当前的滚动距离

为了解决这两个问题，开发者总结出了以下方案。

### body-scroll-lock

![](/assets/img/body-scroll-lock.png)

大致思路是安卓上沿用 `overflow: hidden` 方案。iOS 上监听 `touch` 系列事件，在滚动到顶部和底部时禁用掉浏览器默认行为：
{% prism javascript linenos %}
const clientY = event.targetTouches[0].clientY - initialClientY;
// 滚动到顶
if (targetElement && targetElement.scrollTop === 0 && clientY > 0) {
    return preventDefault(event);
}
// 滚动到底
if (isTargetElementTotallyScrolled(targetElement) && clientY < 0) {
    return preventDefault(event);
}
{% endprism %}

虽然不会存在滚动穿透问题了，但是这个方案依然存在两个问题：
1. 安卓上恢复滚动距离的问题依然存在
2. iOS 上由于禁用掉了浏览器默认行为，弹性滚动也不存在了

那么有没有完美的解决方案呢？

### 1px 滚动

通过观察我们发现，只有滚动超过顶部或者底部触发 iOS 的默认行为才会带来问题，那么如果我们能在滚动到边缘时保留 1px 的距离，这个问题也就不存在了。参考[这个 gist](https://github.com/luster-io/prevent-overscroll/blob/master/index.js)的做法：
{% prism javascript linenos %}
el.addEventListener('touchstart', function() {
    let top = el.scrollTop;
    let totalScroll = el.scrollHeight;
    let currentScroll = top + el.offsetHeight;

    if (top === 0) {
        el.scrollTop = 1;
    } else if (currentScroll === totalScroll) {
        el.scrollTop = top - 1;
    }
});
{% endprism %}

其实 Google AMP 也是这么做的，在 iOS 上打开 AMP 页面，滚动到顶触发 iOS 的弹性滚动，仔细观察页面会有一个轻微的不易察觉的滚动。

## 参考资料

* [AMP, iOS, Scrolling and Position Fixed](https://medium.com/@dvoytenko/amp-ios-scrolling-and-position-fixed-b854a5a0d451)
* [AMP, iOS, Scrolling and Position Fixed Redo — the wrapper approach](https://hackernoon.com/amp-ios-scrolling-and-position-fixed-redo-the-wrapper-approach-8874f0ee7876)
* [The AMP Project and Igalia working together to improve WebKit and the Web Platform](http://frederic-wang.fr/amp-and-igalia-working-together-to-improve-the-web-platform.html)
* [Body scroll lock — making it work with everything](https://medium.com/jsdownunder/locking-body-scroll-for-all-devices-22def9615177)
* [Six things I learnt about iOS Safari's rubber band scrolling](http://blog.christoffer.online/2015-06-10-six-things-i-learnt-about-ios-rubberband-overflow-scrolling/)
