---
layout: post
title:  "在Github Pages中使用第三方插件"
subtitle: "仅对本地构建结果进行版本控制"
cover: "/assets/img/post-bg-js-module.jpg"
date:   2017-07-22
category: coding
tags: ruby jekyll github-pages
author: xiaOp
index: 8
---

[Github Pages](https://pages.github.com/)能够帮助我们托管 jekyll 项目，我们只需要将代码提交到指定分支(master/gh-pages)，github 将自动完成构建。

jekyll 项目不可避免的会使用各种各样的插件，而 Github Pages 目前仅支持到 jekyll 3.4.5 版本，新版本在`_config.yml`中对插件的使用方法发生了变更，使用`plugins`代替旧版本的属性`gems`。所以为了兼容 Github Pages，本地新版本的警告提示信息只能选择忽略了。

和在本地构建不同的一点是，github 构建时会加上`--save`参数，只运行[信任的插件](https://pages.github.com/versions/)。这就导致如果我们的博客使用了不在白名单中的插件，github 构建时就会提示警告（在项目标签页 Setting -> GitHub Pages 中），构建失败也就看不到最新的博客内容了。

Github Pages 给出一种建议是，让用户在本地构建站点，然后仅将构建结果纳入版本控制，提交到指定分支。避免 github 构建，只需要在根目录下放一个空文件`.nojekyll`。

至于本地构建，熟悉 Node.js 的开发者完全可以采用 gulp/grunt 工作流，调用`jekyll build`构建站点，顺便还能做一些静态资源压缩的工作。我在网上找到一种使用 rakefile 的[简单方法](https://www.sitepoint.com/jekyll-plugins-github/)，决定一试。

## 使用 Rakefile 构建

首先我们需要拷贝原有 repo 内容到一个新文件夹中，以后这个新文件夹将成为我们的工作文件夹。由于以后我们不会再在原有 repo 中进行写作，所以此时可以在 repo 中执行`git rm -r ./`清空所有版本控制。

在新文件夹中增加 Rakefile，进行如下操作：
1. 使用`jekyll build`构建站点
2. 清空真正的 repo 内容，拷贝`_site`文件夹内容到里面。
{% prism ruby linenos %}
GH_PAGES_DIR = "xiaoiver.github.io"

desc "Build Jekyll site and copy files"
task :build do
  system "jekyll build"
  system "rm -r ../#{GH_PAGES_DIR}/*" unless Dir['../#{GH_PAGES_DIR}/*'].empty?
  system "cp -r _site/* ../#{GH_PAGES_DIR}/"
end
{% endprism %}

Rakefile 只在构建时使用，所以需要在`_config.yml`中过滤掉，避免拷贝到结果文件夹中：
{% prism yml linenos %}
exclude: [Rakefile]
{% endprism %}

然后回到 repo 中，提交修改即可。

## 使用 gulp 工作流构建

例如这个基于 yeoman 的[模版项目](https://github.com/nirgn975/generator-jekyll-starter-kit)，包含了对 PWA 的支持。

这个模版在构建时使用 gulp 通过 sw-precache 生成`service-worker.js`，缓存`_site`文件夹内全部静态资源。同时也定义了其他 task 用于压缩静态文件。

我在博客中也使用了这种方式。
