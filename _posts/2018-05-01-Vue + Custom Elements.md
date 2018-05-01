---
layout: post
title: "Vue + Custom Elements"
cover: "/assets/img/vue-custom-element-logo-text.png"
date:   2018-05-01
category: coding
tags: vue WebComponents
author: xiaOp
comments: true
---

都知道 Polymer 以 WebComponents 为基础，其实 Vue 也有相关的插件。

Web Components 由 4 个标准组成：
* [HTML imports](https://www.html5rocks.com/en/tutorials/webcomponents/imports/)
* [HTML template](https://www.html5rocks.com/en/tutorials/webcomponents/template/)
* Custom Elements
* Shadow DOM

前两个标准算是基础部分，先介绍下 Custom Elements，值得一提的是这个标准本身也发生过重大修改，最新的版本是 v1。

## Custom Elements 介绍

我们知道浏览器遇到不认识的标签会忽略，Custom Elements 可以帮助扩充浏览器的词汇表。

先来看一下用法，我们可以使用 ES6 的 class 语法声明一个自定义标签，然后使用全局 API 进行注册。这样在 HTML 中，我们就可以直接使用 `<app-drawer>` 自定义标签了：
{% prism javascript linenos %}
class AppDrawer extends HTMLElement {...}
window.customElements.define('app-drawer', AppDrawer);
{% endprism %}

用法很简单，重点是标签类的内容。在深入之前我们不妨思考一个问题，那就是如何将 JS 中的变量修改反映到 HTML 上，反之亦然。
为什么要这么做呢，原因有两个。首先是双向绑定的需要，其次在 HTML 上添加属性便于实现样式的切换。

我们希望在 CSS 中通过 HTML 属性改变样式：
{% prism javascript linenos %}
app-drawer[disabled] {
    opacity: 0.5;
    pointer-events: none;
}
{% endprism %}

带着这个问题我们看看 API 都提供了哪些功能。

### 生命周期钩子

在自定义元素中，提供了若干生命周期钩子供我们扩展，其中包括：
* constructor
* connectedCallback/disconnectedCallback 插入/移除 DOM
* attributeChangedCallback 属性添加、移除、更新或替换

使用这些钩子我们就能完成 HTML 属性和 JS 变量的绑定。

### 映射 HTML property 到 JS attribute

首先，在 JS 中可以通过同名变量 getter/setter 设置 HTML 属性，类似 Vue 中的计算属性。
{% prism javascript linenos %}
get disabled() {
    return this.hasAttribute('disabled');
}

set disabled(val) {
    if (val) {
        this.setAttribute('disabled', '');
    } else {
        this.removeAttribute('disabled');
    }
    // this.toggleDrawer();
}
{% endprism %}

反过来从 HTML 属性到 JS 变量就需要依靠之前的 `attributeChangedCallback` 钩子了。要触发这个钩子，必须先声明要监听的 HTML 属性，类似 Vue 中使用 `data()` 声明响应式数据：
{% prism javascript linenos %}
static get observedAttributes() {
    return ['disabled', 'open'];
}
{% endprism %}

然后在钩子中就可以将 HTML 属性的变化值同步到 JS 变量中了：
{% prism javascript linenos %}
attributeChangedCallback(name, oldValue, newValue) {}
{% endprism %}

### 自定义元素内容

有了数据绑定机制，自定义元素的内容应该如何设置呢？

首先想到，由于 `this` 可以访问 DOM API，所以可以在 `connectedCallback` 钩子中直接插入内容：
{% prism javascript linenos %}
connectedCallback() {
    this.innerHTML = "<b>I'm an x-foo-with-markup!</b>";
}
{% endprism %}

## Vue + Custom Elements

从前面的介绍我们能看出来，虽然提供了 HTML 属性到 JS 变量的映射机制，但是 HTML template 并没有双向绑定语法，在使用时还是免不了大量 DOM 操作。

Polymer 为了解决这些问题做了大量的封装。Vue 的一个插件 [vue-custom-element](https://github.com/karol-f/vue-custom-element) 也能帮助我们更加便捷地使用 Custom Elements。
另外，我们之前省略了 Shadow DOM 在组件封装上的好处。

{% responsive_image path: assets/img/vue-custom-element-why.png alt: "Vue + Custom Elements" %}

### 使用方法

用法有点类似将 `.vue` 中的部分模版挪到 HTML 中直接使用，而且由于不存在挂载点，也不需要手动执行 `$mount()` 操作。
另外，将 `props` 属性绑定到了 DOM 上，修改可以直接触发 HTML 内容改变。
{% prism javascript linenos %}
<widget-vue prop1="1" prop2="string" prop3="true"></widget-vue>

Vue.customElement('widget-vue', {
    props: [
        'prop1',
        'prop2',
        'prop3'
    ],
    data: {
        message: 'Hello Vue!'
    },
    template: '<p>{{ message }}, {{ prop1  }}, {{prop2}}, {{prop3}}</p>'
});

document.querySelector('widget-vue').prop2 // get prop value
document.querySelector('widget-vue').prop2 = 'another string' // set prop value
{% endprism %}

来看看内部对于 Custom Elements 的封装细节吧。

### 内部实现

## 参考资料

* [Custom Elements v1 支持度](https://caniuse.com/#feat=custom-elementsv1) iOS 10+
* [Custom Elements v1 概览](https://developers.google.com/web/fundamentals/web-components/customelements)
