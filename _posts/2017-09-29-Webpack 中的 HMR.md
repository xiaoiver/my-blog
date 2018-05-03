---
layout: post
title:  "Webpack 中的 HMR"
subtitle: "webpack-hot-middleware 中间件实现原理"
cover: "/assets/img/vue-webpack.png"
date:   2017-09-29
category: coding
tags: javascript webpack eventstream
comments: false
author: xiaOp
index: 16
---

在使用 Webpack 构建项目时，开发模式下代码的 hotreload 特性帮助开发者节省了很多手动刷新浏览器的时间。最近在项目中遇到了这方面的问题，决定深入研究一下背后的实现原理。

## 准备工作

首先需要引入 `webpack-hot-middleware` 中间件和 `HotModuleReplacementPlugin` 插件。当然开发环境下 `webpack-dev-middleware` 肯定也是必不可少的。

启动项目后打开 Chrome Devtools，这个`/__webpack_hmr`请求很明显是实现 hotreload 的关键。可以看出请求一直保持连接，并不断接收到“心跳”事件。
![](/assets/img/hmr.png)

## 客户端发起 socket 连接

首先观察一下请求头`text/event-stream`，这个请求是由客户端代码通过[EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)创建的：
{% prism javascript linenos %}
// webpack-hot-middleware/client.js

function init() {
    source = new window.EventSource(options.path);
    source.onopen = handleOnline;
    source.onerror = handleDisconnect;
    source.onmessage = handleMessage;
}
{% endprism %}

这里提到的“客户端代码”是指使用 webpack-hot-middleware 时，除了使用中间件之外，还需要把[client.js](https://github.com/glenjamin/webpack-hot-middleware/blob/master/client.js#L74-L79)加入入口依赖，因此这段代码会被打包到客户端代码中。创建完 EventSource 之后，绑定了若干事件处理函数，我们后续会查看这些细节。

## 服务端中间件响应

在收到客户端请求后，中间件需要进行响应，同时需要管理每一个客户端连接，这里将响应对象通过自增的 ID 保存在`clients`中，当连接断开时删除。
{% prism javascript linenos %}
// webpack-hot-middleware/middleware.js

function(req, res) {
    req.socket.setKeepAlive(true);
    res.writeHead(200, {
        'Access-Control-Allow-Origin': '*',
        'Content-Type': 'text/event-stream;charset=utf-8',
        'Cache-Control': 'no-cache, no-transform',
        'Connection': 'keep-alive',
        'X-Accel-Buffering': 'no'
    });
    res.write('\n');
    var id = clientId++;
    clients[id] = res;
    req.on("close", function(){
        delete clients[id];
    });
}
{% endprism %}

向每个连接发送心跳事件是通过定时器完成的。值得一提的是`unref()`的用法，使这个定时器不会影响事件循环的退出，如果没有其他事件进入队列，事件循环将结束，进程就退出了。更多关于`unref()`的说明，可以参考这个[回答](https://cnodejs.org/topic/570924d294b38dcb3c09a7a0#5709f1b8bc564eaf3c6a48e1)。
{% prism javascript linenos %}
// webpack-hot-middleware/middleware.js

setInterval(function heartbeatTick() {
    everyClient(function(client) {
        // 向每个 client 发送心跳字符
        client.write("data: \uD83D\uDC93\n\n");
    });
}, heartbeat).unref();
{% endprism %}

那么在代码发生变动时，如何通知客户端呢？在创建中间件时，为 webpack 编译器添加了对应编译阶段的监听函数：
{% prism javascript linenos %}
// webpack-hot-middleware/middleware.js

compiler.plugin("compile", function() {
    eventStream.publish({action: "building"});
});
compiler.plugin("done", function(statsResult) {
    eventStream.publish({
        name: stats.name,
        action: action,
        time: stats.time,
        hash: stats.hash,
        warnings: stats.warnings || [],
        errors: stats.errors || [],
        modules: buildModuleMap(stats.modules)
    });
});
{% endprism %}

## 客户端热更新

在接收到代码变动后服务端发送的事件后，客户端代码需要进行模块热替换处理。客户端在创建 EventSource 时通过`onmessage`定义了处理服务端消息的函数。其中`building`事件只是简单的输出控制台信息，我们将着重关注`sync`事件。另外通过`customHandler`支持自定义事件，例如可以为 html-webpack-plugin 添加自动刷新页面功能，后续将会介绍具体做法。
{% prism javascript linenos %}
// webpack-hot-middleware/client.js

case "sync":
    processUpdate(obj.hash, obj.modules, options);
    break;
default:
    if (customHandler) {
        customHandler(obj);
    }
{% endprism %}

我们通过比对当前代码的 hash 和服务端发来的 hash 判断代码是否发生了更新。当前代码的 hash 是通过 webpack 的[全局模块变量](https://doc.webpack-china.org/api/module-variables/#__webpack_hash__-webpack-specific-)`/* global window __webpack_hash__ */`在编译时注入。
{% prism javascript linenos %}
// webpack-hot-middleware/process-update.js

/* global window __webpack_hash__ */
function upToDate(hash) {
    if (hash) lastHash = hash;
    return lastHash == __webpack_hash__;
}
{% endprism %}

当发现 hash 不匹配，也就是代码发生了更新时，首先需要获取最新的代码，这依赖 webpack 模块的[hot.check()](https://doc.webpack-china.org/api/hot-module-replacement#check)接口。调用后将发送一个 HTTP 请求用来获取更新后的模块信息，触发回调函数。
{% prism javascript linenos %}
var result = module.hot.check(false, cb);
if (result && result.then) {
    result.then(function(updatedModules) {
        cb(null, updatedModules);
    });
    result.catch(cb);
}
{% endprism %}

在回调函数中，如果没有报错，就调用模块的 `hot.apply()` 进行热替换。要注意正常的热替换是不需要触发浏览器刷新页面的，会执行代码中`if (module.hot)`条件分支的热替换逻辑。
详细信息可以参考[ Webpack 文档的介绍](https://doc.webpack-china.org/guides/hot-module-replacement)或者 style-loader 的实现。
{% prism javascript linenos %}
var cb = function(err, updatedModules) {
    if (err) return handleError(err);
    var applyResult = module.hot.apply(applyOptions, applyCallback);
}
{% endprism %}

至此我们了解了代码热更新时，服务端和客户端的整个通信流程。

## 为 html-webpack-plugin 添加自动刷新

最后让我们来看一个开发中具体的问题。在使用 html-webpack-plugin 插件生成 HTML 的场景中，我们希望做到修改模版自动刷新页面，节省手动刷新的时间。这里使用到了之前 webpack-hot-middleware 客户端代码接收自定义更新事件 `customHandler`。在使用中间件的服务端代码中，每次 HTML 生成后，都会向客户端发送一个自定义事件 `reload`。
{% prism javascript linenos %}
// 使用 webpack-hot-middleware 的服务端代码

compiler.plugin('compilation', (compilation) => {
    compilation.plugin('html-webpack-plugin-after-emit', (data, cb) => {
        hotMiddleware.publish({
            action: 'reload'
        });
        cb();
    });
});
{% endprism %}

在客户端 entry 代码中，只需要使用 `subscribe` 订阅该自定义事件即可。另外，这里使用到了 Webpack 模块的 query 参数。关于 `name`, `noInfo`, `reload` 这些客户端参数的含义，可以参考 webpack-hot-middleware 的[文档说明](https://github.com/glenjamin/webpack-hot-middleware#config)。
{% prism javascript linenos %}
// 客户端 entry 代码

import 'eventsource-polyfill';
import hotClient from 'webpack-hot-middleware/client?name=compilerName&noInfo=true&reload=true';

hotClient.subscribe(payload => {
    if (payload.action === 'reload' || payload.reload === true) {
        window.location.reload();
    }
});
{% endprism %}

在 vuejs-templates 官方 Webpack 模版中，就采用了这种做法。但是其中有一个[ISSUE](https://github.com/vuejs-templates/webpack/issues/751#issuecomment-309955295)值得关注，包括我自己在使用时，也遇到了同样的问题。

## 参考资料

* [Webpack HMR 官方介绍](https://doc.webpack-china.org/concepts/hot-module-replacement/)
* [vuejs-templates issue](https://github.com/vuejs-templates/webpack/issues/751#issuecomment-309955295)
