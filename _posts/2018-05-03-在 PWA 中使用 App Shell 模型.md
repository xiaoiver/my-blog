---
layout: post
title: "在 PWA 中使用 App Shell 模型"
subtitle: "提升性能和用户感知体验"
cover: "/assets/img/pwa-appshell.png"
date:   2018-05-03
category: coding
tags: PWA
author: xiaOp
comments: true
index: 43
---

在构建 PWA 应用时，使用 App Shell 模型能够在视觉和首屏加载速度方面带来用户体验的提升。另外，在配合 Service Worker 离线缓存之后，用户在后续访问中将得到快速可靠的浏览体验。
在实践过程中，借助流行框架与构建工具提供的众多特性，我们能够在项目中便捷地实现 App Shell 模型及其缓存方案。最后，在常见的 SPA 项目中，我们试图使用 Skeleton 方案进一步提升用户的感知体验。

## App Shell 模型

相比 Native App，PWA 有以下优势：
* Linkable 毕竟是 Web 站点，通过链接跳转，也便于分享以及索引。
* Progressive 渐进式提升站点体验。即使不支持 Service Worker 仍能运行。添加到主屏，消息推送等特性也是如此。
* Responsive 同样针对手持设备，开发者熟悉的响应式设计仍然能很好的应用。

我们都很熟悉 Native App 中常见的 Shell 展示效果，通常快速加载应用的简单 UI （顶部导航条，侧边栏，Loading 动画等）并缓存，后续访问甚至是离线状态仍能立即展示，而页面实际内容动态加载。PWA 在保持以上优势的基础上，也可以借鉴这一方案以提升性能和用户感知体验，这就是 App Shell 模型。

{% responsive_image path: assets/img/appshell.jpeg alt: "App Shell 模型" %}

我们对于 PWA 中的 App Shell 模型的大致总结：
* 内容上是由 HTML CSS 和 JS 组成的资源集合
* 它应该是快速加载运行展示并可以被缓存的
* 还需要负责后续动态加载页面实际内容
* 与 iOS/Android App 相比，体积小得多

那么在具体项目中应该如何应用这一模型呢？或者说，对于已有项目的改造成本有多大呢？

我们熟悉的 Web 项目的架构大致如下：
* Server-side Rendering 首屏加载速度快，但是后续每次页面间跳转都需要重新下载全部资源。
* Client-side Rendering 首屏加载速度慢，后续页面跳转迅速。

{% responsive_image path: assets/img/cs&ss.png alt: "客户端&服务端渲染" %}

所以两者结合可以得到最好的效果，首屏由 SSR 渲染，后续由 CSR 动态渲染页面中部分内容，类似 SPA 的效果。
借助构建工具例如 Webpack 和前端框架（React Vue）提供的服务端渲染特性，同一套代码在编译后可以同时运行在双端，这就是 Universal/Isomorphic 同构应用的思想。

在上述架构下都可以应用 App Shell 模型。首先我们来看在 SPA 中的应用。

## SPA 中的应用

SPA 中的内容全部由 JS 在前端渲染。为了实现 App Shell 的特性，在具体实现或者对于已有项目的改造时，我们可以应用 PRPL 模式。

### PRPL 模式

PRPL 模式是 Google 提出的，包含以下特性：
* Push 推送 - 为初始网址路由推送关键资源。
* Render 渲染 - 渲染初始路由。
* Precache 预缓存 - 预缓存剩余路由。
* Lazyload 延迟加载 - 延迟加载并按需创建剩余路由。

简单用一张图表示整个过程：
{% responsive_image path: assets/img/prpl.png alt: "PRPL 模式演示" %}

前面说过，App Shell 在内容上是由 HTML CSS 和 JS 组成的资源集合。为了保证这些资源的加载速度，必须精简。
在这一思路下，它将包含：
* SPA 唯一的一个 HTML。
* JS 包括：渲染 UI 代码，前端路由器，渲染初始路由内容代码。
* 关键路径样式，其他静态资源。

为了实现全部或者部分特性，我们需要依赖以下技术：
* HTTP/2 服务。尽早帮助浏览器发现静态资源并加载。
* 前端路由。能够渲染初始路由，并且能支持后续动态加载并添加剩余路由。
* Service Worker 预缓存后续所需路由文件及静态资源。
* 构建工具的支持。包括对于 HTTP/2 的 unbundle 支持，对于代码分割的支持等等。

所幸现有的很多优秀工具和框架已经能帮助解决大部分问题，下面我们来看具体实现。

### 代码分割

为了保证 App Shell 包含资源的精简，需要将初始路由内容与后续路由内容分开。
在编译时需要构建工具进行分割打包操作。在编写代码时，有两种做法：
* 代码在编写时就是物理分割好的
* 代码在编写时不分割，使用特殊的语法指示构建工具在编译时进行分割处理

对于第一种做法，我们以 Polymer 为例。由于使用了 HTML imports，需要分割的代码天然就是物理分割，包含在各自 HTML 中的。
在构建时，配套的构建工具会读取自身的配置文件 `polymer.json`，其中显式指明了这三部分内容：
* entrypoint 即项目的入口文件，应该足够精简，仅包含特性检测之后引入的 polyfill
* App Shell。App Shell 包含了前端路由，全局的导航 UI 等等，以及需要实现动态加载 fragment 的逻辑。
* fragment 类似异步路由组件。

{% responsive_image path: assets/img/polymer-code-splitting.png alt: "Polymer 中的代码分割" %}

而对于第二种做法，我们开发者最熟悉的例子就是 Webpack 了。
引入 `babel-plugin-syntax-dynamic-import` 插件，开发者就可以使用 dynamic-import 语法：
{% prism javascript linenos %}
import(/* webpackChunkName: "my-view1" */ './my-view1')
    .then((myView1) => {
        //...
    });
{% endprism %}

现在我们已经将初始路由内容与后续路由内容分开了，渲染内容将由路由负责。

### 路由支持

对于 PRPL 模式中的路由来说，除了负责初始路由的渲染，还需要支持后续动态加载并添加剩余路由。

Polymer 提供了异步引入的 API，供配套的路由使用。
这样就能实现异步加载，并在出错时跳转到 404 页面：
{% prism javascript linenos %}
var resolvedPageUrl = this.resolveUrl('my-view1.html');
this.importHref(resolvedPageUrl,
    null,
    this._importFailedCallback,
    true
);
{% endprism %}

而在 Vue 中，由于框架本身就支持异步组件，在 vue-router 中很容易实现路由的懒加载：
{% prism javascript linenos %}
import Index from './Index.vue';
const MyView1 = () => import('./MyView1.vue');
const router = new VueRouter({
    routes: [
        { path: '/', component: Index }
        { path: '/my-view1', component: MyView1 }
    ]
});
{% endprism %}

React 也是一样：
{% prism javascript linenos %}
import Loadable from 'react-loadable';
import Loading from './Loading';

const LoadableComponent = Loadable({
    loader: () => import('./MyView1.jsx'),
    loading: Loading,
})
{% endprism %}

这样结合之前的代码分割，我们就完成了初始路由的渲染，以及后续剩余路由的按需加载。

### Service Worker 预缓存

虽然说实现了路由内容的按需加载，但毕竟要等到实际路由切换时才会请求相应代码并执行。
如果能提前告知浏览器预取这部分资源，就可以提前完成掉网络开销。

首先能想到的一个方案是 `<prefetch>`，浏览器在空闲时会去请求这些资源放入 HTTP 缓存：
{% prism html linenos %}
<link rel="prefetch" href="image.png">
{% endprism %}
但是对于开发者而言，需要更精确地控制缓存，因此还是得使用 Service Worker。

在项目构建阶段，将静态资源列表（数组形式）及本次构建版本号注入 Service Worker 代码中。
在 SW 运行时（Install 阶段）依次发送请求获取静态资源列表中的资源（JS,CSS,HTML,IMG,FONT…），成功后放入缓存并进入下一阶段（Activated）。这个在实际请求之前进行缓存的过程就是预缓存。

预缓存 App Shell 包含的 HTML JS 和 CSS，以及懒加载需要的路由 JS。
{% prism javascript linenos %}
var cacheName = 'app-shell';
var filesToCache = [
    '/index.html’,
    '/js/main.js',
    '/js/my-view1.js',
    '/js/my-view2.js',
    '/css/main.css'
];

self.addEventListener('install', function(e) {
    e.waitUntil(
        caches.open(cacheName).then(function(cache) {
            return cache.addAll(filesToCache);
        })
    );
});
{% endprism %}

借助 Workbox 提供的命令行工具以及构建工具配套的插件，开发者能轻松地通过配置生成预缓存列表甚至是整个 Service Worker 文件，缓存的更新交给 Workbox 完成。除了预缓存，Workbox 还提供了一系列 API 帮助开发者管理动态缓存，使用默认离线页面等等。
{% prism javascript linenos %}
importScripts('./workbox-sw.prod.js');
importScripts('./precache-manifest.js');

workbox.skipWaiting();
workbox.clientsClaim();

workbox.precaching.precacheAndRoute(self.__precacheManifest);
{% endprism %}

### 推送关键资源

我们知道 HTTP/2 中，服务端在返回 HTML 的同时，可以向浏览器推送所需的静态资源，这样在浏览器解析 HTML 遇到相应的资源时，它们已经在 HTTP 缓存中了。所以针对这一特性，过去打包所有静态资源以减少网络请求数的考量就没有必要了，反而拆分成多个 bundle 更有利于不同页面间共享的缓存。

例如 twitter 的 mobile 站点，注意下载 HTML 和首屏需要的 JS 几乎是同时进行的：
{% responsive_image path: assets/img/twitter-http2.png alt: "twitter HTTP/2" %}

但是对于不支持 HTTP/2 的浏览器，还有 `<preload>` 这种方式，考虑兼容性两者可以同时使用。
{% responsive_image path: assets/img/twitter-preload.png alt: "twitter preload" %}

## SSR 中的应用

在 SPA 架构的应用中，App Shell 通常包含在 HTML 页面中，连同页面一并被预缓存，保证了离线可访问。但是在 SSR 架构场景下，情况就不一样了。所有页面首屏均是服务端渲染，预缓存的页面不再是有限且固定的。如果预缓存全部页面，SW 需要发送大量请求不说，每个页面都包含的 App Shell 部分都被重复缓存，也造成了缓存空间的浪费。

既然针对全部页面的预缓存行不通，我们能不能将 App Shell 剥离出来，单独缓存仅包含这个空壳的页面呢？要实现这一点，就需要对后端模板进行修改，通过传入参数控制返回包含 App Shell 的完整页面 OR 代码片段。这样首屏使用完整页面，而后续页面切换交给前端路由完成，请求代码片段进行填充。

### 通用思路

1. 改造后端模板以支持返回完整页面和内容片段( contentOnly )
2. 服务端增加一条针对 App Shell 的路由规则，返回仅包含 App Shell 的 HTML 页面( shell.html )
3. 预缓存 App Shell 页面
4. Service Worker 拦截所有 HTML 请求，统一返回缓存的 App Shell 页面。同时向服务端请求当前页面需要的内容片段并写入缓存
5. 前端路由( app.js )向服务端请求内容片段，发现缓存中已存在，将其填充进 App Shell 中，完成前端渲染

### 传统后端模版项目

以传统的后端模版项目为例，对于用户的请求，根据 URL 使用默认 Layout + 对应视图模版进行响应。
{% responsive_image path: assets/img/ssr-template-user.png alt: "用户访问服务器" %}

而 Service Worker 安装时，也会向服务器发送请求。对于服务器而言，新增了一种访问角色，与之对应的，需要增加一系列针对 Service Worker 的路由规则，将单独的视图模版和默认 Layout 返回给 Service Worker。
{% responsive_image path: assets/img/ssr-template-sw.png alt: "Service Worker 访问服务器" %}

对于用户而言，在 Service Worker 安装成功之后，对于 HTML 的请求都会被拦截，渲染模板的工作全部由 Service Worker 完成。
{% responsive_image path: assets/img/ssr-template.png alt: "Service Worker 渲染模板" %}

下面我们来看具体的示例代码，如果使用类似 express 这样的服务器：
{% responsive_image path: assets/img/ssr-server.png alt: "服务器渲染示例" %}

而在这样的同构思路下，如果服务端代码也是使用 Node.js 编写，理想情况下 Service Worker 就能复用其中的模板渲染和路由逻辑。
{% responsive_image path: assets/img/ssr-sw.png alt: "Service Worker 渲染示例" %}

## App Shell 性能

另外值得一提的是，除了后续路由，其他不需要出现在初始 bundle 中的模块（例如消息通知，一些 SDK 代码等等）也可以进行懒加载，这样可以大幅减少初始路由内容的大小。

我们以 Vue hackernews 2.0 这个同构项目为例，在没有使用代码分割的情况下，所有的业务逻辑全在 app.js 中。

在 3G 环境下，首屏加载时间约为 2.9s
{% responsive_image path: assets/img/vue-hackernews-raw.png alt: "原始状态" %}

使用代码分割后，首屏不需要的业务逻辑从 app.js 中移动到了异步加载文件中。首屏加载时间约为 1.2s
{% responsive_image path: assets/img/vue-hackernews-code-splitting.png alt: "路由级别的 Code Splitting" %}

使用 Service Worker 预缓存之后，再次访问速度极快，仅 0.2s
{% responsive_image path: assets/img/vue-hackernews-sw.png alt: "使用 Service Worker 后，再次访问" %}

首屏性能提升是很明显的，但是还有优化空间吗？

## Skeleton 方案

在 SPA 中，在实际内容由 JS 渲染完成之前，会存在一段白屏时间。参考 Native App 中的通常做法，可以展示 Skeleton 骨架屏，相比一个简单的 Loading 动画，更能让用户感觉内容就快加载出来了。但需要注意在本质上，这和 Loading 是没有区别的，也并不能减少白屏时间，仅仅是提高了一些用户的感知体验。

![](/assets/img/flipkart-skeleton.gif)

下面我们将从生成方式，不同路由间的差异性问题以及优化展现速度这三方面展开。

### 生成方式

从骨架屏包含的内容上看，与 Loading 一样，都是由内联在 HTML 中的样式和 DOM 结构片段组成。
我们希望在构建阶段自动将这些内容注入 HTML 中，在生成方式上有两种：
* 编写额外组件
* 自动根据页面内容生成

首先来看第一种，Skeleton 也可以视为一种组件，在编写时与其他组件开发体验一致。但不同于其他组件在运行时前端渲染，Skeleton 组件需要在构建时，也就是 Node.js 环境中渲染。借助框架的 SSR 方案，我们很容易配合构建工具实现。

插件大致实现如下：
1. 在 Webpack 当前编译环境中创建一个 childCompiler，继承编译上下文。这样可以保证 Skeleton 组件和项目其他组件使用同样的配置编译，例如 loaders。
2. 使用框架提供的 SSR 方案渲染 Skeleton 组件，得到对应的 HTML 片段
3. 使用插件分离样式，得到 CSS
4. 注入 HTML 中

这种方案存在两个问题：
1. 由于依赖框架的 SSR 方案，针对不同的框架需要开发不同的插件。目前我开发了 `vue-skeleton-webpack-plugin` 和 `react-skeleton-webpack-plugin`。
2. 需要手动编写 Skeleton 组件。

而在第二种方案中，不需要开发者编写额外的 Skeleton 组件，既然骨架屏是要反映页面内容的大致框架，完全可以在真实页面基础上，将内容替换成占位元素得到最终效果。Eleme 团队的 `page-skeleton-webpack-plugin` 就是这样一款优秀的插件。

插件大致实现如下：
1. 使用 puppeteer 提供的 API 在 Node.js 环境中运行 headless Chrome
2. 打开需要生成 Skeleton 的页面
3. 注入样式，将不同的元素替换成占位符
4. 获取页面样式和 HTML 片段
5. 注入 HTML 中

### 根据路由展示

以上两种生成方式都会面临同样的一个问题，那就是 SPA 中如果只生成一份 Skeleton，如何能保证匹配不同的路由页面呢？
在试图用一个 Skeleton 匹配多个差别极大的路由页面时，往往就退化成了 Loading 方案。

所以我们可以在构建时，为几个重要的路由页面生成各自的骨架屏，在 HTML 中注入一小段 JS，根据当前路由路径控制展示某一个。
大致思路如下：
{% prism html linenos %}
<div id="skeleton1" style="display:none">...</div>
<div id="skeleton2" style="display:none">...</div>
<script>
    // 根据路由展示对应 skeleton
</script>
{% endprism %}

## 总结

无论是 SPA 下的 PRPL 模式，还是 SSR 下的同构思路，灵活运用其中的技术思路，借助 App Shell 模型，成熟的框架以及构建工具，相信一定能开发出更多高质量的 PWA 应用。

## 参考资料

* [MDN App Shell](https://developer.mozilla.org/en-US/Apps/Progressive/App_structure)
* [Google App Shell](https://developers.google.com/web/fundamentals/architecture/app-shell?hl=zh-cn)
* [App Shell Video](https://www.youtube.com/watch?v=QhUzmR8eZAo)
* [App Shell 性能](https://medium.com/google-developers/instant-loading-web-apps-with-an-application-shell-architecture-7c0c2f10c73)
* [PWA 性能分析](https://medium.com/@addyosmani/a-tinder-progressive-web-app-performance-case-study-78919d98ece0)
* [Reactive Web Design: The secret to building web apps that feel amazing](https://medium.com/@owencm/reactive-web-design-the-secret-to-building-web-apps-that-feel-amazing-b5cbfe9b7c50)
* [Building the Google I/O 2016 Progressive Web App](https://developers.google.com/web/showcase/2016/iowa2016?hl=zh-cn)
* [PRPL 模式](https://developers.google.com/web/fundamentals/performance/prpl-pattern/)
* [Eleme PWA](https://huangxuan.me/2017/07/12/upgrading-eleme-to-pwa/)
* [自动化生成 H5 骨架页面](https://zhuanlan.zhihu.com/p/34702561)
* [让骨架屏渲染更快](https://zhuanlan.zhihu.com/p/34550387)
* [Link Prefetching FAQ](https://developer.mozilla.org/en-US/docs/Web/HTTP/Link_prefetching_FAQ)