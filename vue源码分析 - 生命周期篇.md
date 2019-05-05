# vue源码分析 - 生命周期篇

在我们实际项目开发过程中，会非常频繁地和 Vue 组件的生命周期打交道，今天，从源码的角度来看一下这些生命周期的钩子函数是如何被执行的。

首先，先看一张很熟悉的图：
![生命周期图](./img/2-1.jpeg)

我们根据源码来对照这个图来看每一步是如何实现的
开始，new Vue（）做了哪些事情？
我们来看一下源码，在src/core/instance/index.js 中

``` javascript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```


好了,就写到这了，希望看过后对你能有帮助。


