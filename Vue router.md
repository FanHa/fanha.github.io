## Vue 初始化router
Vue 创建时把router实例传进;
同时根据Vue加载组件的机制会调用实例的install方法
```js
new Vue({
  el: '#app',
  router,
  // ...
})
```
### 新建Router实例
```js
// index.js
export default class VueRouter {
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 根据路由的配置来创建匹配映射实例
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode
    // 根据设定的模式来创建history属性的实例,这里控制反转,统一了各History实例的接口调用
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
}
```

### install
```js
// install.js
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {
  // ...
  Vue.mixin({
    // 注册Vue的beforeCreate回调,Vue会在初始化自身的时的beforeCreate阶段调用此回调
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        // 调用router实例的init方法初始化 router
        this._router.init(this)

        // 将_route属性做响应式改造
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  // 给Vue实例注册亮哥全局的属性$router 和 $route, 使客户端代码可以使用this.$router, this.$route来做些与路由相关的操作
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 注册RouterView 和 RouterLink 两个vue组件, 使客户端可以使用<router-view>,<router-link>来组成template
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```

### router 的init 方法
```js
// index.js
  init (app: any /* Vue component instance */) {

    this.app = app

    const history = this.history
    // 根据不同的history模式,调用history的接口方法,跳转到
    // 注:似乎前面猜测的控制反转不彻底....
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    // 调用history实例的listen设置好路由变化时的回调
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }
```
```js
//  src/history/base.js
export class History {
  listen (cb: Function) {
    this.cb = cb
  }
```

### transitionTo
```js
// history/base.js
  transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // 先找到当前地址在前面配置的路由中的匹配项
    const route = this.router.match(location, this.current)
    // 这里加了confirmTransition这一步,是为了减少网络请求
    // 有时URL地址变化,但路由没有变化,并不一定需要重新从服务端获取内容,可以直接在浏览器本地操作
    this.confirmTransition(route, () => {

      // 路由更新完后的回调
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => { cb(route) })
      }
    }, err => {
      // ...
    })
  }

  confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    if (
      isSameRoute(route, current) &&
      // 要跳转的路由与当前路由时相同时,不需要做更多
      route.matched.length === current.matched.length
    ) {
      this.ensureURL()
      return abort()
    }

    // 交叉比对解析出需要更新的路由,需要激活,失活的路由
    // 注: 这里的updated,deactivated,activated都是数组,因此猜测“路由”这个词并不是单指URL对应的某个VUE模块;
    // 一个URL的页面下可能有多个子路由板块,当URL变化时,可能只是页面上的一部分视图需要变化,更新;
    // 由此实现比如 “单页应用” 功能来减少URL变化带来的不必要的与服务器之间的交互
    const {
      updated,
      deactivated,
      activated
    } = resolveQueue(this.current.matched, route.matched)

    // 整理编排好 切换路由时要做的事情
    // Vue-router的术语叫“导航守卫(NavigationGuard)”用来定制路由变化时的周期的行为,使用者可以通过给定的api定制路由各个阶段的行为,
    // 可以类比Vue本身的生命周期
    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),
      // global before hooks
      this.router.beforeHooks,
      // in-component update hooks
      extractUpdateHooks(updated),
      // in-config enter guards
      activated.map(m => m.beforeEnter),
      // async components
      resolveAsyncComponents(activated)
    )

    this.pending = route
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      try {
        hook(route, current, (to: any) => {
          if (to === false || isError(to)) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' && (
              typeof to.path === 'string' ||
              typeof to.name === 'string'
            ))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            abort()
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // confirm transition and pass on the value
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }

    runQueue(queue, iterator, () => {
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => { cb() })
          })
        }
      })
    })
  }
```

### History hash模式 与 html5 模式
```js
    if (history instanceof HTML5History) { // HTML5 模式时直接触发路由跳转
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      // Hash 模式时注入了setupHashListener回调,这个回调会在路由变化完成时调用
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }
```
```js
// src/history/hash.js
  setupListeners () {
    // 监听URL的变化,这样当URL变化时,不会直接发送http请求,而执行transitionTo的逻辑,实现了本地路由变化
    window.addEventListener(supportsPushState ? 'popstate' : 'hashchange', () => {
      const current = this.current
      if (!ensureSlash()) {
        return
      }
      this.transitionTo(getHash(), route => {
        if (supportsScroll) {
          handleScroll(this.router, route, current, true)
        }
        if (!supportsPushState) {
          replaceHash(route.fullPath)
        }
      })
    })
  }
```
