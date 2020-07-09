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
  // 对不同浏览器的历史版本的兼容
  normalizeEvents(on)

  // 更新listener, 这里add, remove, createOnceHandler都是回调函数
  updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context)
  target = undefined
}
```

```ts
// src/core/vdom/helpers/update-listeners.js
export function updateListeners (
  on: Object,
  oldOn: Object,
  add: Function,
  remove: Function,
  createOnceHandler: Function,
  vm: Component
) {
  let name, def, cur, old, event
  // 便利node中的on 根据参数调用回调add(对createOnce的事件作了前置回调处理)
  for (name in on) {
    def = cur = on[name]
    old = oldOn[name]
    event = normalizeEvent(name)

    if (isUndef(cur)) {
      // ...
    } else if (isUndef(old)) {
      if (isUndef(cur.fns)) {
        cur = on[name] = createFnInvoker(cur, vm)
      }
      if (isTrue(event.once)) {
        cur = on[name] = createOnceHandler(event.name, cur, event.capture)
      }
      add(event.name, cur, event.capture, event.passive, event.params)
    } else if (cur !== old) {
      old.fns = cur
      on[name] = old
    }
  }
  // 便利旧节点中的 on 事件监听,移除过时的监听
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name)
      remove(event.name, oldOn[name], event.capture)
    }
  }
}
```
```ts
// src/platforms/web/runtime/modules/events.js
// add, remove, createOnceHandler 回调
function createOnceHandler (event, handler, capture) {
  const _target = target // save current target element in closure
  return function onceHandler () {
    const res = handler.apply(null, arguments)
    // 触发一次后调用remove
    if (res !== null) {
      remove(event, onceHandler, capture, _target)
    }
  }
}

function add (
  name: string,
  handler: Function,
  capture: boolean,
  passive: boolean
) {
  // ...
  // 这里调用了更基础的addEventListener接口增加了一个事件监听
  target.addEventListener(
    name,
    handler,
    supportsPassive
      ? { capture, passive }
      : capture
  )
}

function remove (
  name: string,
  handler: Function,
  capture: boolean,
  _target?: HTMLElement
) {
  // 取消监听
  (_target || target).removeEventListener(
    name,
    handler._wrapper || handler,
    capture
  )
}
```

#### domProps
```ts
// src/platforms/web/runtime/modules/dom-props.js
// 注册create 和 update 两个hook函数为updateDOMProps
export default {
  create: updateDOMProps,
  update: updateDOMProps
}

function updateDOMProps (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.domProps) && isUndef(vnode.data.domProps)) {
    return
  }
  let key, cur
  const elm: any = vnode.elm
  const oldProps = oldVnode.data.domProps || {}
  let props = vnode.data.domProps || {}
  // clone observed objects, as the user probably wants to mutate it
  if (isDef(props.__ob__)) {
    props = vnode.data.domProps = extend({}, props)
  }

  // 先把过时的Props置为空
  for (key in oldProps) {
    if (!(key in props)) {
      elm[key] = ''
    }
  }

  for (key in props) {
    cur = props[key]
    // 特殊prop(textContent 和 innerHTML)的处理
    if (key === 'textContent' || key === 'innerHTML') {
      if (vnode.children) vnode.children.length = 0
      if (cur === oldProps[key]) continue
      
      if (elm.childNodes.length === 1) {
        elm.removeChild(elm.childNodes[0])
      }
    }

    // 当prop的key 为value时的特殊处理
    if (key === 'value' && elm.tagName !== 'PROGRESS') {
      // store value as _value as well since
      // non-string values will be stringified
      elm._value = cur
      // avoid resetting cursor position when value is the same
      const strCur = isUndef(cur) ? '' : String(cur)
      if (shouldUpdateValue(elm, strCur)) {
        elm.value = strCur
      }
    } else if (key === 'innerHTML' && isSVG(elm.tagName) && isUndef(elm.innerHTML)) {
      // ...
    } else if ( // 普通的prop值的替换
      
      cur !== oldProps[key]
    ) {
      try {
        elm[key] = cur
      } catch (e) {}
    }
  }
}
```

#### style
```ts
// src/platforms/web/runtime/modules/style.js
// 注册create 和update 两个hook函数 为 updateStyle
export default {
  create: updateStyle,
  update: updateStyle
}

function updateStyle (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  const data = vnode.data
  const oldData = oldVnode.data

  let cur, name
  const el: any = vnode.elm
  const oldStaticStyle: any = oldData.staticStyle
  const oldStyleBinding: any = oldData.normalizedStyle || oldData.style || {}
  const oldStyle = oldStaticStyle || oldStyleBinding

  const newStyle = getStyle(vnode, true)

  // 遍历旧的style,将过时的stype设为空
  for (name in oldStyle) {
    if (isUndef(newStyle[name])) {
      setProp(el, name, '')
    }
  }

  // 遍历新style,设置style值到element
  for (name in newStyle) {
    cur = newStyle[name]
    if (cur !== oldStyle[name]) {
      // ie9 setting to null has no effect, must use empty string
      setProp(el, name, cur == null ? '' : cur)
    }
  }
}

const setProp = (el, name, val) => {
  /* istanbul ignore if */
  if (cssVarRE.test(name)) {
    el.style.setProperty(name, val)
  } else if (importantRE.test(val)) {
    el.style.setProperty(hyphenate(name), val.replace(importantRE, ''), 'important')
  } else {
    const normalizedName = normalize(name)
    if (Array.isArray(val)) {
      for (let i = 0, len = val.length; i < len; i++) {
        el.style[normalizedName] = val[i]
      }
    } else {
      el.style[normalizedName] = val
    }
  }
}
```

#### transition
```ts
// src/platforms/web/runtime/modules/transition.js
// 只在浏览器中注册create, activate 和 remove hook函数
export default inBrowser ? {
  create: _enter,
  activate: _enter,
  remove (vnode: VNode, rm: Function) {
    /* istanbul ignore else */
    if (vnode.data.show !== true) {
      leave(vnode, rm)
    } else {
      rm()
    }
  }
} : {}

function _enter (_: any, vnode: VNodeWithData) {
  // 在show属性还没有为true时调用过渡函数enter
  if (vnode.data.show !== true) {
    enter(vnode)
  }
}
```

### baseModules
```ts
// src/core/vdom/modules/index.js
import directives from './directives'
import ref from './ref'

export default [
  ref,
  directives
]

```

#### ref
```ts
// src/core/vdom/modules/ref.js
// create 和 update, destroy节点时需要对组件的 内部的ref作处理
export default {
  create (_: any, vnode: VNodeWithData) {
    registerRef(vnode)
  },
  update (oldVnode: VNodeWithData, vnode: VNodeWithData) {
    if (oldVnode.data.ref !== vnode.data.ref) {
      registerRef(oldVnode, true)
      registerRef(vnode)
    }
  },
  destroy (vnode: VNodeWithData) {
    registerRef(vnode, true)
  }
}

export function registerRef (vnode: VNodeWithData, isRemoval: ?boolean) {
  const key = vnode.data.ref
  if (!isDef(key)) return

  const vm = vnode.context
  const ref = vnode.componentInstance || vnode.elm
  const refs = vm.$refs
  // 第二个参数为true时移除已有ref
  if (isRemoval) {
    if (Array.isArray(refs[key])) {
      remove(refs[key], ref)
    } else if (refs[key] === ref) {
      refs[key] = undefined
    }
  } else {
    if (vnode.data.refInFor) {
      if (!Array.isArray(refs[key])) {
        refs[key] = [ref]
      } else if (refs[key].indexOf(ref) < 0) {
        // $flow-disable-line
        refs[key].push(ref)
      }
    } else {
      refs[key] = ref
    }
  }
}

```

#### directives
```ts
// src/core/vdom/modules/directives.js
// 注册了create 和 update 的hook函数为updateDirectives,
// destroy 的hook函数流程相似,只是把原node替换为一个空node
export default {
  create: updateDirectives,
  update: updateDirectives,
  destroy: function unbindDirectives (vnode: VNodeWithData) {
    updateDirectives(vnode, emptyNode)
  }
}
```