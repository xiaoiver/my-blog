---
layout: post
title: "Vue Transition 分析"
cover: "/assets/img/vue-render.png"
date:   2018-05-04
category: coding
tags: vue
author: xiaOp
comments: true
index: 44
---

有了之前对于 Vue 整个渲染流程的分析，我们可以深入研究一下 `<transition>` 的实现原理。

Vue 渲染机制分析：
* [从 HTML 字符串到 render 函数]({{ site.baseurl }}{% link _posts/2018-04-27-Vue 渲染机制.md %})
* [从 render 函数到 VNode]({{ site.baseurl }}{% link _posts/2018-04-29-Vue 渲染机制（二）.md %})
* [从 VNode 到 DOM]({{ site.baseurl }}{% link _posts/2018-04-30-Vue 渲染机制（三）.md %})

## 使用方法

先来简单看下使用方法。按照 [Vue Transition 文档](https://vuejs.org/v2/guide/transitions.html)的介绍，`<transition>` 可以应用在下列元素或者组件中：
* 条件渲染 (使用 v-if)
* 条件展示 (使用 v-show)
* 动态组件
* 组件根节点

在进入具体分析之前，先来看下 Vue 中强制触发渲染流程的方法：
{% prism javascript linenos %}
// src/core/instance/lifecycle.js

Vue.prototype.$forceUpdate = function () {
    const vm = this;
    if (vm._watcher) {
        vm._watcher.update();
    }
};
{% endprism %}

之前介绍过的 Watcher，创建时我们传入了回调函数（第二个参数），调用实例上的 `update()` 时就会触发这个回调函数，完成从 render 函数到 VNode 再到 DOM 的渲染流程。
{% prism javascript linenos %}
// src/platforms/web/runtime/index.js

export function mountComponent(vm, el, hydrating) {
    let updateComponent = () => {
        vm._update(vm._render(), hydrating);
    };
    vm._watcher = new Watcher(vm, updateComponent, noop);
}
{% endprism %}

## 生成 VNode 阶段

在这一阶段中，会执行 render 函数得到 VNode。

内置的 `<transition>` 是一个抽象组件（abstract）。[Vue 文档中是没有抽象组件的](https://github.com/vuejs/vuejs.org/issues/720)，应该是 Vue 的内置组件才会用到，比如还有 `<keep-alive>`。
{% prism javascript linenos %}
// src/platforms/web/runtime/components/transition.js

export default {
    name: 'transition',
    props: transitionProps,
    abstract: true,
    render(h) {...}
}
{% endprism %}

### 获取子 VNode 节点

进入 render 函数，结合前面的渲染分析，我们知道 VNode 的生成顺序，先子节点再父节点。
所以这里 `_renderChildren` 一定就是子节点数组了。
另外，transition 只支持单子节点，否则会报警告，后续的处理也仅针对第一个子节点进行。
{% prism javascript linenos %}
let children = this.$options._renderChildren;
if (!children) {
    return;
}
// 只处理第一个子节点
const rawChild = children[0];
{% endprism %}

### 生成 ID

{% prism javascript linenos %}
const id = `__transition-${this._uid}-`;
    child.key = child.key == null
        ? child.isComment
            ? id + 'comment'
            : id + child.tag
        : isPrimitive(child.key)
            ? (String(child.key).indexOf(id) === 0 ? child.key : id + child.key)
            : child.key;
{% endprism %}

### 解析属性 & 标记

在 transition 切换过程中，前后两个节点都需要渲染，旧节点在 `_vnode` 上。
之所以还要调用 `getRealChild()`，是因为子元素有可能还是一个抽象节点，例如 `<keep-alive>`，还需要进一步获取真实元素。
属性保存在 `data.transition` 对象中。
{% prism javascript linenos %}
const data = (child.data || (child.data = {})).transition = extractTransitionData(this);
const oldRawChild = this._vnode;
const oldChild = getRealChild(oldRawChild);
{% endprism %}

出现 `v-show` 指令，标记在 data 上：
{% prism javascript linenos %}
if (child.data.directives && child.data.directives.some(d => d.name === 'show')) {
    child.data.show = true;
}
{% endprism %}

### 过渡模式

默认情况下，前后两个元素的 transition 过渡效果是同时发生的。
对于需要设置先后顺序的场景，提供了[过渡模式](https://cn.vuejs.org/v2/guide/transitions.html#%E8%BF%87%E6%B8%A1%E6%A8%A1%E5%BC%8F)。

对于 `out-in` 也就是当前元素先进行过渡，完成之后新元素过渡进入的情况。
在 VNode 的 `afterLeave` 钩子中触发强制更新。
{% prism javascript linenos %}
if (this._leaving) {
    // 处理 keep-alive，其他元素直接 return
    return placeholder(h, rawChild);
}
if (mode === 'out-in') {
    // return placeholder node and queue update when leave finishes
    this._leaving = true;
    mergeVNodeHook(oldData, 'afterLeave', () => {
        this._leaving = false;
        this.$forceUpdate();
    });
    return placeholder(h, rawChild);
}
{% endprism %}

真实的 DOM 操作都定义在 VNode 上的钩子中，在下一个 patch 阶段执行。

## patch 阶段

在这个阶段中，会调用 VNode 上的一些钩子，主要涉及具体的 DOM 操作。

在前面的文章中介绍过，patch 阶段支持如下钩子，在创建/ Diff 更新/删除 VNode 的各个阶段会调用相应的钩子：
{% prism javascript linenos %}
// src/core/vdom/patch.js

const hooks = ['create', 'activate', 'update', 'remove', 'destroy'];
{% endprism %}

其中 transition 在 VNode 的创建和删除阶段定义了如下钩子：
{% prism javascript linenos %}
// src/platforms/web/runtime/modules/transition.js

export default inBrowser ? {
    create: _enter,
    activate: _enter,
    remove(vnode, rm) {
        if (vnode.data.show !== true) {
            leave(vnode, rm);
        }
        else {
            rm();
        }
    }
} : {};
{% endprism %}

### 进入阶段

前面介绍过，处理条件展示，使用 `v-show`：
{% prism javascript linenos %}
function _enter(_, vnode) {
    if (vnode.data.show !== true) {
        enter(vnode);
    }
}
{% endprism %}

Vue 支持 CSS 动画和 JS 动画两种。先来看看 Vue 最常用的 CSS 动画。
默认情况下 Vue 会监听 CSS Transition 结束事件，动画效果完成后自动调用结束钩子。同时也支持用户显式传入 [duration](https://cn.vuejs.org/v2/guide/transitions.html#%E6%98%BE%E6%80%A7%E7%9A%84%E8%BF%87%E6%B8%A1%E6%8C%81%E7%BB%AD%E6%97%B6%E9%97%B4)，这时会使用 `setTimeout` 按照用户的意愿结束动画。
{% prism javascript linenos %}
if (expectsCSS) {
    addTransitionClass(el, startClass);
    addTransitionClass(el, activeClass);
    nextFrame(() => {
        addTransitionClass(el, toClass);
        removeTransitionClass(el, startClass);
        if (!cb.cancelled && !userWantsControl) {
            // 显式定义了持续时间
            if (isValidDuration(explicitEnterDuration)) {
                setTimeout(cb, explicitEnterDuration);
            }
            // 监听 transition 结束事件
            else {
                whenTransitionEnds(el, type, cb);
            }
        }
    });
}
{% endprism %}

### 监听过渡效果结束

这段代码可谓非常巧妙。考虑到了下面的情况：
1. transitionEnd 的事件的浏览器兼容性。这个比较简单，检测后加上 Webkit 前缀就行
2. 可能定义了**多个**动画属性，持续时间各异。
3. 规定时间未执行完，需要强制结束。

实际做法如下：
* 首先通过 `getTransitionInfo()` 解析 DOM 元素上的样式对象，拿到定义在上面的类型（transition/animation），持续时间（各个属性持续时间的最大值）和属性数目。
* 监听 `transitionEnd` 和 `animationEnd` 事件，触发一次动画属性计数加一，全部动画属性都完成才调用 `end()`。
* 根据上面拿到的持续时间，设定一个定时器，如果到期时还有动画没执行完，强制 `end()`。
{% prism javascript linenos %}
export function whenTransitionEnds(
    el,
    expectedType,
    cb
) {
    const {
        type,
        timeout,
        propCount
    } = getTransitionInfo(el, expectedType);
    if (!type) {
        return cb();
    }

    const event = type === TRANSITION ? transitionEndEvent : animationEndEvent;
    let ended = 0;
    const onEnd = e => {
        if (e.target === el) {
            // 全部属性的动画都执行完成，结束
            if (++ended >= propCount) {
                end();
            }
        }
    };
    const end = () => {
        el.removeEventListener(event, onEnd);
        cb();
    };
    setTimeout(() => {
        // 此时还有属性的动画没有执行完成，强制结束
        if (ended < propCount) {
            end();
        }
    }, timeout + 1);
    el.addEventListener(event, onEnd);
}
{% endprism %}

### JS 钩子

前面介绍了 transition 支持的 CSS 动画。对于需要精确化控制的复杂场景，需要使用[ JS 钩子](https://cn.vuejs.org/v2/guide/transitions.html#JavaScript-%E9%92%A9%E5%AD%90)。

值得一提的是在 CSS 动画中也可以使用这些钩子。在这种情况下 `enter` 和 `leave` 这两个钩子都是可选的，因为进入离开过程中的操作已经由 CSS transition 完成。

如果希望[完全控制](https://cn.vuejs.org/v2/guide/transitions.html#JavaScript-%E9%92%A9%E5%AD%90)，需要显式传入 `v-bind:css="false"`。这时候使用 `enter` 和 `leave` 这两个钩子就能进行复杂的操作了。
