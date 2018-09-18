### Vue 周期
Vue的一切从`new Vue(...)`开始,`new Vue(...)`会返回一个`_init`后的Vue实例
```js
// core/instance/index.js
function Vue (options) {
  this._init(options)
}

/** 下面这一段时给Vue混入一些功能, 
    如initMixin 混入了 上面调用的this._init() **/
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```


#### Vue 的 init
```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
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

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

Vue首先会初始化组件自身的事件和数据,组件的prop,data等数据加上reactive属性是在`initState`阶段,`observe`函数即将组件内的数据对象转成了可被观测的对象
```js
// core/instance/state.js
export function initState (vm: Component) {
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
}

function initData (vm: Component) {
  let data = vm.$options.data
  observe(data, true /* asRootData */)
}
```
然后调用$mount,加载`template`(或render)里的内容,加载内容时会触发已被观测对象的get方法,从而收集并订阅了该对象属性的变化.  
>> TODO: to Vue Watcher

#### 层次初始化
Vue Component 的初始化时深度优先遍历,
```html
//组件A的结构
<template>
  <B></B>
  <C></C>
<template>

//组件B的结构
<template>
  <b1></b1>
  <b2></b2>
</template>
```
+ 先执行A的init,即上面的流程,
+ 然后vm.$mount,
+ 这时A会为B,C分别创建VNode占好位置,并把B,C初始化时需要的信息放在VNode里,
  + 依次处理VNodeB,VNodeC里的内容的,
    + 在处理VNodeB时触发了B组件的init,
      + 同样init了B组件的data,prop等信息后调用mount加载template依次类推到b1,b2
    + 处理VNodeC
      + ...

每一层都只初始化了自身的内容,如果存在子组件,则只是给子组件占个位置,把子组件初始化时需要的内容打包给子组件,具体的初始化由子组件自行处理;  
顺序 A->B->b1->b2->C
>> TODO: to Vue vdom
