# vue源码分析 - 生命周期篇

在我们实际项目开发过程中，会非常频繁地和 Vue 组件的生命周期打交道，今天，从源码的角度来看一下这些生命周期的钩子函数执行时机与如何被执行的。

项目初始化执行 `var app = new Vue({})` 

### new Vue（）做了哪些事情？

我们来看一下源码，在src/core/instance/index.js 中

```javascript
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
Vue 初始化主要就干了几件事情，合并配置，初始化生命周期，初始化事件中心，初始化渲染，初始化 data、props、computed、watcher 等等。

### 生命周期的定义：

  > 每个 Vue 实例在被创建时都要经过一系列的初始化过程——例如，需要设置数据监听、编译模板、将实例挂载到 DOM 并在数据变化时更新 DOM 等。同时在这个过程中也会运行一些叫做生命周期钩子的函数，这给了用户在不同阶段添加自己的代码的机会。

首先，先看一张很熟悉的图：

![生命周期图](./img/2-2.jpeg)

其中展示了8个生命周期，在vue@2.5.16版本中 又新增了3个，共11个生命周期钩子，下面是每一个的解释：

> · 1、**beforeCreate**：在实例初始化之后，数据观测 (data observer) 和 event/watcher 事件配置之前被调用。 

> · 2、**created**：在实例创建完成后被立即调用。在这一步，实例已完成以下的配置：数据观测 (data observer)，属性和方法的运算，watch/event 事件回调。然而，挂载阶段还没开始，$el 属性目前不可见。 

> · 3、**beforeMount**：在挂载开始之前被调用：相关的 render 函数首次被调用。 

> ·  4、**mounted**：el 被新创建的 vm.$el 替换，并挂载到实例上去之后调用该钩子。如果 root 实例挂载了一个文档内元素，当 mounted 被调用时 vm.$el 也在文档内（PS:注意 mounted 不会承诺所有的子组件也都一起被挂载。如果你希望等到整个视图都渲染完毕，可以用 vm.$nextTick 替换掉 mounted：）。vm.$nextTick会在后面的章节详细讲解，这里大家需要知道有这个东西。 

> ·  5、**beforeUpdate**：数据更新时调用，发生在虚拟 DOM 打补丁之前。这里适合在更新之前访问现有的 DOM，比如手动移除已添加的事件监听器。 

> ·  6、**updated**：由于数据更改导致的虚拟 DOM 重新渲染和打补丁，在这之后会调用该钩子。当这个钩子被调用时，组件 DOM 已经更新，所以你现在可以执行依赖于 DOM 的操作。然而在大多数情况下，你应该避免在此期间更改状态。如果要相应状态改变，通常最好使用计算属性或 watcher 取而代之（PS:计算属性与watcher会在后面的章节进行介绍）。 

> ·  7、**activated**：keep-alive 组件激活时调用（PS：与组件相关，关于keep-alive会在讲解组件的时候为大家介绍）。 

> ·  8、**deactivated**：keep-alive 组件停用时调用（PS：与组件相关，关于keep-alive会在讲解组件的时候为大家介绍）。 

> ·  9、**beforeDestroy**：实例销毁之前调用。在这一步，实例仍然完全可用。 

> ·  10、**destroyed**：Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。 

> ·  11、**errorCaptured**（2.5.0+ 新增）：当捕获一个来自子孙组件的错误时被调用。此钩子会收到三个参数：错误对象、发生错误的组件实例以及一个包含错误来源信息的字符串。此钩子可以返回 false 以阻止该错误继续向上传播。

> ·  原文[https://blog.csdn.net/u011068996/article/details/80970284]

接下来我们通过源码来看一下生命周期是怎么执行的，执行函数怎么实现的？

首先我们在/src/core/instance/init.js中，看下`_init`的方法实现。
```javascript
Vue.prototype._init = function (options?: Object) {
    ...
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    ...
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
 }
```
这里出现了两个生命周期：`beforeCreate` 和 `cteated`

在`beforeCreate`前 先调用了initLifecycle(vm)初始化生命周期、initEvents(vm)初始化事件、initRender(vm)渲染函数

在`beforeCreate` 和 `cteated`之间，调用了initInjections(vm)、initState(vm)、initProvide(vm)这三个方法用于初始化data、props、watcher，也就是说这个时候我们已经可以获取到data，props等数据，但还不能够访问DOM（可以通过vm.$nextTick来访问）

在`cteated`之后, 执行 `vm.$mount(vm.$options.el)` 进行 DOM挂载。这里先引入下，后边将详细介绍。

如果组件在加载的时候需要和后端有交互，放在这俩个钩子函数执行都可以，如果是需要访问 props、data 等数据的话，就需要使用 created 钩子函数。

### 生命周期的执行方式
我们能够发现，源码中最终执行生命周期的函数都是调用 callHook 方法，它的定义在 src/core/instance/lifecycle 中：

```javascript
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]    // 获取Vue选项中的生命周期钩子函数
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)    // 执行生命周期函数，更好的异步错误处理
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)  // 判断是否存在生命周期钩子的事件侦听器
  }
  popTarget()
}
}
```
异常处理的逻辑放在 /src/core/util/error.js 中
```javascript
export function invokeWithErrorHandling (
  handler: Function,
  context: any,
  args: null | any[],
  vm: any,
  info: string
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)  // 根据参数选择不同的handle执行方式
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // 避免嵌套调用时catch多次的触发
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```
created 钩子方式： 
```
callHook(vm, 'created')
```
**解释**：
· 以 pushTarget() 开头，并以 popTarget() 结尾,是为了避免在某些生命周期钩子中使用 props 数据导致收集冗余的依赖。
· 传入hook，获取vm.$options[hook]对应的回调函数数组，遍历执行。
· 各个阶段的生命周期函数会被合并到vm.options中，通过callback回调

**延展**：生命周期钩子的事件侦听器：
```javascript
<child
  @hook:beforeCreate="handleChildBeforeCreate"
  @hook:created="handleChildCreated"
  @hook:mounted="handleChildMounted"
  @hook:生命周期钩子
 />
```
使用 hook: 加 生命周期钩子名称 的方式来监听组件相应的生命周期事件

### DOM挂载：Vue.prototype.$mount原型方法 (mountComponent)



### 页面正常交互: beforeUpdate和updated

这两个钩子函数是在数据更新的时候进行回调的函数。
`beforeUpdate` 的执行时机是在渲染 Watcher 的 before 函数中，代码在/src/core/instance/lifecycle.js下
```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  // ...
}
```
这里可以看出在组件mount过程中，会实例化一个watcher去监听wm上的数据变化从而重新渲染，那么在实例化watcher中，是如何将当前watcher实例赋给wm，代码实现在 src/core/observer/watcher.js 中
```javascript
export default class Watcher {
  // ...
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // ...
  }
}
```
wathcer 实例 push 到 `vm._watchers` 中，`vm._watcher` 是专门用来监听 vm 上数据变化然后重新渲染的，

需要在组件mounted之后调用这个钩子函数，接下来会在 glushSchedulerQueue中执行update，在src/core/observer/scheduler.js中
```javascript
function flushSchedulerQueue () {
  // ...
  // 获取到 updatedQueue
  callUpdatedHooks(updatedQueue)
}
function callUpdatedHooks (queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    // 判断满足当前 watcher 为 vm._watcher 以及组件已经 mounted 才执行updated钩子函数
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'updated')
    }
  }
}
```
也就是说在`callUpdatedHooks`函数中，只有`vm._watcher`回调后执行updated钩子函数。

### 销毁的时候回调：beforeDestroy 、destroyed

完全销毁一个实例。清理它与其它实例的连接，解绑它的全部指令及事件监听器。最终会调用 $destroy 方法，它的定义在 src/core/instance/lifecycle.js 中：
```javascript
Vue.prototype.$destroy = function () {
    const vm: Component = this
    if (vm._isBeingDestroyed) {
      return
    }
    callHook(vm, 'beforeDestroy')
    vm._isBeingDestroyed = true
    // remove self from parent
    const parent = vm.$parent
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
      remove(parent.$children, vm)
    }
    // teardown watchers
    if (vm._watcher) {
      vm._watcher.teardown()
    }
    let i = vm._watchers.length
    while (i--) {
      vm._watchers[i].teardown()
    }
    // remove reference from data ob
    // frozen object may not have observer.
    if (vm._data.__ob__) {
      vm._data.__ob__.vmCount--
    }
    // call the last hook...
    vm._isDestroyed = true
    // invoke destroy hooks on current rendered tree
    vm.__patch__(vm._vnode, null)
    // fire destroyed hook
    callHook(vm, 'destroyed')
    // turn off all instance listeners.
    vm.$off()
    // remove __vue__ reference
    if (vm.$el) {
      vm.$el.__vue__ = null
    }
    // release circular reference (#6759)
    if (vm.$vnode) {
      vm.$vnode.parent = null
    }
  }
```
从 parent 的 $children 中删掉自身，删除 watcher，当前渲染的 VNode 执行销毁钩子函数等，执行完毕后再调用 destroy 钩子函数,
`vm.__patch__(vm._vnode, null)` 触发它子组件的销毁钩子函数，这样一层层的递归调用，所以 destroy 钩子函数执行顺序是先子后父，和 mounted 过程一样。

目前为止，整个Vue生命周期图示中的所有生命周期钩子都已经被执行完成了。那么剩下的activated、deactivated、errorCaptured这三个钩子函数是在何时被执行的呢？

### 新增生命周期： activated、deactivated、errorCaptured
> 其中activated、deactivated这两个钩子函数分别是在keep-alive 组件激活和停用之后回调的，它们不牵扯到整个Vue的生命周期之中
组件一旦被 <keep-alive> 缓存，那么再次渲染的时候就不会执行 created、mounted 等钩子函数，但是我们很多业务场景都是希望在我们被缓存的组件再次被渲染的时候做一些事情，这个时候就需要这两个钩子函数了，他们的定义在src/core/vdom/create-component.js 中（deactivated相同）
```javascript
const componentVNodeHooks = {
  insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },
  // ...
}
```
我们先来看下非mount的情况，函数实现在src/core/instance/lifecycle.js中
```javascript
export function activateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = false
    if (isInInactiveTree(vm)) {
      return
    }
  } else if (vm._directInactive) {
    return
  }
  if (vm._inactive || vm._inactive === null) {
    vm._inactive = false
    for (let i = 0; i < vm.$children.length; i++) {
      activateChildComponent(vm.$children[i])
    }
    callHook(vm, 'activated')
  }
}
```
可以看到这里就是执行组件的 acitvated 钩子函数，并且递归去执行它的所有子组件的 activated 钩子函数。

**errorCaptured** 是唯一一个没有通过callHook方法来执行的钩子函数，直接通过遍历cur(vm).$options.errorCaptured，来执行config.errorHandler.call(null, err, vm, info)的钩子函数
```javascript
export function handleError (err: Error, vm: any, info: string) {
  if (vm) {
    let cur = vm
    while ((cur = cur.$parent)) {
      const hooks = cur.$options.errorCaptured
      if (hooks) {
        for (let i = 0; i < hooks.length; i++) {
          try {
            const capture = hooks[i].call(cur, err, vm, info) === false
            if (capture) return
          } catch (e) {
            globalHandleError(e, cur, 'errorCaptured hook')
          }
        }
      }
    }
  }
  globalHandleError(err, vm, info)
}

function globalHandleError (err, vm, info) {
  if (config.errorHandler) {
    try {
      return config.errorHandler.call(null, err, vm, info)
    } catch (e) {
      logError(e, null, 'config.errorHandler')
    }
  }
  logError(err, vm, info)
}

function logError (err, vm, info) {
  if (process.env.NODE_ENV !== 'production') {
    warn(`Error in ${info}: "${err.toString()}"`, vm)
  }
  /* istanbul ignore else */
  if ((inBrowser || inWeex) && typeof console !== 'undefined') {
    console.error(err)
  } else {
    throw err
  }
}
```
**handleError** 方法中首先获取到报错的组件，之后递归查找当前组件的父组件，依次调用 `errorCaptured` 方法。在遍历调用完所有 errorCaptured 方法、或 errorCaptured 方法有报错时，会调用 globalHandleError 方法。

**globalHandleError** 方法调用了全局的 errorHandler 方法。
如果 errorHandler 方法自己报错了,生产环境下会使用 console.error 在控制台中输出。

(待续)
好了,就写到这了，希望看过后对你能有帮助。


