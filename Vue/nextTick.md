### 过程
```js
// render.js
// $nextTick 把当前Vue组件的上下文环境(this) 和 回调的函数fn 传入 nextTick()
Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
}
```

```js
// core/util/next-tick.js
const callbacks = []
let pending = false


export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 将回调cb push到callbacks队列里
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })

  // 将pending开关置为true,然后执行;
  // 这个开关用来避免重复调用,即callbacks[] 在依次调用时新来的nextTick里的回调先只push进callbacks[],不立即调用
  if (!pending) {
    pending = true
    // macro 和 micro 时HTML5标准的两种任务队列,这里都可以看作是把要执行的回调push到一个特定的队列里
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
}

// 使用Promise.resolve 把最终要执行的callBack(s)压入microTask队列;
// 类比macroTimerFunc 则是用javascript提供的其他机制压入 macro队列;
// 因为javascript使用单线程的event loop来执行程序流, 这里相当于在当前循环通过nextTick(cb)只是把cb里的内容压入另外一个队列假设为A(macroTask or microTask),javascript的执行机制会在当前的程序流事件结束后再去调用A里面的程序流
const p = Promise.resolve()
microTimerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
}

// macroTimefunc
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// 实际执行
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```


### 数据驱动视图更新 为什么nextTick里的函数能取到更新后的dom里的值
+ data里a得初始值为 1;
+ template里绑定了 data里的变量 a;
+ 改变变量 a = 2;
+ 因为变量a的改变到template里值得改变时异步得,这里也是把这个操作压入一个抽象队列 B;
+ 取template里a得值:
    + 如果是直接取,则因为template里得a的值的改变操作还在队列里,所以得到值为1;
    + 如果使用nextTick,则把取值的操作压入一个队列,假设也为B,逻辑上晚于前面那个队列里的操作;
+ ...
+ 当前event-loop队列里的循环结束后,开始执行B队列里的操作;
    + 设置template里a的值为2;
    + 这时取template 里 a的值则为正确的2;



