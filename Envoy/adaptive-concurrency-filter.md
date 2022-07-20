## 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

## 序
adaptive-concurency-filter是envoy官方提供的自适应限流filter

## 核心代码逻辑
### Filter入口
```cpp
// source/extensions/filters/http/adaptive_concurrency/adaptive_concurrency_filter.cc
// 这里是filter的入口,envoy收到了对服务的请求的处理, TODO 确认一个这样的filter实例是不是就代表一个请求
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

  // 记录本次请求的 RTT值,便于后面
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
  // 统计信息更新
  stats_.min_rtt_msecs_.set(
      std::chrono::duration_cast<std::chrono::milliseconds>(min_rtt_).count());
  // !!更新限流值!! TODO deferred_limit_value_ 的来源
  updateConcurrencyLimit(deferred_limit_value_.load());
  // 本次采样算rtt结束了, 重新将 deferred_limit_value_ 设置为0
  deferred_limit_value_.store(0);
  stats_.min_rtt_calculation_active_.set(0);

  // TODO
  min_rtt_calc_timer_->enableTimer(
      applyJitter(config_.minRTTCalcInterval(), config_.jitterPercent()));
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