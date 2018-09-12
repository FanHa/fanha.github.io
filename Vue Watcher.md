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
一般简单的观察者模式(教科书),`观察者`订阅`被观察者`的变化,并传入回调,`被观察者`变化时遍历执行所有观察者的回调.  
Vue在观察者 和 被观察者 中间加了一层 Dep,便于更好的管理和优化订阅关系;  
每个被观察的属性都对应新建了一个 Dep, 订阅和通知(回调)都通过Dep进行. 
```js
// core/observer/dep.js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

Dep.target = null
const targetStack = []

export function pushTarget (_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

Dep类中有static的全局属性target,可接受的值为Watcher,初始值为null,观察者Watcher订阅被观察者的变化时,先调用`pushTarget`把自己放在全局的target里,然后调用属性的get时会触发属性`dep.depend`;
```js
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        // ...
      }
      return value
    },
```

`dep.depend()`回调了target所指的Watcher的`addDep()`,
```js
//dep.js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```
在Watcher中`addDep`会回调`dep.addSub()`,这样属性对应的dep.subs中就保存了所有订阅了该属性的Watcher,
```js
//watcher.js
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```
在属性的`set`函数里,会触发`dep.notify`,遍历执行了属性`dep.subs`中所有watcher的update()回调;
```js
    set: function reactiveSetter (newVal) {
      //...
      dep.notify()
    }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()

    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```

#### Watcher
当有了可被观察的对象 vm.data时,调用 `new Watcher(vm, expOrFn, cb)`触发观察者对被观察对象的订阅;  
在Watcher的创建过程中调用了`Dep`里的`pushTarget`和`popTarget()`指定了当前观察者,然后调用了传入的`expOrFn`触发被观察对象的get,收集观察者,并提供对外接口`update()`供被观察者set数据时回调执行;
```js
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  computed: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  dep: Dep;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    //...
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()

    // ...
    this.value = this.get()

  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }


  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.computed) {
      //...
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      this.getAndInvoke(this.cb)
    }
  }

  getAndInvoke (cb: Function) {
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      this.dirty = false
      if (this.user) {
        try {
          cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```
