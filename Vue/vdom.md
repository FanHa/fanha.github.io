### Vnode 结构
VNode 对更底层的DOM进行了一层封装,拦截上层对底层dom的操作,经过自身的机制,算法减少实际的DOM操作

Vue操作VNode一般是先new VNode,占个位置,把整体结构搭好,但VNode并没有完全初始化好,然后在后面的'某个时候'再把VNode里的信息完善,并实装到真正的DOM

+ 生成VNode整体结构,里面包含一层层的parent,sibling,children等
+ 初始化页面,或者更新页面时,通过`__patch__`对VNode结构进行操作,一层层比对并更新DOM
```js
// vnode.js
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }
}
```


### 外部使用接口
Vue改变Dom的方式
1. 通过create-xxx.js里的方法构建新的VNode
2. 通过patch函数更新DOM
    + patch函数会根据新旧VNode的差异对更新DOM的方式进行优化,减少实际更新DOM的次数

### VNode(Component)使用流程 组合模式
通常我们使用Vue时是用A包含一个B,而B组件本身也是一个包含C,D的组件这样的组合方式
```html
/* A.vue */
<A>
  <B></B>
</A>

/* B.vue */
<B>
  <C></C>
  <D></D>
</B>
```

深度优先递归创建

+ 创建一个A 的VNode,里面带有A的元信息;
+ patch A的VNode,这是根据A的元信息 发现有B(以及其他的data,prop,on,回调等等),初始化这些信息,创建B的初始Vnode,并附上B的元信息;
  + 触发B 的patch;
    + 创建C 的初始VNode;
      + 触发C 的patch
        + ...
    + 创建D 的初始Vnode;
      + 触发D 的patch  
        + ...

更新页面时也是通过先改变VNode里的信息,然后通过patch ,一层层将信息推进到底层的实际DOM并更新

#### create
`create-component.js`,`create-element.js`,`create-functional-component.js`用来创建不同类型的初始Vnode.这个Vnode包含了必要的信息(和一个占位但暂时没有实际意义的DOM?)


```js
// core/instance/render.js
// 这个_render函数实际调用的是编写vue组件时下载<script>里的render函数,如果没有render函数,vue也会在上一层先把<template>里的内容统一转换成render函数,便于这里的Vue.prototype._render调用
  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    let vnode
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {

    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
```

render函数的格式和实际调用如下,初始化了一些信息,并新建了VNode
>> createComponent也是新建一种Component类型的Vnode,调用的就是create-component.js里的接口
```js
// core/vdom/create-element.js
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}

export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {

  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {

    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

有了新建VNode的入口后,在组件的mount阶段将`vm._render`作为参数放在`vm._update`回调里,供程序的其他部分(观察者)在某个时刻(新建或更新)时调用(即patch)
```js
// platforms/web/runtime/index.js
// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

// core/instance/lifecycle.js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }

  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
}
```

#### patch
patch.js 中createPatchFunction 返回一个patch函数,外部通过调用`patch(oldVnode, newVnode, ...)`:
+ 解析VNode里的信息
+ 根据解析的信息递归创建子VNode 或 更新实际DOM;
  + 递归创建的子VNode的解析和处理

```js
// patch.js
export function createPatchFunction (backend) {
    /* 一系列供下面组装调用的内部函数
    ... 
    */
    return function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
        // ...
    }
}
```

Vue 解析了template生成了一个新的`VNode`时,后者运行中被其他方式(如watch变量变动)改变了`VNode`,就会调用`_update`执行`__patch__`(即上面生成的patch函数);  

```js
// /core/instance/lifecycle.js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    //...
    vm.$el = vm.__patch__(prevVnode, vnode)
    //...
}

```

patch VNode 通常有3种情况
1. 新增,即prevVnode 不存在, vnode存在;
2. 修改,即prevVnode 和 vnode 都存在;
3. 删除,即prevVnode 存在, vnode不存在;
```js
// patch.js
 if (isUndef(oldVnode)) {
     // ...
     // 新增
      createElm(vnode, insertedVnodeQueue, parentElm, refElm)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // 修改(包括删除)
        patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
      } else {
        // ...
      }
    }
```

##### createElm 新增
通常有4种类型的vnode
1. component,即VNode本身代表的是个Vue的Component结构,调用createComponent
2. 普通的html tag,如"div","span"之类的,创建自身后再调用createChildren(),递归调用createElm创建子节点
3. 注释节点
4. 文字节点

上面这些操作都是会实装生成(或修改)了DOM,并将新DOM插入父DOM(调用
`insert(parentElm, vnode.elm, refElm)`)
```js
  function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    // ...
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }
    // ...
    if (isDef(tag)) {

      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)
      if (__WEEX__) {
        // ...
      }
      else {
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        insert(parentElm, vnode.elm, refElm)
      }

    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }

  // ...
  function createChildren (vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
      // ...
      for (let i = 0; i < children.length; ++i) {
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
      }
    } else if (isPrimitive(vnode.text)) {
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
    }
  }
```
##### patchVnode 修改
//TODO

##### createComponent
通常使用Vue时是抽象了一个虚的tag,如`el-table`,这样生成的VNode不能直接对应一个DOM,此时调用的是CreateComponent建立的Component类型的VNode.
>> 注: 这里的`createComponent` 与 接口`create-component.js`不同,接口`create-component.js` 是新建了一个VNode(并新建了没有实际意义的占位置的DOM),但并没有把VNode代表的内容落实到真正的DOM里,这里的`createComponent`才把VNode里的内容实装到DOM里;

从代码里,createComponent取出了vnode里的data,`let i = vnode.data`,然后
1. 执行了`i(vnode, false /* hydrating */, parentElm, refElm)`,`(isDef(i = i.hook) && isDef(i = i.init))`;
2. `initComponent(vnode, insertedVnodeQueue)`;
3. `reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)`;
可以看出这个createComponent其实是对一个已有的vnode做操作

```js
// patch.js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */, parentElm, refElm)
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
```
##### init vnode
上面的i指向的是vnode.data.init, 而init函数是在create-component.js新建vnode时挂到的vnode.data上的,
调用该init函数会将初始VNode里的信息转换成实例,然后$mount 该实例,递归到了下一个
`创建VNode`->`patch VNode`的流程
```ts
// create-component.js
const componentVNodeHooks = {
  init (
    vnode: VNodeWithData,
    hydrating: boolean,
    parentElm: ?Node,
    refElm: ?Node
  ): ?boolean {
    if (
      //...
    ) {
      // ...
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance,
        parentElm,
        refElm
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
  //...
}

export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
  parentElm?: ?Node,
  refElm?: ?Node
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    parent,
    _parentVnode: vnode,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```







