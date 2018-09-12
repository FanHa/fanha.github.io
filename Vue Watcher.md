### 外部使用方式

#### 改造对象变为可观测
Vue 中把一个对象变成可被观察的对象是通过`observe()`,如在组件的initData阶段,observe(data),这样就把整个data对象变成了一个可观察的对象;

```js
// core/instance/state.js
function initData (vm: Component) {
  let data = vm.$options.data
  // ...
  observe(data, true /* asRootData */)
}
```

#### 订阅可观测的对象,并指定回调
当一个Component中的数据可被观测时,可以通过`new Watcher()`把要观测的对象和观测到对象变化时的回调进行绑定,如在组件的mountComponent阶段
```js
// core/instance/lifecycle.js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  //...
  //这里参数传的是带有被观察对象的组件vm
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  //...
}
```

### Observer
`observe()`函数主要通过新建一个Observer类将value对象改造成可被观测
```js
// core/observer/index.js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  //...
  ob = new Observer(value)
  //...
}
```

Observer在创建时,对data里的每一个键值对执行`defineReactive`操作,如果对象的键对应的值是个数组,则遍历数组,对数组深度优先遍历并执行`observe`
```js
/*  */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

#### defineReactive
`defineReactive`通过对对象的元信息变成,改变对象的基本`get`和`set`行为;  
假如一个对象`obj`,  
`let a=obj.a` 即调用了基本get行为,  
`obj.a = 'xxx'` 即调用了基本的 set行为,  
通过改造对象的基本get,可以在数据被使用时触发对该数据的依赖收集,  
改造对象的基本set,可以在数据变幻时触发订阅了该数据变化的观察者的回调.

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        // ...
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      //...
      dep.notify()
    }
  })
}
```

#### Dep
一般简单的观察者模式(教科书),`观察者`订阅`被观察者`的变化,并传入回调,`被观察者`变化时执行回调
