# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
adaptive-concurency-filter是envoy官方提供的自适应限流filter

# 核心代码逻辑
## 初始化Controller
初始化Controller时需要创建两个关键timer,一个`min_rtt_calc_timer_`用来周期性计算`理想RTT(min_rtt)`,一个`sample_reset_timer_`用来重制采样的数据
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
GradientController::GradientController(GradientControllerConfig config,
                                       Event::Dispatcher& dispatcher, Runtime::Loader&,
                                       const std::string& stats_prefix, Stats::Scope& scope,
                                       Random::RandomGenerator& random, TimeSource& time_source)
    : config_(std::move(config)), dispatcher_(dispatcher), scope_(scope),
      stats_(generateStats(scope_, stats_prefix)), random_(random), time_source_(time_source),
      deferred_limit_value_(0), num_rq_outstanding_(0),
      concurrency_limit_(config_.minConcurrency()),
      latency_sample_hist_(hist_fast_alloc(), hist_free) {
  // 触发min_rtt重新计算的timer,调用 ‘enterMinRTTSamplingWindow’ #ref enterMinRTTSamplingWindow
  min_rtt_calc_timer_ = dispatcher_.createTimer([this]() -> void { enterMinRTTSamplingWindow(); });
  // 根据真实请求数据的‘SampleRtt’ 与 ‘MinRtt’对比,设置新的 ‘ConcurrencyLimit’ 值
  sample_reset_timer_ = dispatcher_.createTimer([this]() -> void {
    // 正在MinRTT采样周期内,则不采样(采到的数据会失真,反应不了当前服务的真实RTT)
    if (inMinRTTSamplingWindow()) {
      return;
    }

    {
      absl::MutexLock ml(&sample_mutation_mtx_);
      // 根据采样信息算出sampleRtt,再根据‘SampleRtt’与‘MinRtt’ 的对比决定是否要增大或减小‘ConcurrencyLimit’ #ref resetSampleWindow
      resetSampleWindow();
    }

    sample_reset_timer_->enableTimer(config_.sampleRTTCalcInterval());
  });

  enterMinRTTSamplingWindow();
  sample_reset_timer_->enableTimer(config_.sampleRTTCalcInterval());
  stats_.concurrency_limit_.set(concurrency_limit_.load());
}
```
### enterMinRTTSamplingWindow
进入抽样计算`理想RTT(minRtt)`的窗口阶段,这里只是打开一个开关,实际仍是在每个请求的filter实例逻辑里根据设置的采样值决定放不放行,根据样本信息来更新MinRTT值
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
void GradientController::enterMinRTTSamplingWindow() {
  // ...
  absl::MutexLock ml(&sample_mutation_mtx_);

  // 将当前限流值保存到 ‘deferred_limit_value_’,便于取样结束后恢复
  deferred_limit_value_.store(GradientController::concurrencyLimit());
  // 将限流值设置为最小限流值
  updateConcurrencyLimit(config_.minConcurrency());

  // 更新取样计算min_rtt的窗口版本(只有在这个时间后收到的请求才有资格进入取样环节)
  min_rtt_epoch_ = time_source_.monotonicTime();
}
```

### resetSampleWindow
根据已有的真实请求`SampleRtt`与`MinRtt`之间的比较,设置新的`ConcurrencyLimit`值
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
void GradientController::resetSampleWindow() {
  // 得到真实请求的‘SampleRtt’
  sample_rtt_ = processLatencySamplesAndClear();
  // ...
  // calculateNewLimit 根据‘SampleRtt’ 与 ‘MinRtt’ 使用算法算出新的‘ConcurrencyLimit’值,并设置 #ref calculateNewLimit
  updateConcurrencyLimit(calculateNewLimit());
}
```

### calculateNewLimit TODO
```cpp
uint32_t GradientController::calculateNewLimit() {
  ASSERT(sample_rtt_.count() > 0);

  // Calculate the gradient value, ensuring it's clamped between 0.5 and 2.0.
  // This prevents extreme changes in the concurrency limit between each sample
  // window.
  const auto buffered_min_rtt = min_rtt_.count() + min_rtt_.count() * config_.minRTTBufferPercent();
  const double raw_gradient = static_cast<double>(buffered_min_rtt) / sample_rtt_.count();
  const double gradient = std::max<double>(0.5, std::min<double>(2.0, raw_gradient));
  stats_.gradient_.set(gradient);

  const double limit = concurrencyLimit() * gradient;
  const double burst_headroom = sqrt(limit);
  stats_.burst_queue_size_.set(burst_headroom);

  // The final concurrency value factors in the burst headroom and must be clamped to keep the value
  // in the range [configured_min, configured_max].
  const uint32_t new_limit = limit + burst_headroom;
  return std::max<uint32_t>(config_.minConcurrency(),
                            std::min<uint32_t>(config_.maxConcurrencyLimit(), new_limit));
}
```

## Filter逻辑
这里是filter的入口,envoy收到了对服务的请求的处理, TODO 确认一个这样的filter实例是不是就代表一个请求
```cpp
// source/extensions/filters/http/adaptive_concurrency/adaptive_concurrency_filter.cc
Http::FilterHeadersStatus AdaptiveConcurrencyFilter::decodeHeaders(Http::RequestHeaderMap&, bool) {
  // 当filter开关配置没有开启  或 当前是一个流调用的健康检查请求时,直接放行
  if (!config_->filterEnabled() || decoder_callbacks_->streamInfo().healthCheck()) {
    return Http::FilterHeadersStatus::Continue;
  }

  // 调用 “controller_->forwardingDecision()”方法判断该请求有没有触发限流;#ref forwardingDecision
  if (controller_->forwardingDecision() == Controller::RequestForwardingAction::Block) {
    // 触发限流直接返回限流错误码
    decoder_callbacks_->sendLocalReply(Http::Code::ServiceUnavailable, "reached concurrency limit",
                                       nullptr, absl::nullopt, "reached_concurrency_limit");
    return Http::FilterHeadersStatus::StopIteration;
  }

  // 记录请求的时间戳, 并设置当前请求得到回应时的回调 ‘ controller_->recordLatencySample(now)’,更新请求的各种统计信息;#ref recordLatencySample
  const auto now = config_->timeSource().monotonicTime();
  deferred_sample_task_ =
      std::make_unique<Cleanup>([this, now]() { controller_->recordLatencySample(now); });

  return Http::FilterHeadersStatus::Continue;
}
```

### forwardingDecision 判断请求是否触发了了限流
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
RequestForwardingAction GradientController::forwardingDecision() {
  // 判断还没有收到服务回应的请求‘num_rq_outstanding_’ 是不是超过了 “限流值”,没有超过则‘num_rq_outstanding_++’,并放行
  if (num_rq_outstanding_.load() < concurrencyLimit()) {
    ++num_rq_outstanding_;
    return RequestForwardingAction::Forward;
  }
  // 触发限流,更新统计信息,并返回‘Block’
  stats_.rq_blocked_.inc();
  return RequestForwardingAction::Block;
}
```

### recordLatencySample 收到从服务返回的请求后,统计时延等信息
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
void GradientController::recordLatencySample(MonotonicTime rq_send_time) {
  // 当前正在等待回应的请求-1
  --num_rq_outstanding_;

  //只统计属于当前采样窗口的请求
  if (rq_send_time < min_rtt_epoch_) {
    // Disregard samples from requests started in the previous minRTT window.
    return;
  }

  // 记录当前请求的响应时间间隔 ‘rq_latency’
  const std::chrono::microseconds rq_latency =
      std::chrono::duration_cast<std::chrono::microseconds>(time_source_.monotonicTime() -
                                                            rq_send_time);

  // 记录本次请求的 RTT值
  hist_insert(latency_sample_hist_.get(), rq_latency.count(), 1);

  // 更新用来做限流参考的MinRTT值,
  // 注: 方法内部会根据现在是不是处在取样窗口阶段而走不走实际更新MinRtt值的逻辑
  updateMinRTT();
}
```

### updateMinRTT 更新用来做限流参考的MinRTT值
```cpp
void GradientController::updateMinRTT() {
  // 只有在取样窗口周期 并且 已经超过设定的取样数时,才根据得到的取样rtt值们对minRTT做出更新
  if (!inMinRTTSamplingWindow() ||
      hist_sample_count(latency_sample_hist_.get()) < config_.minRTTAggregateRequestCount()) {
    return;
  }

  // 根据取样值和设置的percentile值得到 将要更新的min_rtt值 ?? TODO 这个值似乎只用来统计?
  min_rtt_ = processLatencySamplesAndClear();

  // 采样结束,将限流值恢复为采样前的值‘deferred_limit_value_’ 
  updateConcurrencyLimit(deferred_limit_value_.load());
  // 本次采样算rtt结束了, 重新将 deferred_limit_value_ 设置为0
  deferred_limit_value_.store(0);

  // 更新MinRTT值后需要根据配置设置‘min_rtt_calc_timer_’,过一阵子再次触发这个timer里的回调,执行取样逻辑
  min_rtt_calc_timer_->enableTimer(
      applyJitter(config_.minRTTCalcInterval(), config_.jitterPercent()));
  // 设置下一次触发真实请求采样回调的时间,‘sampleRTTCalcInterval’是通过配置设置的值
  sample_reset_timer_->enableTimer(config_.sampleRTTCalcInterval());
}
```

#### updateConcurrencyLimit 更新限流值
```cpp
// source/extensions/filters/http/adaptive_concurrency/controller/gradient_controller.cc
void GradientController::updateConcurrencyLimit(const uint32_t new_limit) {
  // ...
  // 根据参数设定limit值
  concurrency_limit_.store(new_limit);
  // 更新统计值
  stats_.concurrency_limit_.set(concurrency_limit_.load());
  // ...
}
```
