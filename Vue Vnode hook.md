### hook种类
```ts
//src/core/vdom/patch.js
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
```

### 注入hook
在`createPatchFunction`时,将传进来的参数里的`hook`内容按不同类别写入`cbs`
```ts
//src/core/vdom/patch.js
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    // 这里看出hook的具体内容由调用出reatePatchFunction的上一层传入的参数 backend决定
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  //...
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    //...
  }
}
```

### patch内部调用hook的方式
```ts
  function reactivateComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i
    let innerNode = vnode
    while (innerNode.componentInstance) {
      innerNode = innerNode.componentInstance._vnode
      if (isDef(i = innerNode.data) && isDef(i = i.transition)) {
        // active hook 调用
        for (i = 0; i < cbs.activate.length; ++i) {
          cbs.activate[i](emptyNode, innerNode)
        }
        insertedVnodeQueue.push(innerNode)
        break
      }
    }
    insert(parentElm, vnode.elm, refElm)
  }

  function invokeCreateHooks (vnode, insertedVnodeQueue) {
    // create hook调用
    for (let i = 0; i < cbs.create.length; ++i) {
      cbs.create[i](emptyNode, vnode)
    }
    //...
  }

  function invokeDestroyHook (vnode) {
    let i, j
    const data = vnode.data
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
      // destroy hook调用
      for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
    }
    if (isDef(i = vnode.children)) {
      //递归 广度优先调用子node的destroy hook
      for (j = 0; j < vnode.children.length; ++j) {
        invokeDestroyHook(vnode.children[j])
      }
    }
  }

  function removeAndInvokeRemoveHook (vnode, rm) {
    if (isDef(rm) || isDef(vnode.data)) {
      //...
      // remove hook 调用
      for (i = 0; i < cbs.remove.length; ++i) {
        cbs.remove[i](vnode, rm)
      }
      if (isDef(i = vnode.data.hook) && isDef(i = i.remove)) {
        i(vnode, rm)
      } else {
        rm()
      }
    } else {
      removeNode(vnode.elm)
    }
  }

  function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    // ...
    if (isDef(data) && isPatchable(vnode)) {
      // update hook 调用
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    // ...
  }
```

### hook的实际内容
从前面得知hook的实际内容由调用`createPatchFunction`的地方负责组装进`modules`参数中,目前vue支持两种平台(web,weex),以web平台为例.
```ts
//src/platforms/web/runtime/patch.js
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// 这里可知hook分为platformModules 和 baseModules
//
const modules = platformModules.concat(baseModules)

export const patch: Function = createPatchFunction({ nodeOps, modules })
```

### plateformModules
平台module的不同子module
```ts
// src/platforms/web/runtime/modules/index.js
import attrs from './attrs'
import klass from './class'
import events from './events'
import domProps from './dom-props'
import style from './style'
import transition from './transition'

export default [
  attrs,
  klass,
  events,
  domProps,
  style,
  transition
]

```

#### attrs
```ts
// src/platforms/web/runtime/modules/attrs.js
// 注册了create 和 update 两个hook函数,都指向updateAttrs
export default {
  create: updateAttrs,
  update: updateAttrs
}

function updateAttrs (oldVnode: VNodeWithData, vnode: VNodeWithData) {

  let key, cur, old
  const elm = vnode.elm
  const oldAttrs = oldVnode.data.attrs || {}
  let attrs: any = vnode.data.attrs || {}
  // clone observed objects, as the user probably wants to mutate it
  if (isDef(attrs.__ob__)) {
    attrs = vnode.data.attrs = extend({}, attrs)
  }

  // 新的属性与旧的属性比较,不同则替换
  for (key in attrs) {
    cur = attrs[key]
    old = oldAttrs[key]
    if (old !== cur) {
      setAttr(elm, key, cur)
    }
  }
  // #4391: in IE9, setting type can reset value for input[type=radio]
  // #6666: IE/Edge forces progress value down to 1 before setting a max
  /* istanbul ignore if */
  if ((isIE || isEdge) && attrs.value !== oldAttrs.value) {
    setAttr(elm, 'value', attrs.value)
  }
  // 移除element中的过时属性
  for (key in oldAttrs) {
    if (isUndef(attrs[key])) {
      if (isXlink(key)) {
        elm.removeAttributeNS(xlinkNS, getXlinkProp(key))
      } else if (!isEnumeratedAttr(key)) {
        elm.removeAttribute(key)
      }
    }
  }
}

```

#### class
```ts
// src/platforms/web/runtime/modules/class.js
export default {
  // 注册了create 和 update 两个hook 为updateClass
  create: updateClass,
  update: updateClass
}

function updateClass (oldVnode: any, vnode: any) {
  const el = vnode.elm
  const data: VNodeData = vnode.data
  const oldData: VNodeData = oldVnode.data
  if (
    isUndef(data.staticClass) &&
    isUndef(data.class) && (
      isUndef(oldData) || (
        isUndef(oldData.staticClass) &&
        isUndef(oldData.class)
      )
    )
  ) {
    return
  }

  let cls = genClassForVnode(vnode)

  // handle transition classes
  const transitionClass = el._transitionClasses
  if (isDef(transitionClass)) {
    cls = concat(cls, stringifyClass(transitionClass))
  }

  // set the class
  if (cls !== el._prevClass) {
    el.setAttribute('class', cls)
    el._prevClass = cls
  }
}
```

#### events
```ts
// src/platforms/web/runtime/modules/events.js
// 注册了create 与 update 两个hook 为updateDomListeners函数
export default {
  create: updateDOMListeners,
  update: updateDOMListeners
}

function updateDOMListeners (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  const on = vnode.data.on || {}
  const oldOn = oldVnode.data.on || {}
  target = vnode.elm
  normalizeEvents(on)
  updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context)
  target = undefined
}
```