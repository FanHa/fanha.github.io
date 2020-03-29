### 单例
```cpp
// src/execution/microtask-queue.h
class V8_EXPORT_PRIVATE MicrotaskQueue final : public v8::MicrotaskQueue {
 public:
  // 初始化并设立默认的MicrotaskQueue实例
  static void SetUpDefaultMicrotaskQueue(Isolate* isolate);
  // ??
  static std::unique_ptr<MicrotaskQueue> New(Isolate* isolate);

  ~MicrotaskQueue();
  //...
  private:

  // 构造函数私有保证单例
  MicrotaskQueue();
}
```

#### 静态方法 SetUpDefaultMicrotaskQueue 与 New
```cpp
// src/execution/microtask-queue.cc
// 创建一个MicrotaskQueue实例,并设置为当前运行环境的默认queue
void MicrotaskQueue::SetUpDefaultMicrotaskQueue(Isolate* isolate) {
  DCHECK_NULL(isolate->default_microtask_queue());
  MicrotaskQueue* microtask_queue = new MicrotaskQueue;
  microtask_queue->next_ = microtask_queue;
  microtask_queue->prev_ = microtask_queue;
  isolate->set_default_microtask_queue(microtask_queue);
}

// 似乎V8的引擎支持一个运行环境有多个队列,并通过链表连起来
std::unique_ptr<MicrotaskQueue> MicrotaskQueue::New(Isolate* isolate) {
  DCHECK_NOT_NULL(isolate->default_microtask_queue());

  std::unique_ptr<MicrotaskQueue> microtask_queue(new MicrotaskQueue);

  // Insert the new instance to the next of last MicrotaskQueue instance.
  MicrotaskQueue* last = isolate->default_microtask_queue()->prev_;
  microtask_queue->next_ = last->next_;
  microtask_queue->prev_ = last;
  last->next_->prev_ = microtask_queue.get();
  last->next_ = microtask_queue.get();

  return microtask_queue;
}
```

### 任务入队列接口
```cpp
// src/execution/microtask-queue.h
class V8_EXPORT_PRIVATE MicrotaskQueue final : public v8::MicrotaskQueue {
 public:
  void EnqueueMicrotask(v8::Isolate* isolate,
                        v8::Local<Function> microtask) override;
  void EnqueueMicrotask(v8::Isolate* isolate, v8::MicrotaskCallback callback,
                        void* data) override;
  void EnqueueMicrotask(Microtask microtask);
}
```

```cpp
// src/execution/microtask-queue.cc
// 这个重载应该是普通的js函数调用的入口
void MicrotaskQueue::EnqueueMicrotask(v8::Isolate* v8_isolate,
                                      v8::Local<Function> function) {
  Isolate* isolate = reinterpret_cast<Isolate*>(v8_isolate);
  HandleScope scope(isolate);
  // 新建一个microtask
  Handle<CallableTask> microtask = isolate->factory()->NewCallableTask(
      Utils::OpenHandle(*function), isolate->native_context());
  EnqueueMicrotask(*microtask);
}

// 这个重载应该是已经进入microtask队列并执行后的回调的调用入口;
// 猜想比如某个函数是个microtask,但这个函数内部有个异步或者其他的“宏任务”的代码块,就需要走这个入口
void MicrotaskQueue::EnqueueMicrotask(v8::Isolate* v8_isolate,
                                      v8::MicrotaskCallback callback,
                                      void* data) {
  Isolate* isolate = reinterpret_cast<Isolate*>(v8_isolate);
  HandleScope scope(isolate);
  Handle<CallbackTask> microtask = isolate->factory()->NewCallbackTask(
      isolate->factory()->NewForeign(reinterpret_cast<Address>(callback)),
      isolate->factory()->NewForeign(reinterpret_cast<Address>(data)));
  EnqueueMicrotask(*microtask);
}

// 最终两个重载的入队列方法都是将microtask作为参数,执行任务入队列操作
void MicrotaskQueue::EnqueueMicrotask(Microtask microtask) {
  if (size_ == capacity_) {
    // 当当前size大于已有队列的容量时,扩容队列
    intptr_t new_capacity = std::max(kMinimumCapacity, capacity_ << 1);
    ResizeBuffer(new_capacity);
  }

  DCHECK_LT(size_, capacity_);
  // 平时更高级的语言的push,pop之类的队列操作最终还是会一层层解剖到朴实无华的数组+偏移结构
  ring_buffer_[(start_ + size_) % capacity_] = microtask.ptr();
  ++size_;
}
```

### RunMicrotasks
```cpp
// src/execution/microtask-queue.cc
int MicrotaskQueue::RunMicrotasks(Isolate* isolate) {
  // 如果当前microtaskQueue里并没有任务需要做,则直接调用Oncompleted 钩子
  if (!size()) {
    OnCompleted(isolate);
    return 0;
  }

  // 统计信息
  intptr_t base_count = finished_microtask_count_;

  HandleScope handle_scope(isolate);
  MaybeHandle<Object> maybe_exception;

  MaybeHandle<Object> maybe_result;

  int processed_microtask_count;
  { 
    // 设置一些信息表明当前microtaskQueue实例正在running
    SetIsRunningMicrotasks scope(&is_running_microtasks_);

    // ??这里官方的文档注释是说抑制当前Isolate 的microtask执行?觉得可能是在关键节点上加锁,防止某些临界条件,一个microtask执行过程中,另一个地方同样队macrotaskQueue做操作而产生错误
    v8::Isolate::SuppressMicrotaskExecutionScope suppress(
        reinterpret_cast<v8::Isolate*>(isolate));
    // ??这里官方的文档注释是与 “多线程”运行相关,需要保存一些线程环境信息
    // 猜想平时从各种资料上看到的js引擎单线程执行其实应该是不太准确的,应该是js引擎保证代码按es规则“顺序”执行,但并不一定就是完全放在一个线程里,有些js语法上的顺序串行其实可以分拆成并行并且不会影响最终结果,这里可能就是与这个有关的处理
    HandleScopeImplementer::EnteredContextRewindScope rewind_scope(
        isolate->handle_scope_implementer());
    TRACE_EVENT_BEGIN0("v8.execute", "RunMicrotasks");
    {
      TRACE_EVENT_CALL_STATS_SCOPED(isolate, "v8", "V8.RunMicrotasks");
      // 把自己(this)作为参数,交由Execution实例来执行isolate里的microtask
      // 这是个同步调用,必定要把当前miacroQueue里的事情执行完,或者发生异常
      maybe_result = Execution::TryRunMicrotasks(isolate, this,
                                                 &maybe_exception);
      //统计信息
      processed_microtask_count =
          static_cast<int>(finished_microtask_count_ - base_count);
    }
    TRACE_EVENT_END1("v8.execute", "RunMicrotasks", "microtask_count",
                     processed_microtask_count);
  }

  // If execution is terminating, clean up and propagate that to TryCatch scope.
  // 异常终止时的情况
  if (maybe_result.is_null() && maybe_exception.is_null()) {
    // 清空队列
    delete[] ring_buffer_;
    ring_buffer_ = nullptr;
    capacity_ = 0;
    size_ = 0;
    start_ = 0;
    DCHECK(isolate->has_scheduled_exception());
    // 设置异常信息及处理
    isolate->SetTerminationOnExternalTryCatch();
    OnCompleted(isolate);
    return -1;
  }
  DCHECK_EQ(0, size());
  // 调用OnComplete 的hook
  OnCompleted(isolate);

  return processed_microtask_count;
}
```
