[TOC]
# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
outlier_detection 用来配置envoy 的 cluster里的host的检测机制,配置在什么情况下,cluster里的某个host应该被认为不健康,下次有请求时不转给它
>> 注: istio 的 DestinationRule的Subset里的outlier配置,最终也是变成了envoy里的cluster outlier配置


# 接口
被监控的上游cluster立的host都有一个对应的`DetectorHostMonitorImpl`实例
```cpp
// source/common/upstream/outlier_detection_impl.h
class DetectorHostMonitorImpl : public DetectorHostMonitor {
public:
  // ...
  // 在请求处理的各个环节里触发信息收集,并根据已有信息决定要不要逐出host todo 哪里恢复
  void putHttpResponseCode(uint64_t response_code) override;
  void putResult(Result result, absl::optional<uint64_t> code) override;
  void putResponseTime(std::chrono::milliseconds) override {}
  // ...
}
```

## putHttpResponseCode 触发的地方
### onUpstreamHeaders 收到上游回复头时触发
```cpp
// source/common/router/router.cc
void Filter::onUpstreamHeaders(uint64_t response_code, Http::ResponseHeaderMapPtr&& headers,
                               UpstreamRequest& upstream_request, bool end_stream) {
  // ...
  // 如果请求是grpc时需要先把‘grpc-status’转换成对应的‘http-status’再调用‘putHttpResponseCode’方法更新host的统计信息
  // 这里的grpc-status 并不包括‘onUpstreamTrailers’传来的grpc-status,
  // 如果‘onUpstreamTrailers’的阶段的grpc-status报错,是不会触发‘putHttpResponseCode’方法的
  absl::optional<Grpc::Status::GrpcStatus> grpc_status;
  uint64_t grpc_to_http_status = 0;
  if (grpc_request_) {
    grpc_status = Grpc::Common::getGrpcStatus(*headers);
    if (grpc_status.has_value()) {
      grpc_to_http_status = Grpc::Utility::grpcToHttpStatus(grpc_status.value());
    }
  }

  if (grpc_status.has_value()) {
    upstream_request.upstreamHost()->outlierDetector().putHttpResponseCode(grpc_to_http_status);
  } else {
    upstream_request.upstreamHost()->outlierDetector().putHttpResponseCode(response_code);
  }
  // ...

  // 在retry前就处理了host统计更新,所以每一次retry都会更新一次
  if (retry_state_) {
    // ...
  }
  // ...
                               }
```
## putResult 触发的地方
### UpstreamRequest 连接初始化好时
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::onPoolReady(
    std::unique_ptr<GenericUpstream>&& upstream, Upstream::HostDescriptionConstSharedPtr host,
    const Network::Address::InstanceConstSharedPtr& upstream_local_address,
    StreamInfo::StreamInfo& info, absl::optional<Http::Protocol> protocol) {
  // ...
  host->outlierDetector().putResult(Upstream::Outlier::Result::LocalOriginConnectSuccess);
  // ...
}
```

### router filter在各种本地报错时触发putResult方法
#### 封装
`router filter` 对`putResult`的封装,只是验证了一下host是否存在,因为有些错误与host无关
```cpp
// source/common/router/router.cc
void Filter::updateOutlierDetection(Upstream::Outlier::Result result,
                                    UpstreamRequest& upstream_request,
                                    absl::optional<uint64_t> code) {
  if (upstream_request.upstreamHost()) {
    upstream_request.upstreamHost()->outlierDetector().putResult(result, code);
  }
}
```
#### onResponseTimeout
在发往上游的请求超时时触发
```cpp
// source/common/router/router.cc
void Filter::onResponseTimeout() {
  
  while (!upstream_requests_.empty()) {
    // ...
    // header都还没接收时才需要记录,如果已经接收header了那已经触发了‘putHttpResponseCode’方法记录信息了
    if (upstream_request->awaitingHeaders()) {
      // ...
      if (!upstream_request->outlierDetectionTimeoutRecorded()) {
        updateOutlierDetection(Upstream::Outlier::Result::LocalOriginTimeout, *upstream_request,
                               absl::optional<uint64_t>(enumToInt(timeout_response_code_)));
      }

      // ...
    }
    upstream_request->resetStream();
  }
  // ...
}
```

#### onPerTryTimeoutCommon
在单次发往上游的请求失败时触发
```cpp
// source/common/router/router.cc
void Filter::onPerTryTimeoutCommon(UpstreamRequest& upstream_request, Stats::Counter& error_counter,
                                   const std::string& response_code_details) {
  // ...
  // 记录本地超时事件
  updateOutlierDetection(Upstream::Outlier::Result::LocalOriginTimeout, upstream_request,
                         absl::optional<uint64_t>(enumToInt(timeout_response_code_)));

  // ...
}
```
#### onUpstreamReset
当连接因为各种原因被重置时触发
```cpp
// source/common/router/router.cc
void Filter::onUpstreamReset(Http::StreamResetReason reset_reason,
                             absl::string_view transport_failure_reason,
                             UpstreamRequest& upstream_request) {

  // 记录本地连接错误事件
  updateOutlierDetection(Upstream::Outlier::Result::LocalOriginConnectFailed, upstream_request,
                         absl::nullopt);

  // ...
}
```

### 其他extension 如`dubbo_proxy`,`redis_proxy`等也会触发`putResult`方法更新统计信息,暂不深入

## putResponseTime 使用的地方
### 在完整收到上游回复时触发
```cpp
void Filter::onUpstreamComplete(UpstreamRequest& upstream_request) {
  // ...
  if (config_.emit_dynamic_stats_ && !callbacks_->streamInfo().healthCheck() &&
      DateUtil::timePointValid(downstream_request_complete_time_)) {
    upstream_request.upstreamHost()->outlierDetector().putResponseTime(response_time);
    // ...
  }
  // ...
}

```

# 核心代码逻辑
## HostMonitor
### putHttpResponseCode
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorHostMonitorImpl::putHttpResponseCode(uint64_t response_code) {
  external_origin_sr_monitor_.incTotalReqCounter();
  if (Http::CodeUtility::is5xx(response_code)) {
    std::shared_ptr<DetectorImpl> detector = detector_.lock();
    // 判断连续是否出发了连续GatewayError
    if (Http::CodeUtility::isGatewayError(response_code)) {
      if (++consecutive_gateway_failure_ ==
          detector->runtime().snapshot().getInteger(
              ConsecutiveGatewayFailureRuntime, detector->config().consecutiveGatewayFailure())) {
        detector->onConsecutiveGatewayFailure(host_.lock());
      }
    } else { // 有一次成功就清0
      consecutive_gateway_failure_ = 0;
    }

    // 判断是否触发了配置的连续5xx错误逐出
    if (++consecutive_5xx_ == detector->runtime().snapshot().getInteger(
                                  Consecutive5xxRuntime, detector->config().consecutive5xx())) {
      detector->onConsecutive5xx(host_.lock());
    }
  } else { // 不是5xx就代表‘连续’已经断了
    external_origin_sr_monitor_.incSuccessReqCounter();
    consecutive_5xx_ = 0;
    consecutive_gateway_failure_ = 0;
  }
}
```
### putResult
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorHostMonitorImpl::putResult(Result result, absl::optional<uint64_t> code) {
  // put_result_func_ 根据是否配置了‘splitExternalLocalOriginErrors’选择调用不同的方法
  put_result_func_(this, result, code);
}
```
#### putResultWithLocalExternalSplit
设置了本地事件与上游回复处理分离时
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorHostMonitorImpl::putResultWithLocalExternalSplit(Result result,
                                                              absl::optional<uint64_t>) {
  switch (result) {
  case Result::LocalOriginConnectSuccess:
  case Result::LocalOriginConnectSuccessFinal:
    // 走本地事件处理逻辑
    return localOriginNoFailure();
  case Result::LocalOriginTimeout:
  case Result::LocalOriginConnectFailed:
    // 走本地事件处理逻辑
    return localOriginFailure();
  
  // 特殊情况虽然时本地事件,但从情境上来说可以认为是上游回复503 todo 什么情况??
  case Result::ExtOriginRequestFailed:
    return putHttpResponseCode(enumToInt(Http::Code::ServiceUnavailable));
  
  case Result::ExtOriginRequestSuccess:
    putHttpResponseCode(enumToInt(Http::Code::OK));
    localOriginNoFailure();
    break;
  }
}
```
#### putResultNoLocalExternalSplit
设置了本地事件与上游回复处理不分离时
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorHostMonitorImpl::putResultNoLocalExternalSplit(Result result,
                                                            absl::optional<uint64_t> code) {
  if (code) {
    putHttpResponseCode(code.value());
  } else {
    // 将事件转换成http_code
    // Result::LocalOriginTimeout => Http::Code::GatewayTimeout(504)
    // Result::LocalOriginConnectFailed =>  Http::Code::ServiceUnavailable(503)

    absl::optional<Http::Code> http_code = resultToHttpCode(result);
    if (http_code) {
      putHttpResponseCode(enumToInt(http_code.value()));
    }
  }
}
```

## detector
### create创建一个detector
- 输入
  - cluster 目标集群
  - config 配置
  - dispatcher todo
- 输出
  - 初始化好的detector
```cpp
// source/common/upstream/outlier_detection_impl.cc
td::shared_ptr<DetectorImpl> DetectorImpl::create(
    const Cluster& cluster, const envoy::config::cluster::v3::OutlierDetection& config,
    Event::Dispatcher& dispatcher, Runtime::Loader& runtime, TimeSource& time_source,
    EventLoggerSharedPtr event_logger, Random::RandomGenerator& random) {
  // 创建‘DetectorImpl’实例并初始化
  std::shared_ptr<DetectorImpl> detector(
      new DetectorImpl(cluster, config, dispatcher, runtime, time_source, event_logger, random));
  // ...
  detector->initialize(cluster);

  return detector;
}

void DetectorImpl::initialize(const Cluster& cluster) {
  // 遍历cluster的所有优先级,所有host,将每个host纳入本地监控中
  for (auto& host_set : cluster.prioritySet().hostSetsPerPriority()) {
    for (const HostSharedPtr& host : host_set->hosts()) {
      // todo
      addHostMonitor(host);
    }
  }
  member_update_cb_ = cluster.prioritySet().addMemberUpdateCb( // 监听由cluster本身产生的host更新事件 todo
      [this](const HostVector& hosts_added, const HostVector& hosts_removed) -> void {
        for (const HostSharedPtr& host : hosts_added) {
          addHostMonitor(host);
        }

        for (const HostSharedPtr& host : hosts_removed) {
          ASSERT(host_monitors_.count(host) == 1);
          if (host->healthFlagGet(Host::HealthFlag::FAILED_OUTLIER_CHECK)) {
            ASSERT(ejections_active_helper_.value() > 0);
            ejections_active_helper_.dec();
          }

          host_monitors_.erase(host);
        }
      });
  // todo
  armIntervalTimer();
}
```

### onConsecutive5xx 触发一个host的‘连续5xx’异常报告
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorImpl::onConsecutive5xx(HostSharedPtr host) {
  notifyMainThreadConsecutiveError(host, envoy::data::cluster::v3::CONSECUTIVE_5XX);
}
// 这一层主要是因为detector被各个线程共享,需要机制只让主线程做最终的host异常处理(暂不深入)
// 最终调用了‘onConsecutiveErrorWorker’方法
void DetectorImpl::notifyMainThreadConsecutiveError(
    HostSharedPtr host, envoy::data::cluster::v3::OutlierEjectionType type) {
      // ...
  std::weak_ptr<DetectorImpl> weak_this = shared_from_this();
  dispatcher_.post([weak_this, host, type]() -> void {
    std::shared_ptr<DetectorImpl> shared_this = weak_this.lock();
    if (shared_this) {
      shared_this->onConsecutiveErrorWorker(host, type);
    }
  });
}

void DetectorImpl::onConsecutiveErrorWorker(HostSharedPtr host,
                                            envoy::data::cluster::v3::OutlierEjectionType type) {
  // ...
  // 逐出host #ref ejectHost
  ejectHost(host, type);
  // ...
}
```
#### ejectHost 将一个host暂时驱逐出cluster
cluster触发需要标记一个host不可用时调用 ejctHost方法,传入要驱逐的host和触发的驱逐规则类型
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorImpl::ejectHost(HostSharedPtr host,
                             envoy::data::cluster::v3::OutlierEjectionType type) {
  // 获取设置的最大逐出比例
  uint64_t max_ejection_percent = std::min<uint64_t>(
      100, runtime_.snapshot().getInteger(MaxEjectionPercentRuntime, config_.maxEjectionPercent()));
  double ejected_percent = 100.0 * ejections_active_helper_.value() / host_monitors_.size();
  // 如果配置了最大可驱逐host的比例,只有当前已驱逐的host的比例低于此值才会真正触发驱逐
  if (ejected_percent < max_ejection_percent) {
    // ...
    if (enforceEjection(type)) {
      // ...
      // 调用当前host列表里的host_monitor触发eject方法,驱逐该host todo #ref eject
      host_monitors_[host]->eject(time_source_.monotonicTime());
      // ...

      // 调用注册的回调 todo #ref runCallbacks
      runCallbacks(host);
      // ...
    } else {
      // ...
    }
  } else {
    // ...
  }
}
```