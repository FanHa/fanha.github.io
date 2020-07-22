# GC
触发GC时根据传入的type进入两种GC大分支`Minor`和`Full`;对应轻量级的部分的GC和整体全部的GC
```cpp
// src/extensions/gc-extension.cc
void InvokeGC(v8::Isolate* isolate, v8::Isolate::GarbageCollectionType type,
              v8::EmbedderHeapTracer::EmbedderStackState embedder_stack_state) {
  Heap* heap = reinterpret_cast<Isolate*>(isolate)->heap();
  switch (type) {
    case v8::Isolate::GarbageCollectionType::kMinorGarbageCollection:
      // Minor类型时直接调用CollectGarbage方法对 i::NEW_SPACE 进行GC
      heap->CollectGarbage(i::NEW_SPACE, i::GarbageCollectionReason::kTesting,
                           kGCCallbackFlagForced);
      break;
    case v8::Isolate::GarbageCollectionType::kFullGarbageCollection:
      heap->SetEmbedderStackStateForNextFinalizaton(embedder_stack_state);
      heap->PreciseCollectAllGarbage(i::Heap::kNoGCFlags,
                                     i::GarbageCollectionReason::kTesting,
                                     kGCCallbackFlagForced);
      break;
  }
}
```
`Full`类型的调用封装如下
```cpp
// src/heap/heap.cc
void Heap::PreciseCollectAllGarbage(int flags,
                                    GarbageCollectionReason gc_reason,
                                    const GCCallbackFlags gc_callback_flags) {
  if (!incremental_marking()->IsStopped()) {
    // GC的增量标记没有停止时强制停止增量标记,为后面的彻彻底底的GC让路
    FinalizeIncrementalMarkingAtomically(gc_reason);
  }
  CollectAllGarbage(flags, gc_reason, gc_callback_flags);
}

void Heap::CollectAllGarbage(int flags, GarbageCollectionReason gc_reason,
                             const v8::GCCallbackFlags gc_callback_flags) {
  // 最终调用CollectGarbage方法对 OLD_SPACE 进行GC
  CollectGarbage(OLD_SPACE, gc_reason, gc_callback_flags);

}
```

## CollectGarbage
```cpp
// src/heap/heap.cc
bool Heap::CollectGarbage(AllocationSpace space,
                          GarbageCollectionReason gc_reason,
                          const v8::GCCallbackFlags gc_callback_flags) {
  const char* collector_reason = nullptr;
  // 根据要进行GC的space类型决定GC的方法,目前GarbageCollector类型有SCAVENGER, MARK_COMPACTOR, MINOR_MARK_COMPACTOR
  GarbageCollector collector = SelectGarbageCollector(space, &collector_reason);
  is_current_gc_forced_ = gc_callback_flags & v8::kGCCallbackFlagForced;

  // ...

  // 开始GC前给整个语言执行程序设置一个state表示正在GC
  VMState<GC> state(isolate());
  // ...

  bool next_gc_likely_to_collect_more = false;
  size_t committed_memory_before = 0;

  if (collector == MARK_COMPACTOR) {
    // 记录执行GC前系统可以使用的内存量
    committed_memory_before = CommittedOldGenerationMemory();
  }

  {
    // 跟踪器开始运行,猜想是用来跟踪和调试GC信息
    tracer()->Start(collector, gc_reason, collector_reason);
    // ...
    // 开始GC的前置工作
    GarbageCollectionPrologue();

    {
      // ...
      // performGC
      next_gc_likely_to_collect_more =
          PerformGarbageCollection(collector, gc_callback_flags);
      // ...
    }

    is_current_gc_forced_ = false;
    // GC收尾工作
    GarbageCollectionEpilogue();


    if (collector == MARK_COMPACTOR) {
      // 记录GC后系统资源状况
      size_t committed_memory_after = CommittedOldGenerationMemory();
      size_t used_memory_after = OldGenerationSizeOfObjects();
      MemoryReducer::Event event;
      event.type = MemoryReducer::kMarkCompact;
      event.time_ms = MonotonicallyIncreasingTimeInMs();
      event.next_gc_likely_to_collect_more =
          (committed_memory_before > committed_memory_after + MB) ||
          HasHighFragmentation(used_memory_after, committed_memory_after) ||
          (detached_contexts().length() > 0);
      event.committed_memory = committed_memory_after;
      if (deserialization_complete_) {
        // TODO 这个deserialization_complete_是用于什么情况?
        // 将本次GC的信息记录通知memory_reducer调度准备下一次GC
        memory_reducer_->NotifyMarkCompact(event);
      }
      if (initial_max_old_generation_size_ < max_old_generation_size_ &&
          used_memory_after < initial_max_old_generation_size_threshold_) {
        max_old_generation_size_ = initial_max_old_generation_size_;
      }
    }
    // 跟踪器结束运行
    tracer()->Stop(collector);
  }

  // ...

  // 如果当前的GC方法不是`FULL`类型,则可以保存此次GC的信息,便于下次增量GC时参考本次GC完成后的位置
  if (IsYoungGenerationCollector(collector)) {
    StartIncrementalMarkingIfAllocationLimitIsReached(
        GCFlagsForIncrementalMarking(),
        kGCCallbackScheduleIdleGarbageCollection);
  }

  return next_gc_likely_to_collect_more;
}
```
### PerformGarbageCollection
```cpp

```
### 附
```cpp
// src/common/globals.h
enum GarbageCollector { SCAVENGER, MARK_COMPACTOR, MINOR_MARK_COMPACTOR };
```