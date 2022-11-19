# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
router filter 的decodeHeaders 在请求发送前会调用 FilterUtility:finalTimeout 方法计算一个timeout_值

# RouterFilter 中与timeout相关的代码
```cpp
// source/common/router/router.cc

  // 根据route配置, 下游请求头, routerfilter配置, 是否时grpc请求, 创建一个结构实例,保存各种超时设置,后面用
  timeout_ = FilterUtility::finalTimeout(*route_entry_, headers, !config_.suppress_envoy_headers_,
                                         grpc_request_, hedging_params_.hedge_on_per_try_timeout_,
                                         config_.respect_expected_rq_timeout_);

  // 如果收到了下游传过来的header ‘x-envoy-upstream-duration-ms’,消化并保存到本地
  // 在upstreamRequest实例的onPoolReady里会用到这个值
  const Http::HeaderEntry* header_max_stream_duration_entry =
      headers.EnvoyUpstreamStreamDurationMs();
  if (header_max_stream_duration_entry) {
    dynamic_max_stream_duration_ =
        FilterUtility::tryParseHeaderTimeout(*header_max_stream_duration_entry);
    // remove这个头,不要继续传递了
    headers.removeEnvoyUpstreamStreamDurationMs();
  }

  // 如果收到了下游传来的‘x-envoy-upstream-rq-timeout-alt-response‘,将timeout_response_code_ 设为204
  // todo 哪里用到了
  if (headers.EnvoyUpstreamRequestTimeoutAltResponse()) {
    timeout_response_code_ = Http::Code::NoContent;
    // remove这个头,不要继续传递了
    headers.removeEnvoyUpstreamRequestTimeoutAltResponse();
  }


```
# 关键接口
```cpp
// source/common/router/router.h
class FilterUtility {
  // ...
  // 根据route 配置 和 下游请求头计算timeout
  static TimeoutData finalTimeout(const RouteEntry& route, Http::RequestHeaderMap& request_headers,
                                  bool insert_envoy_expected_request_timeout_ms, bool grpc_request,
                                  bool per_try_timeout_hedging_enabled,
                                  bool respect_expected_rq_timeout);
}
```

# 代码实现
## FilterUtility::finalTimeout
根据route 配置 和 下游请求头计算timeout
```cpp
// source/common/router/router.cc
FilterUtility::TimeoutData
FilterUtility::finalTimeout(const RouteEntry& route, Http::RequestHeaderMap& request_headers,
                            bool insert_envoy_expected_request_timeout_ms, bool grpc_request,
                            bool per_try_timeout_hedging_enabled,
                            bool respect_expected_rq_timeout) {
  TimeoutData timeout;
  if (!route.usingNewTimeouts()) { // 判断是否设置了route的 max_stream_timeout{} 系列参数,没有设置才会走下面的逻辑
    if (grpc_request && route.maxGrpcTimeout()) { // 如果请求是grpc 且 route设置了max_grpc_timeout值
      const std::chrono::milliseconds max_grpc_timeout = route.maxGrpcTimeout().value();
      // 从下游请求头中取出 grpc-timeout值
      auto header_timeout = Grpc::Common::getGrpcTimeout(request_headers);
      std::chrono::milliseconds grpc_timeout =
          header_timeout ? header_timeout.value() : std::chrono::milliseconds(0);
      if (route.grpcTimeoutOffset()) { // 如果route设置了grpc_timeout_offset变量,则需要把grpc_timeout减去相应值
        const auto offset = *route.grpcTimeoutOffset();
        if (offset < grpc_timeout) {
          grpc_timeout -= offset;
        }
      }

      // 根据route设置的‘max_grpc_timeout’值调整grpc_timeout
      if (max_grpc_timeout != std::chrono::milliseconds(0) &&
          (grpc_timeout == std::chrono::milliseconds(0) || grpc_timeout > max_grpc_timeout)) {
        grpc_timeout = max_grpc_timeout;
      }
      // 设置请求的全局timeout值
      timeout.global_timeout_ = grpc_timeout;
    } else {
      // 不是grpc请求或没有设置max_grpc_timeout,使用route的 ‘timeout’设置(如果没有设置则为15000ms)
      timeout.global_timeout_ = route.timeout();
    }
  }
  // 根据route的 ‘retry_policy‘ 设置单次try请求的超时时间
  timeout.per_try_timeout_ = route.retryPolicy().perTryTimeout();
  timeout.per_try_idle_timeout_ = route.retryPolicy().perTryIdleTimeout();

  uint64_t header_timeout;

  // 如果‘route filter’ 设置了 ‘respect_expected_rq_timeout’ 为 true, 则需要考虑下游请求头中的‘x-envoy-expected-rq-timeout-ms’值,否则就忽略
  if (respect_expected_rq_timeout) {
    const Http::HeaderEntry* header_expected_timeout_entry =
        request_headers.EnvoyExpectedRequestTimeoutMs();
    if (header_expected_timeout_entry) {
      // 如果请求头中带了 x-envoy-expected-rq-timeout-ms ,则需要根据头中的值重新设置 timeout
      trySetGlobalTimeout(*header_expected_timeout_entry, timeout);
    } else {
      // 没有‘x-envoy-expected-rq-timeout-ms’头时,取‘x-envoy-upstream-rq-timeout-ms’的值
      const Http::HeaderEntry* header_timeout_entry =
          request_headers.EnvoyUpstreamRequestTimeoutMs();

      if (header_timeout_entry) {
        trySetGlobalTimeout(*header_timeout_entry, timeout);
        // ‘x-envoy-upstream-rq-timeout-ms’的值消费掉后需要remove,不再传递给下一跳
        request_headers.removeEnvoyUpstreamRequestTimeoutMs();
      }
    }
  } else {
    // router filter 没有设置‘respect_expected_rq_timeout’时,取‘x-envoy-expected-rq-timeout-ms’头的值设置全局timeout,并remove该头不再传递给下一跳
    const Http::HeaderEntry* header_timeout_entry = request_headers.EnvoyUpstreamRequestTimeoutMs();

    if (header_timeout_entry) {
      trySetGlobalTimeout(*header_timeout_entry, timeout);
      request_headers.removeEnvoyUpstreamRequestTimeoutMs();
    }
  }

  // 根据下游头‘x-upstream-rq-per-try-timeout-ms’ 设置per_try_timeout值
  const absl::string_view per_try_timeout_entry =
      request_headers.getEnvoyUpstreamRequestPerTryTimeoutMsValue();
  if (!per_try_timeout_entry.empty()) {
    if (absl::SimpleAtoi(per_try_timeout_entry, &header_timeout)) {
      timeout.per_try_timeout_ = std::chrono::milliseconds(header_timeout);
    }
    request_headers.removeEnvoyUpstreamRequestPerTryTimeoutMs();
  }

  // 当‘per_tyr_timeout’值大于‘global’值时,不需要设置了,清0就行
  if (timeout.per_try_timeout_ >= timeout.global_timeout_ && timeout.global_timeout_.count() != 0) {
    timeout.per_try_timeout_ = std::chrono::milliseconds(0);
  }

  // #ref setTimeoutHeaders
  setTimeoutHeaders(0, timeout, route, request_headers, insert_envoy_expected_request_timeout_ms,
                    grpc_request, per_try_timeout_hedging_enabled);

  return timeout;
}

```
### setTimeoutHeaders
```cpp
// source/common/router/router.cc
// elapse_time 是传进来表示这个请求已经过去了多久
// timeout 是这个请求的各种timeout设置
// todo
void FilterUtility::setTimeoutHeaders(uint64_t elapsed_time,
                                      const FilterUtility::TimeoutData& timeout,
                                      const RouteEntry& route,
                                      Http::RequestHeaderMap& request_headers,
                                      bool insert_envoy_expected_request_timeout_ms,
                                      bool grpc_request, bool per_try_timeout_hedging_enabled) {
                                      
  const uint64_t global_timeout = timeout.global_timeout_.count();
  
  // 根据设置计算出基本的exptected_timeout值,用来传递给上游envoy,让其更清楚的知道什么时候会超时
  uint64_t expected_timeout = timeout.per_try_timeout_.count();

  if (per_try_timeout_hedging_enabled || expected_timeout == 0) { // todo hedge
    expected_timeout = global_timeout;
  }
  // If the expected timeout is 0 set no timeout, as Envoy treats 0 as infinite timeout.
  if (expected_timeout > 0) {
    if (global_timeout > 0) {

      if (elapsed_time >= global_timeout) { 
        // 如果请求消耗的时间已经超过了全局timeout,则设置一个非常小的expected_timeout
        // ?? todo 怎么保证这个请求不被发出去??
        expected_timeout = 1;
      } else {
        expected_timeout = std::min(expected_timeout, global_timeout - elapsed_time);
      }
    }
    // router filter没有明确设置不要传x-envoy-xxxx头时,会在请求头里设置x-envoy-expected-rq-timeout-ms
    if (insert_envoy_expected_request_timeout_ms) {
      request_headers.setEnvoyExpectedRequestTimeoutMs(expected_timeout);
    }

    // 如果是grpc请求,且route没有设置新的max_stream_timeout属性,则将exptected_timeout值也设置到头grpc-timeout里
    if (grpc_request && !route.usingNewTimeouts() && route.maxGrpcTimeout()) {
      Grpc::Common::toGrpcTimeout(std::chrono::milliseconds(expected_timeout), request_headers);
    }
  }
```

# UpstreamRequest 中怎么使用timeout
## onPoolReady 准备实际发送给上游前
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::onPoolReady(
    std::unique_ptr<GenericUpstream>&& upstream, Upstream::HostDescriptionConstSharedPtr host,
    const Network::Address::InstanceConstSharedPtr& upstream_local_address,
    StreamInfo::StreamInfo& info, absl::optional<Http::Protocol> protocol) {
  // ...
  if (parent_.downstreamEndStream()) {
    // 如果下游的请求已经完全接收完毕了,则直接利用router已经设置好的值设置perTryTimeout
    // todo setupPerTryTimeout
    setupPerTryTimeout();
  } else {
    // 如果下游请求还没结束,则先设置个标记,等请求结束了再设置perTryTimeout
    // ?? todo 这样是不是其实会有时间差,向上游的请求已经开始发出去了然后才开始设置超时
    create_per_try_timeout_on_request_complete_ = true;
  }

  // ...

  absl::optional<std::chrono::milliseconds> max_stream_duration;
  if (parent_.dynamicMaxStreamDuration().has_value()) {
    // 如果router 收到了‘x-envoy-upstream-duration-ms’头,则取这个值
    max_stream_duration = parent_.dynamicMaxStreamDuration().value();
  } else if (upstream_host_->cluster().commonHttpProtocolOptions().has_max_stream_duration()) {
    // 已经depreciated 的兼容设置
    // ...
  }
  // 设置一个‘max_stream_duration’超时回调
  // todo 这个超时和前面的perTrytimeout回调有什么区别??
  if (max_stream_duration.has_value() && max_stream_duration->count()) {
    max_stream_duration_timer_ = parent_.callbacks()->dispatcher().createTimer(
        [this]() -> void { onStreamMaxDurationReached(); });
    max_stream_duration_timer_->enableTimer(*max_stream_duration);
  }
  // ...
}
```
### create_per_try_timeout_on_request_complete_ 的使用
在routerFilter感知到下游请求已经完全接收好后会调用`onRequestComplete`
```cpp
// source/common/router/router.cc
void Filter::onRequestComplete() {
  // ...
  downstream_end_stream_ = true;
  // ...

  // Possible that we got an immediate reset.
  if (!upstream_requests_.empty()) {
    // ...
    maybeDoShadowing();

    if (timeout_.global_timeout_.count() > 0) {
      response_timeout_ = dispatcher.createTimer([this]() -> void { onResponseTimeout(); });
      response_timeout_->enableTimer(timeout_.global_timeout_);
    }

    for (auto& upstream_request : upstream_requests_) {
      if (upstream_request->createPerTryTimeoutOnRequestComplete()) {
        upstream_request->setupPerTryTimeout();
      }
    }
  }
}
```