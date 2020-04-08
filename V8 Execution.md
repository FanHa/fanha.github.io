### Interface
```cpp
// src/execution/execution.h
class Execution final : public AllStatic {
 public:
  // 与DEBUG相关??
  enum class MessageHandling { kReport, kKeepPending };

  // 两种执行目标常量,一种是执行一个具体的Callable, 另一种是执行一个miacrotaskQueue队列里所有的任务
  enum class Target { kCallable, kRunMicrotasks };

  // 这些应该都是执行Callable,只是不同入口有细微差别
  V8_EXPORT_PRIVATE V8_WARN_UNUSED_RESULT static MaybeHandle<Object> Call(
      Isolate* isolate, Handle<Object> callable, Handle<Object> receiver,
      int argc, Handle<Object> argv[]);

  V8_WARN_UNUSED_RESULT static MaybeHandle<Object> CallBuiltin(
      Isolate* isolate, Handle<JSFunction> builtin, Handle<Object> receiver,
      int argc, Handle<Object> argv[]);

  V8_WARN_UNUSED_RESULT static MaybeHandle<Object> New(
      Isolate* isolate, Handle<Object> constructor, int argc,
      Handle<Object> argv[]);
  V8_WARN_UNUSED_RESULT static MaybeHandle<Object> New(
      Isolate* isolate, Handle<Object> constructor, Handle<Object> new_target,
      int argc, Handle<Object> argv[]);

  V8_EXPORT_PRIVATE static MaybeHandle<Object> TryCall(
      Isolate* isolate, Handle<Object> callable, Handle<Object> receiver,
      int argc, Handle<Object> argv[], MessageHandling message_handling,
      MaybeHandle<Object>* exception_out, bool reschedule_terminate = true);

  // 执行microtask队列
  static MaybeHandle<Object> TryRunMicrotasks(
      Isolate* isolate, MicrotaskQueue* microtask_queue,
      MaybeHandle<Object>* exception_out);

  // 执行webAssembly
  V8_EXPORT_PRIVATE static void CallWasm(Isolate* isolate,
                                         Handle<Code> wrapper_code,
                                         Address wasm_call_target,
                                         Handle<Object> object_ref,
                                         Address packed_args);
};

}  // namespace internal
}  // namespace v8

#endif  // V8_EXECUTION_EXECUTION_H_
```

### Implement
Execution里的各种调用最终都是return 一个Invoke(...),而Invoke里都通过InvokeParams::SetUpForXXX根据要调用的不同类型创建了一个InvokeParams实例
```cpp
// src/execution/execution.cc
MaybeHandle<Object> Execution::Call(Isolate* isolate, Handle<Object> callable,
                                    Handle<Object> receiver, int argc,
                                    Handle<Object> argv[]) {
  return Invoke(isolate, InvokeParams::SetUpForCall(isolate, callable, receiver,
                                                    argc, argv));
}

MaybeHandle<Object> Execution::CallBuiltin(Isolate* isolate,
                                           Handle<JSFunction> builtin,
                                           Handle<Object> receiver, int argc,
                                           Handle<Object> argv[]) {
  DCHECK(builtin->code().is_builtin());
  DisableBreak no_break(isolate->debug());
  return Invoke(isolate, InvokeParams::SetUpForCall(isolate, builtin, receiver,
                                                    argc, argv));
}

// static
MaybeHandle<Object> Execution::New(Isolate* isolate, Handle<Object> constructor,
                                   int argc, Handle<Object> argv[]) {
  return New(isolate, constructor, constructor, argc, argv);
}

// static
MaybeHandle<Object> Execution::New(Isolate* isolate, Handle<Object> constructor,
                                   Handle<Object> new_target, int argc,
                                   Handle<Object> argv[]) {
  return Invoke(isolate, InvokeParams::SetUpForNew(isolate, constructor,
                                                   new_target, argc, argv));
}
// static
MaybeHandle<Object> Execution::TryCall(
    Isolate* isolate, Handle<Object> callable, Handle<Object> receiver,
    int argc, Handle<Object> argv[], MessageHandling message_handling,
    MaybeHandle<Object>* exception_out, bool reschedule_terminate) {
  return InvokeWithTryCatch(
      isolate, InvokeParams::SetUpForTryCall(
                   isolate, callable, receiver, argc, argv, message_handling,
                   exception_out, reschedule_terminate));
}

// static
MaybeHandle<Object> Execution::TryRunMicrotasks(
    Isolate* isolate, MicrotaskQueue* microtask_queue,
    MaybeHandle<Object>* exception_out) {
  return InvokeWithTryCatch(
      isolate, InvokeParams::SetUpForRunMicrotasks(isolate, microtask_queue,
                                                   exception_out));
}
```

#### InvokeParams
```cpp
// src/execution/execution.cc
// 各种SetUpForXXXX 都是新建一个InvokeParams实例,包装要执行的信息,给外面的Invoke函数使用
InvokeParams InvokeParams::SetUpForNew(Isolate* isolate,
                                       Handle<Object> constructor,
                                       Handle<Object> new_target, int argc,
                                       Handle<Object>* argv) {
  InvokeParams params;
  params.target = constructor;
  params.receiver = isolate->factory()->undefined_value();
  params.argc = argc;
  params.argv = argv;
  params.new_target = new_target;
  params.microtask_queue = nullptr;
  params.message_handling = Execution::MessageHandling::kReport;
  params.exception_out = nullptr;
  params.is_construct = true;
  params.execution_target = Execution::Target::kCallable;
  params.reschedule_terminate = true;
  return params;
}

// static
InvokeParams InvokeParams::SetUpForCall(Isolate* isolate,
                                        Handle<Object> callable,
                                        Handle<Object> receiver, int argc,
                                        Handle<Object>* argv) {
  InvokeParams params;
  params.target = callable;
  params.receiver = NormalizeReceiver(isolate, receiver);
  params.argc = argc;
  params.argv = argv;
  params.new_target = isolate->factory()->undefined_value();
  params.microtask_queue = nullptr;
  params.message_handling = Execution::MessageHandling::kReport;
  params.exception_out = nullptr;
  params.is_construct = false;
  params.execution_target = Execution::Target::kCallable;
  params.reschedule_terminate = true;
  return params;
}

// static
InvokeParams InvokeParams::SetUpForTryCall(
    Isolate* isolate, Handle<Object> callable, Handle<Object> receiver,
    int argc, Handle<Object>* argv, Execution::MessageHandling message_handling,
    MaybeHandle<Object>* exception_out, bool reschedule_terminate) {
  InvokeParams params;
  params.target = callable;
  params.receiver = NormalizeReceiver(isolate, receiver);
  params.argc = argc;
  params.argv = argv;
  params.new_target = isolate->factory()->undefined_value();
  params.microtask_queue = nullptr;
  params.message_handling = message_handling;
  params.exception_out = exception_out;
  params.is_construct = false;
  params.execution_target = Execution::Target::kCallable;
  params.reschedule_terminate = reschedule_terminate;
  return params;
}

// static
InvokeParams InvokeParams::SetUpForRunMicrotasks(
    Isolate* isolate, MicrotaskQueue* microtask_queue,
    MaybeHandle<Object>* exception_out) {
  auto undefined = isolate->factory()->undefined_value();
  InvokeParams params;
  params.target = undefined;
  params.receiver = undefined;
  params.argc = 0;
  params.argv = nullptr;
  params.new_target = undefined;
  params.microtask_queue = microtask_queue;
  params.message_handling = Execution::MessageHandling::kReport;
  params.exception_out = exception_out;
  params.is_construct = false;
  params.execution_target = Execution::Target::kRunMicrotasks;
  params.reschedule_terminate = true;
  return params;
}
```

#### TryRunMicrotasks
```cpp
// src/execution/execution.cc
MaybeHandle<Object> Execution::TryRunMicrotasks(
    Isolate* isolate, MicrotaskQueue* microtask_queue,
    MaybeHandle<Object>* exception_out) {
  // 创建一个InvokeParams实例包裹要执行的microtask队列,调用InvokeWithTryCatch
  return InvokeWithTryCatch(
      isolate, InvokeParams::SetUpForRunMicrotasks(isolate, microtask_queue,
                                                   exception_out));
}

MaybeHandle<Object> InvokeWithTryCatch(Isolate* isolate,
                                       const InvokeParams& params) {
  bool is_termination = false;
  MaybeHandle<Object> maybe_result;
  if (params.exception_out != nullptr) {
    *params.exception_out = MaybeHandle<Object>();
  }
  
  {
    // 创建一个TryCatch实例用来实现JS语言层面的异常捕捉机制
    // Js的异常在JS看来是抛出了,在Cpp这边看来是给结果设置了一个值
    v8::TryCatch catcher(reinterpret_cast<v8::Isolate*>(isolate));
    catcher.SetVerbose(false);
    catcher.SetCaptureMessage(false);

    // 调用Invoke
    maybe_result = Invoke(isolate, params);

    // 检测到调用没有正常结束时的情况
    if (maybe_result.is_null()) {
      DCHECK(isolate->has_pending_exception());
      if (isolate->pending_exception() ==
          ReadOnlyRoots(isolate).termination_exception()) {
        is_termination = true;
      } else {
        if (params.exception_out != nullptr) {
          DCHECK(catcher.HasCaught());
          DCHECK(isolate->external_caught_exception());
          *params.exception_out = v8::Utils::OpenHandle(*catcher.Exception());
        }
        if (params.message_handling == Execution::MessageHandling::kReport) {
          isolate->OptionalRescheduleException(true);
        }
      }
    }
  }

  if (is_termination && params.reschedule_terminate) {
    // Reschedule terminate execution exception.
    isolate->OptionalRescheduleException(false);
  }

  return maybe_result;
}
```

##### Invoke(Microtask)
```cpp
// src/execution/execution.cc
V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,
                                                 const InvokeParams& params) {
  RuntimeCallTimerScope timer(isolate, RuntimeCallCounterId::kInvoke);

  if (params.target->IsJSFunction()) {
    // 前面条件设定Microtask的target为undefined了
  }

  // Entering JavaScript.
  VMState<JS> state(isolate);
  CHECK(AllowJavascriptExecution::IsAllowed(isolate));
  if (!ThrowOnJavascriptExecution::IsAllowed(isolate)) {
    isolate->ThrowIllegalOperation();
    if (params.message_handling == Execution::MessageHandling::kReport) {
      isolate->ReportPendingMessages();
    }
    return MaybeHandle<Object>();
  }
  if (!DumpOnJavascriptExecution::IsAllowed(isolate)) {
    V8::GetCurrentPlatform()->DumpWithoutCrashing();
    return isolate->factory()->undefined_value();
  }

  if (params.execution_target == Execution::Target::kCallable) {
    // Microtask 的param 的execution_target设定为
    //   params.execution_target = Execution::Target::kRunMicrotasks;

  }

  // Placeholder for return value.
  Object value;

  // 这里这个JSEntry返回可执行代码的地址入口;
  // 这里已经很接近底层汇编代码,根据不同的硬件生成了不同的汇编(伪汇编)代码指令,暂且放下不深入 
  // TODO
  Handle<Code> code =
      JSEntry(isolate, params.execution_target, params.is_construct);
  {
    // 一运行环境相关的设置
    SaveContext save(isolate);
    SealHandleScope shs(isolate);

    if (FLAG_clear_exceptions_on_js_entry) isolate->clear_pending_exception();

    if (params.execution_target == Execution::Target::kCallable) {
      // Microtask param的 这个属性是 Execution::Target::kRunMicrotasks, 掠过
    } else {
      DCHECK_EQ(Execution::Target::kRunMicrotasks, params.execution_target);

      // clang-format off
      // return value: tagged pointers
      // {microtask_queue}: pointer to a C++ object
      using JSEntryFunction = GeneratedCode<Address(
          Address root_register_value, MicrotaskQueue* microtask_queue)>;
      // 从上面的可执行代码的地址入口code 中取出 “instructionStart”,指令的开始地址
      JSEntryFunction stub_entry =
          JSEntryFunction::FromAddress(isolate, code->InstructionStart());

      RuntimeCallTimerScope timer(isolate, RuntimeCallCounterId::kJS_Execution);
      // 执行指令
      value = Object(stub_entry.Call(isolate->isolate_data()->isolate_root(),
                                     params.microtask_queue));
    }
  }
  // 异常检测,JS的异常检测,在cpp这里是在运行过程中已经捕获了,并在“全局”变量中设置了某个值,然后在这里去判断这个值决定js是不是发生了异常
  bool has_exception = value.IsException(isolate);
  DCHECK(has_exception == isolate->has_pending_exception());
  if (has_exception) {
    if (params.message_handling == Execution::MessageHandling::kReport) {
      isolate->ReportPendingMessages();
    }
    return MaybeHandle<Object>();
  } else {
    isolate->clear_pending_message();
  }

  return Handle<Object>(value, isolate);
}
```

#### TryCall
在RunMicrotask中正常情况下就是依次call每个执行任务
```cpp
// static
MaybeHandle<Object> Execution::TryCall(
    Isolate* isolate, Handle<Object> callable, Handle<Object> receiver,
    int argc, Handle<Object> argv[], MessageHandling message_handling,
    MaybeHandle<Object>* exception_out, bool reschedule_terminate) {
  // 和TryRunMicrotask一样是调用InvokeWithTryCatch,
  // 但由于参数InvokeParams的不同,执行了不同的代码分支
  return InvokeWithTryCatch(
      isolate, InvokeParams::SetUpForTryCall(
                   isolate, callable, receiver, argc, argv, message_handling,
                   exception_out, reschedule_terminate));
}

V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,
                                                 const InvokeParams& params) {
  RuntimeCallTimerScope timer(isolate, RuntimeCallCounterId::kInvoke);

  if (params.target->IsJSFunction()) {
    // 当调用的是target是JSFunction 的情况
    Handle<JSFunction> function = Handle<JSFunction>::cast(params.target);
    if ((!params.is_construct || function->IsConstructor()) && //当要调用的是constructor函数的情形
        function->shared().IsApiFunction() &&
        !function->shared().BreakAtEntry()) {
      // 环境设置
      SaveAndSwitchContext save(isolate, function->context());
      DCHECK(function->context().global_object().IsJSGlobalObject());

      Handle<Object> receiver = params.is_construct
                                    ? isolate->factory()->the_hole_value()
                                    : params.receiver;

      // 执行调用InvokeApiFunction执行function里的内容
      auto value = Builtins::InvokeApiFunction(
          isolate, params.is_construct, function, receiver, params.argc,
          params.argv, Handle<HeapObject>::cast(params.new_target));
      // cpp执行的异常已经在invoke里捕获了,js的异常通过这个设置的值来标明
      bool has_exception = value.is_null();
      DCHECK(has_exception == isolate->has_pending_exception());
      if (has_exception) {
        if (params.message_handling == Execution::MessageHandling::kReport) {
          isolate->ReportPendingMessages();
        }
        return MaybeHandle<Object>();
      } else {
        isolate->clear_pending_message();
      }
      // 在这里已经返回了
      return value;
    }

    // Set up a ScriptContext when running scripts that need it.
    if (function->shared().needs_script_context()) {
      Handle<Context> context;
      if (!NewScriptContext(isolate, function).ToHandle(&context)) {
        if (params.message_handling == Execution::MessageHandling::kReport) {
          isolate->ReportPendingMessages();
        }
        return MaybeHandle<Object>();
      }

      // We mutate the context if we allocate a script context. This is
      // guaranteed to only happen once in a native context since scripts will
      // always produce name clashes with themselves.
      function->set_context(*context);
    }
  }

  // Entering JavaScript.
  VMState<JS> state(isolate);
  CHECK(AllowJavascriptExecution::IsAllowed(isolate));
  if (!ThrowOnJavascriptExecution::IsAllowed(isolate)) {
    isolate->ThrowIllegalOperation();
    if (params.message_handling == Execution::MessageHandling::kReport) {
      isolate->ReportPendingMessages();
    }
    return MaybeHandle<Object>();
  }
  if (!DumpOnJavascriptExecution::IsAllowed(isolate)) {
    V8::GetCurrentPlatform()->DumpWithoutCrashing();
    return isolate->factory()->undefined_value();
  }

  if (params.execution_target == Execution::Target::kCallable) {
    Handle<Context> context = isolate->native_context();
    // ?? 存在callback ??
    if (!context->script_execution_callback().IsUndefined(isolate)) {
      v8::Context::AbortScriptExecutionCallback callback =
          v8::ToCData<v8::Context::AbortScriptExecutionCallback>(
              context->script_execution_callback());
      v8::Isolate* api_isolate = reinterpret_cast<v8::Isolate*>(isolate);
      v8::Local<v8::Context> api_context = v8::Utils::ToLocal(context);
      callback(api_isolate, api_context);
      DCHECK(!isolate->has_scheduled_exception());
      // Always throw an exception to abort execution, if callback exists.
      isolate->ThrowIllegalOperation();
      return MaybeHandle<Object>();
    }
  }

  // Placeholder for return value.
  Object value;

  Handle<Code> code =
      JSEntry(isolate, params.execution_target, params.is_construct);
  {
    // Save and restore context around invocation and block the
    // allocation of handles without explicit handle scopes.
    SaveContext save(isolate);
    SealHandleScope shs(isolate);

    if (FLAG_clear_exceptions_on_js_entry) isolate->clear_pending_exception();

    // 掉需要执行的目标代码是Callable类型时
    if (params.execution_target == Execution::Target::kCallable) {
      
      using JSEntryFunction = GeneratedCode<Address(
          Address root_register_value, Address new_target, Address target,
          Address receiver, intptr_t argc, Address** argv)>;
      // 取出可执行代码的地址入口
      JSEntryFunction stub_entry =
          JSEntryFunction::FromAddress(isolate, code->InstructionStart());

      Address orig_func = params.new_target->ptr();
      Address func = params.target->ptr();
      Address recv = params.receiver->ptr();
      Address** argv = reinterpret_cast<Address**>(params.argv);
      RuntimeCallTimerScope timer(isolate, RuntimeCallCounterId::kJS_Execution);
      // 执行并把结果赋值给value
      value = Object(stub_entry.Call(isolate->isolate_data()->isolate_root(),
                                     orig_func, func, recv, params.argc, argv));
    } else {
      // ...
  }

  // 异常处理
  bool has_exception = value.IsException(isolate);
  DCHECK(has_exception == isolate->has_pending_exception());
  if (has_exception) {
    if (params.message_handling == Execution::MessageHandling::kReport) {
      isolate->ReportPendingMessages();
    }
    return MaybeHandle<Object>();
  } else {
    isolate->clear_pending_message();
  }

  return Handle<Object>(value, isolate);
}
```

