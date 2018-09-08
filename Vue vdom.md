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
1. 通过create里的方法构建新的VNode
2. 通过patch函数更新DOM
    + patch函数会根据新旧VNode的差异对更新DOM的方式进行优化,减少实际更新DOM的次数
#### create
`create-component.js`,`create-element.js`,`create-functional-component.js`用来创建不同类型的Vnode.
#### patch
patch.js 中createPatchFunction 返回一个patch函数,外部通过调用`patch(oldVnode, newVnode, ...)`:
+ 更新Vnode;
+ 经过处理后,更新该Vnode中必要更新的DOM;

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

Vue 解析了template生成了一个新的`VNode`时,后者运行中被其他方式(如wartch变量变动)改变了`VNode`,就会调用`_update`执行`__patch__`(即上面生成的patch函数);  

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

上面这些操作都是会实装生成(或修改)了DOM,并将新DOM插入父DOM,即调用
`insert(parentElm, vnode.elm, refElm)`
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

##### createComponent

##### patchVnode 修改





