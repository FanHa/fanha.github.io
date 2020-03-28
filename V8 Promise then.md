### Then
```tq
// src/builtins/promise-then.tq
  transitioning javascript builtin
  PromisePrototypeThen(js-implicit context: NativeContext, receiver: JSAny)(
      onFulfilled: JSAny, onRejected: JSAny): JSAny {
    //then是Promise的方法,如果此时this不是Promise,抛出错误
    const promise = Cast<JSPromise>(receiver) otherwise ThrowTypeError(
        MessageTemplate::kIncompatibleMethodReceiver, 'Promise.prototype.then',
        receiver);

    // 新建一个PROMISE_FUNCTION(JSFunction)结构
    const promiseFun = UnsafeCast<JSFunction>(
        context[NativeContextSlot::PROMISE_FUNCTION_INDEX]);

    let resultPromiseOrCapability: JSPromise|PromiseCapability;
    let resultPromise: JSAny;
    try {
      // 判断promise链是否完整,完整就直接去初始化Promise;
      // 这里的Promise链是否完整应该指promise链是否定义了catch 或finally之类的方法把?
      if (IsPromiseSpeciesLookupChainIntact(context, promise.map)) {
        goto AllocateAndInit;
      }

      const constructor = SpeciesConstructor(promise, promiseFun);
      if (TaggedEqual(constructor, promiseFun)) {
        goto AllocateAndInit;
      } else {
        // 因为promise链不完整,需要创建一个PromiseCapability结构,包含了当前Promise的resolve 和 reject 信息与 当前promise信息的结构
        
        const promiseCapability = NewPromiseCapability(constructor, True);
        resultPromiseOrCapability = promiseCapability;
        resultPromise = promiseCapability.promise;
      }
    }
    label AllocateAndInit {
      // Then方法和PromiseConstructor一样,结果也是应该返回一个JSPromise结构,
      const resultJSPromise = NewJSPromise(promise);
      resultPromiseOrCapability = resultJSPromise;
      resultPromise = resultJSPromise;
    }

    // 初始化then方法传入的两个handler参数,resolve 和 reject
    const onFulfilled = TaggedIsCallable(onFulfilled) ?
        UnsafeCast<Callable>(onFulfilled) :
        Undefined;
    const onRejected = TaggedIsCallable(onRejected) ?
        UnsafeCast<Callable>(onRejected) :
        Undefined;

    // “运行”then的Promise内容,并返回
    // 这个“运行”不一定是真运行,只是把当前then(Promise)里的信息建了个任务派给实际做事的队列之类的,然后返回promise结构
    PerformPromiseThenImpl(
        promise, onFulfilled, onRejected, resultPromiseOrCapability);
    return resultPromise;
  }
```

#### PerformPromiseThenImpl
```tq
// src/builtins/promise-abstract-operations.tq
  @export
  transitioning macro PerformPromiseThenImpl(implicit context: Context)(
      promise: JSPromise, onFulfilled: Callable|Undefined,
      onRejected: Callable|Undefined,
      resultPromiseOrCapability: JSPromise|PromiseCapability|Undefined): void {
    if (promise.Status() == PromiseState::kPending) {
      // 当Promise的状态是pending时,并不执行promise里的内容,创建一个PromiseReaction,等待将来“某个事件”发生(state状态改变)时再触发,然后把这些信息返回
      const promiseReactions =
          UnsafeCast<(Zero | PromiseReaction)>(promise.reactions_or_result);
      const reaction = NewPromiseReaction(
          promiseReactions, resultPromiseOrCapability, onFulfilled, onRejected);
      promise.reactions_or_result = reaction;
    } else {
      // Promise 状态是fulfill或reject时,根绝状态的不同可以立即把promise的内容走执行流程了
      let map: Map;
      let handler: Callable|Undefined = Undefined;
      let handlerContext: Context;
      if (promise.Status() == PromiseState::kFulfilled) {
        map = PromiseFulfillReactionJobTaskMapConstant();
        handler = onFulfilled;
        handlerContext = ExtractHandlerContext(onFulfilled, onRejected);
      } else
        // deferred用来表明括号类的内容可以延迟到整个函数最后再去编译,毕竟Reject在实际运行时并不常执行
        deferred {
          assert(promise.Status() == PromiseState::kRejected);
          map = PromiseRejectReactionJobTaskMapConstant();
          handler = onRejected;
          handlerContext = ExtractHandlerContext(onRejected, onFulfilled);
          if (!promise.HasHandler()) {
            runtime::PromiseRevokeReject(promise);
          }
        }

      const reactionsOrResult = promise.reactions_or_result;
      // 建立一个microtask,放入引擎的微任务执行队列
      const microtask = NewPromiseReactionJobTask(
          map, handlerContext, reactionsOrResult, handler,
          resultPromiseOrCapability);
      EnqueueMicrotask(handlerContext, microtask);
    }
    promise.SetHasHandler();
  }
```