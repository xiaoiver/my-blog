---
layout: post
title:  "为jekyll增加PWA特性"
header-img: "img/post-bg-js-module.jpg"
date:   2017-07-21 19:21:32
category: coding
tags: ruby jekyll
---

## 背景

试图为 jekyll 增加 PWA 特性支持，顺便整理了下自己之前放在 github 上的博客。

## bundler

我们都知道 ruby 中安装依赖的工具是 gem，类似 Node.js 中的 npm。但不同之处在于，gem 的依赖是全局性的，并没有项目內的`node_modules`，而且 gem 并没有类似 package.json 这样的依赖描述文件，这就导致安装特定版本号的依赖很不方便。

而 bundler 提供了缺失的依赖管理功能，正如自身的介绍“一个管理 gem 的 gem”。主要依靠两个描述文件：
* `Gemfile`类似`package.json`，包含依赖列表，版本要求等。
* 安装所有依赖后会生成`Gemfile.lock`，类似 yarn 中的`yarn.lock`，存储依赖的精确版本。

这样将这两个文件连同项目一并提交，就能保证代码共享后依赖的一致性。

首先使用 gem 安装，然后创建`Gemfile`文件，写入依赖并安装：
{% highlight bash %}
gem install bundler
bundle init
echo 'gem "rspec"' >> Gemfile
bundle install
bundle exec rspec
{% endhighlight %}

命令都和 npm 一样，不知道是否有借鉴。

## 已有的支持 PWA 模版

我找到了一个基于 yeoman 的[模版](https://github.com/nirgn975/generator-jekyll-starter-kit)，包含了对 PWA 的支持。这个模版在构建时使用 gulp 通过 sw-precache 生成`service-worker.js`。缓存`_site`文件夹内全部静态资源。同时定义了其他 task，包括使用 jekyll build 文章列表。所以使用这个模版之后，其实构建就完全使用 gulp 工作流了。

## 后续思路

所以我也打算以模版而非插件形式提供支持，参考 pjax 思路，使用 ajax 请求页面，替换内容，同时配合 sw-toolbox 缓存异步请求。


