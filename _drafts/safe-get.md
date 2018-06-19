if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode);
}

[如何优雅安全地在深层数据结构中取值](https://zhuanlan.zhihu.com/p/27748589)

[](https://github.com/tc39/proposal-optional-chaining)