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

### 原始方案

AMP 最初使用了这么一种解决方法：虽然 iframe 不能滚动，但是可以把 HTML 和 BODY 作为滚动容器，让其中的内容滚动。[Online Demo](http://output.jsbin.com/deyibinehu)
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

虽然 iframe 可以滚动了，但是这种方法存在以下问题：
1. 在 AMP 中，用户定义在 body 上的部分 CSS 规则会失效，例如 `margin`
2. 由于在容器内滚动，`body.scrollTop` 会始终为 0，`body.scrollHeight` 也等于视口高度而非实际全部内容高度

第二个问题影响很大，例如要实现“回到顶部”这样的组件，就无法通过 `window.scrollTo()` 完成了。
针对这个问题，AMP 给出了这样的 HACK 方案：
* 向 `<body>` 中插入一个不可见的定位元素 `<div>`，使用绝对定位 `position:absolute;top:0;left:0;width:0;height:0;visibility:hidden;`
* 这样在滚动时，通过 `-topElement.getBoundingClientRect().top` 得到顶部的滚动距离
* 类似的，插入另一个底部定位元素，通过 `endElement.offsetTop` 获取滚动高度
* 创建一个新的顶部定位元素，在执行滚动到某个位置时，改变其 `top`，然后调用 `scrollIntoView()`

很麻烦是不是，需要创建三个定位元素。于是 AMP 很快提出了一种**改进方案**。

### 改进方案

既然直接在 body 上滚动会有损失，在 body 外再套一个 wrapper。
之所以选择 `<html>` 是为了保证 `html > body` 这样的规则能继续生效。
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

AMP 目前使用的就是这种方案，但是我在 iOS 7 的测试中，会报出错误。
原因是不支持在例如 `document` 这样的对象上调用 `defineProperty`。

在我们的开发场景下，为了兼容 iOS 7，最终采用的其实是第一种原始方案。
当然如果不考虑 iOS 7 的覆盖情况，可以大胆采用这个改进方案。

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

但是有两点需要注意：
* 最好不要监听 `touchstart` 事件，否则在一些第三方浏览器（UC）中，会出现点击输入框弹起软键盘时出现不必要的滚动。应该监听 `scroll` 事件。
* 如果监听的是 `scroll` 事件，页面在初始状态就应该触发一次，或者直接调用滚动 1px。`touchstart` 则不需要。

## UC/手百 隐藏 iframe 造成页面未响应

这个问题一般的场景不会遇到，包括 AMP，是关于隐藏掉已有的 iframe 造成的。

在 MIP 的多页面切换方案中，进行页面切换时，会把当前的 `<iframe>` 进行隐藏/展现，使用 `display: none`。
但是在 iOS 的 **UC/手百** 下，切换时会出现页面不响应，假死的现象，十分奇怪。

打开这个简单的 **_测试页面 /mip/examples/page/iframe/uc.html_** ，就可以复现，步骤如下：
1. 原始页面使用 `<iframe>` 嵌入一个测试页面 `scroll.html`
2. 点击隐藏按钮隐藏掉整个 iframe
3. 原始页面不响应，表现为无法滚动，按钮无法点击等等

测试页面 `scroll.html` 包含一个菜单，使用了 iOS 弹性滚动。另外还包含以下测试：
1. 嵌入 `m.baidu`，`AMP 页面` 都会出现这种情况。而 PC 百度，eleme H5 则不会出现。
2. 使用以下方法隐藏 `<iframe>` 同样会出问题：
  - `visibility: hidden`
  - `opacity: 0` + `height: 0` + `width: 0`

经过一番探索，发现 iOS 下 UC/手百 使用的是 UIWebView，而使用了较新的 WKWebview 的例如微信就不存在这个问题。
UIWebView 除了这个奇怪的问题，还有诸如 **scroll 事件延迟** 等其他滚动相关的问题，[详见](https://harttle.land/2018/06/23/uiwebview-bugs.html)。

总之，有问题的页面使用了弹性滚动 `-webkit-overflow-scrolling: touch;`。
而一旦不使用这个属性，或者在隐藏 iframe 的同时由被嵌入的页面去掉这个属性，就不会出现问题。
```javascript
// 需要在隐藏的同时去掉 -webkit-overflow-scrolling: touch;
iframe.contentDocument.querySelector('.menu').classList.remove('touch-scrolling')
```

最终，我们使用了这种方法覆盖掉弹性滚动特性，在页面 `<iframe>` 隐藏时插入一段固定的
`<style>* {-webkit-overflow-scrolling: auto!important;}</style>`
[ISSUE](https://github.com/mipengine/mip2/issues/19)

## 总结

iOS + iframe，简直就是无尽的麻烦。

## 参考资料

* [AMP, iOS, Scrolling and Position Fixed](https://medium.com/@dvoytenko/amp-ios-scrolling-and-position-fixed-b854a5a0d451)
* [AMP, iOS, Scrolling and Position Fixed Redo — the wrapper approach](https://hackernoon.com/amp-ios-scrolling-and-position-fixed-redo-the-wrapper-approach-8874f0ee7876)
* [The AMP Project and Igalia working together to improve WebKit and the Web Platform](http://frederic-wang.fr/amp-and-igalia-working-together-to-improve-the-web-platform.html)
* [Body scroll lock — making it work with everything](https://medium.com/jsdownunder/locking-body-scroll-for-all-devices-22def9615177)
* [Six things I learnt about iOS Safari's rubber band scrolling](http://blog.christoffer.online/2015-06-10-six-things-i-learnt-about-ios-rubberband-overflow-scrolling/)
* [MIP ISSUE - iOS 下 UC/手百 切换页面造成未响应](https://github.com/mipengine/mip2/issues/19)
