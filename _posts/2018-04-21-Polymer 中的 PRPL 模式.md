---
layout: post
title: "Polymer 中的 PRPL 模式"
cover: "/assets/img/polymer.png"
date:   2018-04-21
category: coding
tags: WebComponents Polymer
author: xiaOp
comments: true
---

Chrome 提出 PRPL 的模式在自家 Polymer 中的应用状况是什么样的呢？与其他流行框架相比有哪些区别呢？

## PRPL 模式

首先简单介绍一下 [PRPL 模式](https://developers.google.com/web/fundamentals/performance/prpl-pattern/?hl=zh-cn):
* 推送 Push - 为初始网址路由推送关键资源。
* 渲染 Render - 渲染初始路由。
* 预缓存 Preload - 预缓存剩余路由。
* 延迟加载 Lazy-load - 延迟加载并按需创建剩余路由。

结合 PWA 的 Service Worker 缓存方案以及 HTTP/2 的资源推送，能够有效减少首屏加载时间，并提升后续访问性能。

借助现有的流行前端框架和构建工具，开发者在应用中实现这一模式已经十分方便，尤其是按需加载路由这部分。

## 路由懒加载

Webpack 提供的 [Code Splitting 代码分离](https://doc.webpack-china.org/guides/code-splitting/)可以将模块分离出来，在运行时异步动态加载：
{% prism javascript linenos %}
import(
    /* webpackChunkName: "my-chunk-name" */
    /* webpackMode: "lazy" */
    'module'
);
{% endprism %}

结合流行框架配套的前端路由，我们就能实现路由的懒加载。例如在 vue-router 中，支持使用异步组件：
{% prism javascript linenos %}
const Foo = () => import('./Foo.vue')
const router = new VueRouter({
    routes: [
        { path: '/foo', component: Foo }
    ]
})
{% endprism %}

这样在路由切换时才会请求所需的路由组件文件。

## Polymer 中的做法

首先让我们看一下 Polymer SPA 的项目结构：

{% responsive_image path: assets/img/app-build-components.png alt: "Polymer 项目结构" %}

entrypoint 类似 Webpack 所需的入口文件，应该足够精简，仅包含特性检测之后引入的 polyfill，以及通过 HTML imports 引用的 App Shell。
App Shell 包含了前端路由，全局的导航 UI 等等，以及需要实现动态加载 fragment 的逻辑。
fragment 类似异步路由组件。

Polymer 并没有使用类似 Webpack 这样的构建工具，在自身的配置文件 `polymer.json` 中指明了这三部分内容：
{% prism json linenos %}
{
    "entrypoint": "index.html",
    "shell": "src/my-app.html",
    "fragments": [
        "src/my-view1.html",
        "src/my-view2.html",
        "src/my-view3.html",
        "src/my-view404.html"
    ]
}
{% endprism %}

那么 Polymer 是如何解决异步加载问题的呢？

### 异步加载路由

Polymer 提供了异步引入的 API：
{% prism javascript linenos %}
var resolvedPageUrl = this.resolveUrl('my-' + page + '.html');
this.importHref(resolvedPageUrl,
    null,
    this._importFailedCallback,
    true
);
{% endprism %}

通过这个 API，开发者能够配合路由组件，实现异步加载，并在出错时跳转到 404 页面。

### 打包策略

之前我们一直在关注 PRPL 中的 Lazy-load 部分，在 Push 也就是推送资源方面又是怎么考虑的呢？

在 Polymer 中，可以在 `polymer.json` 中配置不同的打包策略：
{% prism json linenos %}
"builds": [
    {
       "preset": "es5-bundled"
    },
    {
       "preset": "es6-bundled"
    },
    {
       "preset": "es6-unbundled"
    }
]
{% endprism %}

es5/6 很好理解，bundled/unbundled 是啥意思呢？

我们知道 HTTP/2 中，服务端在返回 HTML 的同时，可以向浏览器推送所需的静态资源，这样在浏览器解析 HTML 遇到相应的资源时，它们已经在 HTTP 缓存中了。所以针对这一特性，过去打包所有静态资源以减少网络请求数的考量就没有必要了，反而拆分成多个 bundle 更有利于不同页面间共享的缓存。

{% responsive_image path: assets/img/app-build-bundles.png alt: "Polymer 打包策略" %}

## 参考资料

* [vue-router 的路由懒加载](https://router.vuejs.org/zh-cn/advanced/lazy-loading.html)
* [Polymer 中的 PRPL 模式](https://www.polymer-project.org/2.0/toolbox/prpl)