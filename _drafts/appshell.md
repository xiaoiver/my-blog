

# 主题：在 PWA 中使用 App Shell 模型提升性能和用户感知体验

# 简介

在构建 PWA 应用时，使用 App Shell 模型能够在视觉和首屏加载速度方面带来用户体验的提升。另外，在配合 Service Worker 离线缓存之后，用户在后续访问中将得到快速可靠的浏览体验。在实践过程中，借助流行框架与构建工具提供的众多特性，我们能够在项目中便捷地实现 App Shell 模型及其缓存方案。最后，在常见的 SPA 项目中，我们试图使用 Skeleton 方案进一步提升用户的感知体验。

# 提纲

1. App Shell 模型

在 PWA 中，使用 App Shell 模型将通用的资源与动态内容分离，可以实现对于用户的快速响应。配合 Service Worker 实现 App Shell 的预缓存之后，在弱网甚至离线环境依然能给予用户可靠的浏览体验。另外，借助流行框架与构建工具的先进特性，开发者不必从头实现 App Shell 的全部细节。

(1) 介绍 PRPL 模式和 App Shell 模型思想
(2) 介绍应用该模型后，在用户体验上带来的提升效果
    1. 用户浏览站点时，带来近似 Native App 的视觉效果
    2. 提升首屏加载速度
(3) 介绍在实际项目开发中，如何借助框架和构建工具实现该模型
(4) 介绍在不同架构（SPA、MPA、SSR）下的 Service Worker 通用预缓存方案

2. Skeleton 方案

为了进一步提升用户感知体验，在 SPA 中可以使用 Skeleton (骨架屏)减少白屏时间。
我们将介绍两种生成方案的实现思路，以及在灵活可用性、展现速度上的优化方式。

(1) 构建阶段的生成方案，包含以下两种：
    1. 额外编写组件。使用框架的 SSR (服务端渲染)功能
    2. 自动化生成。使用 headless Chrome 渲染真实页面内容，随后使用占位符进行替换
(2) 解决 SPA 中多页面差异性问题。根据不同页面路由展现不同内容
(3) 优化展现速度。异步加载样式，避免阻塞以进一步减少白屏时间

# 听众收益

1. 了解 App Shell 模型的思想，感受应用该模型后对于用户体验的提升效果
2. 了解使用已有流行框架工具实现 App Shell 的推荐方式
3. 了解不同项目架构下使用 Service Worker 缓存 App Shell 的通用方案
4. 了解 SPA 中的 Skeleton 方案，能够使用现有生成工具在项目中应用

[appshell](https://www.youtube.com/watch?v=QhUzmR8eZAo)
[appshell](https://medium.com/google-developers/instant-loading-web-apps-with-an-application-shell-architecture-7c0c2f10c73)

[Skeleton](https://medium.com/@owencm/reactive-web-design-the-secret-to-building-web-apps-that-feel-amazing-b5cbfe9b7c50)
https://developers.google.com/web/showcase/2016/iowa2016?hl=zh-cn
https://developers.google.com/web/fundamentals/architecture/app-shell?hl=zh-cn
https://developers.google.com/web/fundamentals/performance/prpl-pattern/
https://huangxuan.me/2017/07/12/upgrading-eleme-to-pwa/
https://zhuanlan.zhihu.com/p/34702561

[PWA + React](https://medium.com/@addyosmani/progressive-web-apps-with-react-js-part-i-introduction-50679aef2b12)
[PWA 指标](https://www.youtube.com/watch?v=IxXGMesq_8s)
[PWA Showcase](https://developers.google.com/web/showcase/)

[Route-based chunking](https://gist.github.com/addyosmani/44678d476b8843fd981ff8011d389724)
[Flipkart Slide](https://speakerdeck.com/abhinavrastogi/next-gen-web-scaling-progressive-web-apps)
