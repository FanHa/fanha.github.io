### PromiseConstructor
```tq
// src/builtins/promise-constructor.tq
namespace promise {
  transitioning javascript builtin
  PromiseConstructor(
      js-implicit context: NativeContext, receiver: JSAny,
      newTarget: JSAny)(executor: JSAny): JSAny {
    // 1. If NewTarget is undefined, throw a TypeError exception.
    if (newTarget == Undefined) {
      ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
    }

    // 2. If IsCallable(executor) is false, throw a TypeError exception.
    if (!TaggedIsCallable(executor)) {
      ThrowTypeError(MessageTemplate::kResolverNotAFunction, executor);
    }

    const promiseFun = UnsafeCast<JSFunction>(
        context[NativeContextSlot::PROMISE_FUNCTION_INDEX]);

    // Silently fail if the stack looks fishy.
    if (HasAccessCheckFailed(context, promiseFun, executor)) {
      IncrementUseCounter(
          context, SmiConstant(kPromiseConstructorReturnedUndefined));
      return Undefined;
    }

    let result: JSPromise;
    if (promiseFun == newTarget) {
      result = NewJSPromise();
    } else {
      result = UnsafeCast<JSPromise>(EmitFastNewObject(
          context, promiseFun, UnsafeCast<JSReceiver>(newTarget)));
      PromiseInit(result);
      if (IsPromiseHookEnabledOrHasAsyncEventDelegate()) {
        runtime::PromiseHookInit(result, Undefined);
      }
    }

    const isDebugActive = IsDebugActive();
    if (isDebugActive) runtime::DebugPushPromise(result);

    const funcs = CreatePromiseResolvingFunctions(result, True, context);
    const resolve = funcs.resolve;
    const reject = funcs.reject;
    try {
      Call(context, UnsafeCast<Callable>(executor), Undefined, resolve, reject);
    } catch (e) {
      Call(context, reject, Undefined, e);
    }

    if (isDebugActive) runtime::DebugPopPromise();
    return result;
  }
}
```

### 参数
```tq
transitioning javascript builtin
  PromiseConstructor(
      js-implicit context: NativeContext, receiver: JSAny,
      newTarget: JSAny)(executor: JSAny): JSAny {}
```
+   context,recerver,newTarget 是js运行过程中的一些隐藏变量;  
    + context 运行环境
    + recerver 相当于运行时的this
    + newTarget 对应new.target,判断当前是不是由关键字new触发的
+   executor是传入Promise的参数

```js
// Promise 里的function就是executor
const promise1 = new Promise(function(resolve, reject) {
    // ...
})
```

### 防御性检测
```tq
    // 检测Promise是否由"new"原语触发
    if (newTarget == Undefined) {
      ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
    }

    // 检测传入的参数是不是function
    if (!TaggedIsCallable(executor)) {
      ThrowTypeError(MessageTemplate::kResolverNotAFunction, executor);
    }

    // 从环境中取出Promise的运行模版(JSFunction)
    const promiseFun = UnsafeCast<JSFunction>(
        context[NativeContextSlot::PROMISE_FUNCTION_INDEX]);

    // 一些其他防御性检测
    if (HasAccessCheckFailed(context, promiseFun, executor)) {
      IncrementUseCounter(
          context, SmiConstant(kPromiseConstructorReturnedUndefined));
      return Undefined;
    }
```

### 链式与非链式调用Promise的初始化
```tq
    let result: JSPromise;
    if (promiseFun == newTarget) {
      // 非链式或者是链式的第一层
      result = NewJSPromise();
    } else {
      // 链式
      result = UnsafeCast<JSPromise>(EmitFastNewObject(
          context, promiseFun, UnsafeCast<JSReceiver>(newTarget)));
      PromiseInit(result);
      if (IsPromiseHookEnabledOrHasAsyncEventDelegate()) {
        runtime::PromiseHookInit(result, Undefined);
      }
    }
```

#### 非链式的Promise初始化
```tq
// src/builtins/promise-misc.tq
@export
  transitioning macro NewJSPromise(implicit context: Context)(): JSPromise {
    return NewJSPromise(Undefined);
  }
// ...

 @export
  transitioning macro NewJSPromise(implicit context: Context)(parent: Object):
      JSPromise {
    // 创建一个JSPromise实例
    const instance = InnerNewJSPromise();
    // 初始化JSPromise实例,设置一些Promise运行过程中的状态为初始化状态
    PromiseInit(instance);

    if (IsPromiseHookEnabledOrHasAsyncEventDelegate()) {
      // 运行Promise的Init Hook函数
      runtime::PromiseHookInit(instance, parent);
    }
    return instance;
  }
```

### 初始化resolve 和 reject函数
初始化resolve和reject函数的环境等信息
```tq
    const funcs = CreatePromiseResolvingFunctions(result, True, context);
    const resolve = funcs.resolve;
    const reject = funcs.reject;
```

### 执行executor

```tq
  try {
      // 先按正常执行传入的参数executor里的内容,这里resolve,reject也作为参数传进了调用里,因为executor里可以手动调用resolve 和 reject 函数
      Call(context, UnsafeCast<Callable>(executor), Undefined, resolve, reject);
    } catch (e) {
      // 捕获并执行reject 函数
      Call(context, reject, Undefined, e);
    }
```

