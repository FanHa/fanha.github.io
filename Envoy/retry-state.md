# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
`RetryStateImpl`记录一个`routerFilter`的重试情况,`routerFilter`根据上游返回的结果,超时等情况调用`RetryStateImpl`的方法决定要不要重试

# interface
```cpp
// source/common/router/retry_state_impl.h
class RetryStateImpl : public RetryState {
public:
  static std::unique_ptr<RetryStateImpl>
  create(const RetryPolicy& route_policy, Http::RequestHeaderMap& request_headers,
         const Upstream::ClusterInfo& cluster, const VirtualCluster* vcluster,
         RouteStatsContextOptRef route_stats_context, Runtime::Loader& runtime,
         Random::RandomGenerator& random, Event::Dispatcher& dispatcher, TimeSource& time_source,
         Upstream::ResourcePriority priority);
  ~RetryStateImpl() override;

  /**
   * Returns the RetryPolicy extracted from the x-envoy-retry-on header.
   * @param config is the value of the header.
   * @return std::pair<uint32_t, bool> the uint32_t is a bitset representing the
   *         valid retry policies in @param config. The bool is TRUE iff all the
   *         policies specified in @param config are valid.
   */
  static std::pair<uint32_t, bool> parseRetryOn(absl::string_view config);

  /**
   * Returns the RetryPolicy extracted from the x-envoy-retry-grpc-on header.
   * @param config is the value of the header.
   * @return std::pair<uint32_t, bool> the uint32_t is a bitset representing the
   *         valid retry policies in @param config. The bool is TRUE iff all the
   *         policies specified in @param config are valid.
   */
  static std::pair<uint32_t, bool> parseRetryGrpcOn(absl::string_view retry_grpc_on_header);

  // Router::RetryState
  bool enabled() override { return retry_on_ != 0; }
  absl::optional<std::chrono::milliseconds>
  parseResetInterval(const Http::ResponseHeaderMap& response_headers) const override;

  // routerFilter收到上游返回的header时,根据上游header和下游header判断是否需要retry
  RetryStatus shouldRetryHeaders(const Http::ResponseHeaderMap& response_headers,
                                 const Http::RequestHeaderMap& original_request,
                                 DoRetryHeaderCallback callback) override;
  // Returns if the retry policy would retry the passed headers and how. Does not
  // take into account circuit breaking or remaining tries.
  RetryDecision wouldRetryFromHeaders(const Http::ResponseHeaderMap& response_headers,
                                      const Http::RequestHeaderMap& original_request,
                                      bool& disable_early_data) override;
  RetryStatus shouldRetryReset(Http::StreamResetReason reset_reason, Http3Used http3_used,
                               DoRetryResetCallback callback) override;
  RetryStatus shouldHedgeRetryPerTryTimeout(DoRetryCallback callback) override;

  void onHostAttempted(Upstream::HostDescriptionConstSharedPtr host) override {
    std::for_each(retry_host_predicates_.begin(), retry_host_predicates_.end(),
                  [&host](auto predicate) { predicate->onHostAttempted(host); });
    if (retry_priority_) {
      retry_priority_->onHostAttempted(host);
    }
  }

  bool shouldSelectAnotherHost(const Upstream::Host& host) override {
    return std::any_of(
        retry_host_predicates_.begin(), retry_host_predicates_.end(),
        [&host](auto predicate) { return predicate->shouldSelectAnotherHost(host); });
  }

  const Upstream::HealthyAndDegradedLoad& priorityLoadForRetry(
      const Upstream::PrioritySet& priority_set,
      const Upstream::HealthyAndDegradedLoad& original_priority_load,
      const Upstream::RetryPriority::PriorityMappingFunc& priority_mapping_func) override {
    if (!retry_priority_) {
      return original_priority_load;
    }
    return retry_priority_->determinePriorityLoad(priority_set, original_priority_load,
                                                  priority_mapping_func);
  }

  uint32_t hostSelectionMaxAttempts() const override { return host_selection_max_attempts_; }

  bool isAutomaticallyConfiguredForHttp3() const { return auto_configured_for_http3_; }

private:
  // 创建RetryState,在routerFilter的decodeHeader阶段创建
  RetryStateImpl(const RetryPolicy& route_policy, Http::RequestHeaderMap& request_headers,
                 const Upstream::ClusterInfo& cluster, const VirtualCluster* vcluster,
                 RouteStatsContextOptRef route_stats_context, Runtime::Loader& runtime,
                 Random::RandomGenerator& random, Event::Dispatcher& dispatcher,
                 TimeSource& time_source, Upstream::ResourcePriority priority,
                 bool auto_configured_for_http3,
                 bool conn_pool_new_stream_with_early_data_and_http3);

  void enableBackoffTimer();
  void resetRetry();
  // Returns if the retry policy would retry the reset and how. Does not
  // take into account circuit breaking or remaining tries.
  // disable_http3: populated to tell the caller whether to disable http3 or not when the return
  // value indicates retry.
  RetryDecision wouldRetryFromReset(const Http::StreamResetReason reset_reason,
                                    Http3Used http3_used, bool& disable_http3);
  RetryStatus shouldRetry(RetryDecision would_retry, DoRetryCallback callback);

  const Upstream::ClusterInfo& cluster_;
  const VirtualCluster* vcluster_;
  RouteStatsContextOptRef route_stats_context_;
  Runtime::Loader& runtime_;
  Random::RandomGenerator& random_;
  Event::Dispatcher& dispatcher_;
  TimeSource& time_source_;
  uint32_t retry_on_{};
  uint32_t retries_remaining_{};
  DoRetryCallback backoff_callback_;
  Event::SchedulableCallbackPtr next_loop_callback_;
  Event::TimerPtr retry_timer_;
  Upstream::ResourcePriority priority_;
  BackOffStrategyPtr backoff_strategy_;
  BackOffStrategyPtr ratelimited_backoff_strategy_{};
  std::vector<Upstream::RetryHostPredicateSharedPtr> retry_host_predicates_;
  Upstream::RetryPrioritySharedPtr retry_priority_;
  uint32_t host_selection_max_attempts_;
  std::vector<uint32_t> retriable_status_codes_;
  std::vector<Http::HeaderMatcherSharedPtr> retriable_headers_;
  std::vector<ResetHeaderParserSharedPtr> reset_headers_{};
  std::chrono::milliseconds reset_max_interval_{};
  const bool auto_configured_for_http3_{};
  const bool conn_pool_new_stream_with_early_data_and_http3_{};
};
```

# 实现
## create 
// source/common/router/retry_state_impl.cc
```cpp
std::unique_ptr<RetryStateImpl>
RetryStateImpl::create(const RetryPolicy& route_policy, Http::RequestHeaderMap& request_headers,
                       const Upstream::ClusterInfo& cluster, const VirtualCluster* vcluster,
                       RouteStatsContextOptRef route_stats_context, Runtime::Loader& runtime,
                       Random::RandomGenerator& random, Event::Dispatcher& dispatcher,
                       TimeSource& time_source, Upstream::ResourcePriority priority) {
  std::unique_ptr<RetryStateImpl> ret;

  const bool conn_pool_new_stream_with_early_data_and_http3 =
      Runtime::runtimeFeatureEnabled(Runtime::conn_pool_new_stream_with_early_data_and_http3);
  // 只有在下游请求header中带有retry相关的信息或本地配置的route信息中有retry信息时才创建‘RetryStateImpl’实例
  if (request_headers.EnvoyRetryOn() || request_headers.EnvoyRetryGrpcOn() ||
      route_policy.retryOn()) {
    ret.reset(new RetryStateImpl(route_policy, request_headers, cluster, vcluster,
                                 route_stats_context, runtime, random, dispatcher, time_source,
                                 priority, false, conn_pool_new_stream_with_early_data_and_http3));
  }
  // ...

  // 移除掉下游请求头中的retry相关,不要把这些头继续发给上游了
  request_headers.removeEnvoyRetryOn();
  request_headers.removeEnvoyRetryGrpcOn();
  request_headers.removeEnvoyMaxRetries();
  request_headers.removeEnvoyHedgeOnPerTryTimeout();
  request_headers.removeEnvoyRetriableHeaderNames();
  request_headers.removeEnvoyRetriableStatusCodes();
  request_headers.removeEnvoyUpstreamRequestPerTryTimeoutMs();

  return ret;
}

```

### RetryStateImpl
```cpp
RetryStateImpl::RetryStateImpl(const RetryPolicy& route_policy,
                               Http::RequestHeaderMap& request_headers,
                               const Upstream::ClusterInfo& cluster, const VirtualCluster* vcluster,
                               RouteStatsContextOptRef route_stats_context,
                               Runtime::Loader& runtime, Random::RandomGenerator& random,
                               Event::Dispatcher& dispatcher, TimeSource& time_source,
                               Upstream::ResourcePriority priority, bool auto_configured_for_http3,
                               bool conn_pool_new_stream_with_early_data_and_http3)
    : cluster_(cluster), vcluster_(vcluster), route_stats_context_(route_stats_context),
      runtime_(runtime), random_(random), dispatcher_(dispatcher), time_source_(time_source),
      retry_on_(route_policy.retryOn()), retries_remaining_(route_policy.numRetries()),
      priority_(priority), retry_host_predicates_(route_policy.retryHostPredicates()),
      retry_priority_(route_policy.retryPriority()),
      retriable_status_codes_(route_policy.retriableStatusCodes()),
      retriable_headers_(route_policy.retriableHeaders()),
      reset_headers_(route_policy.resetHeaders()),
      reset_max_interval_(route_policy.resetMaxInterval()),
      auto_configured_for_http3_(auto_configured_for_http3),
      conn_pool_new_stream_with_early_data_and_http3_(
          conn_pool_new_stream_with_early_data_and_http3) {
  // ...
  // 设置retry的基础间隔时间(base_interval)
  std::chrono::milliseconds base_interval(
      runtime_.snapshot().getInteger("upstream.base_retry_backoff_ms", 25));
  if (route_policy.baseInterval()) {
    base_interval = *route_policy.baseInterval();
  }

  // 设置retry的最大间隔时间(max_interval)
  std::chrono::milliseconds max_interval = base_interval * 10;
  if (route_policy.maxInterval()) {
    max_interval = *route_policy.maxInterval();
  }

  // retry 间隔时间生成器
  backoff_strategy_ = std::make_unique<JitteredExponentialBackOffStrategy>(
      base_interval.count(), max_interval.count(), random_);
  // 一个host最多尝试多少次(host_selection_retry_max_attempts)
  host_selection_max_attempts_ = route_policy.hostSelectionMaxAttempts();

  // 合并配置的retryOn 与 通过下游头得到的retryOn
  if (request_headers.EnvoyRetryOn()) {
    retry_on_ |= parseRetryOn(request_headers.getEnvoyRetryOnValue()).first;
  }
  if (request_headers.EnvoyRetryGrpcOn()) {
    retry_on_ |= parseRetryGrpcOn(request_headers.getEnvoyRetryGrpcOnValue()).first;
  }

  // route 策略里设置 retriable_request_headers 时,只有在下游请求头中带了这些header之一才会设置retry_on,不然就把所有retry_on设置清0
  const auto& retriable_request_headers = route_policy.retriableRequestHeaders();
  if (!retriable_request_headers.empty()) {
    // If this route limits retries by request headers, make sure there is a match.
    bool request_header_match = false;
    for (const auto& retriable_header : retriable_request_headers) {
      if (retriable_header->matchesHeaders(request_headers)) {
        request_header_match = true;
        break;
      }
    }

    if (!request_header_match) {
      retry_on_ = 0;
    }
  }
  // 下游请求头带有envoy-max-retries头时,用这个值覆盖原本的本地设置
  if (retry_on_ != 0 && request_headers.EnvoyMaxRetries()) {
    uint64_t temp;
    if (absl::SimpleAtoi(request_headers.getEnvoyMaxRetriesValue(), &temp)) {
      // The max retries header takes precedence if set.
      retries_remaining_ = temp;
    }
  }

  // 下游请求头带有envoy-retriable-status-codes时,将这些值合并进本地的设置里
  if (request_headers.EnvoyRetriableStatusCodes()) {
    for (const auto& code :
         StringUtil::splitToken(request_headers.getEnvoyRetriableStatusCodesValue(), ",")) {
      unsigned int out;
      if (absl::SimpleAtoi(code, &out)) {
        retriable_status_codes_.emplace_back(out);
      }
    }
  }

  // 下游请求头带有 ‘x-envoy-retriable-header-names’ 时,将这些头合并入本地配置的 ‘retriable_headers_’
  if (request_headers.EnvoyRetriableHeaderNames()) {
    for (const auto& header_name : StringUtil::splitToken(
             request_headers.EnvoyRetriableHeaderNames()->value().getStringView(), ",")) {
      envoy::config::route::v3::HeaderMatcher header_matcher;
      header_matcher.set_name(std::string(absl::StripAsciiWhitespace(header_name)));
      retriable_headers_.emplace_back(
          std::make_shared<Http::HeaderUtility::HeaderData>(header_matcher));
    }
  }
}
```

## shouldRetryHeaders
通过上游`header`和下游`header`判断是否需要`retry`
```cpp
// source/common/router/retry_state_impl.cc
RetryStatus RetryStateImpl::shouldRetryHeaders(const Http::ResponseHeaderMap& response_headers,
                                               const Http::RequestHeaderMap& original_request,
                                               DoRetryHeaderCallback callback) {
                                                // ...
  // 先判断从请求和返回头来看是否需要retry #ref wouldRetryFromHeaders
  const RetryDecision retry_decision =
      wouldRetryFromHeaders(response_headers, original_request, disable_early_data);

  // 返回结果是‘RetryWithBackoff’时需要设置一个‘retry’间隔
  if (retry_decision == RetryDecision::RetryWithBackoff && !reset_headers_.empty()) {
    const auto backoff_interval = parseResetInterval(response_headers);
    if (backoff_interval.has_value() && (backoff_interval.value().count() > 1L)) {
      ratelimited_backoff_strategy_ = std::make_unique<JitteredLowerBoundBackOffStrategy>(
          backoff_interval.value().count(), random_);
    }
  }
  // 得到了根据header信息判断的‘retry_decision’,还需要调用shouldRetry todo #ref shouldRetry
  return shouldRetry(retry_decision,
                     [disable_early_data, callback]() { callback(disable_early_data); });
}

RetryState::RetryDecision
RetryStateImpl::wouldRetryFromHeaders(const Http::ResponseHeaderMap& response_headers,
                                      const Http::RequestHeaderMap& original_request,
                                      bool& disable_early_data) {
  // A response that contains the x-envoy-ratelimited header comes from an upstream envoy.
  // We retry these only when the envoy-ratelimited policy is in effect.
  // 上游带x-envoy-ratelimited 头时,根据本地配置决定是返回 'RetryWithBackoff' 还是 ‘NoRetry’
  if (response_headers.EnvoyRateLimited() != nullptr) {
    return (retry_on_ & RetryPolicy::RETRY_ON_ENVOY_RATE_LIMITED) ? RetryDecision::RetryWithBackoff
                                                                  : RetryDecision::NoRetry;
  }
  // 获取返回的http status
  uint64_t response_status = Http::Utility::getResponseStatus(response_headers);

  // 判断是否配置了5xx retry
  if (retry_on_ & RetryPolicy::RETRY_ON_5XX) {
    if (Http::CodeUtility::is5xx(response_status)) {
      return RetryDecision::RetryWithBackoff;
    }
  }

  // 判断是否配置了‘gateway error’(502,503,504) retry
  if (retry_on_ & RetryPolicy::RETRY_ON_GATEWAY_ERROR) {
    if (Http::CodeUtility::isGatewayError(response_status)) {
      return RetryDecision::RetryWithBackoff;
    }
  }

  // 判断是否配置了 ‘retriable_4xx' retry
  if ((retry_on_ & RetryPolicy::RETRY_ON_RETRIABLE_4XX)) {
    Http::Code code = static_cast<Http::Code>(response_status);
    // 只有409才会retry
    if (code == Http::Code::Conflict) {
      return RetryDecision::RetryWithBackoff;
    }
  }

  // 判断是否设置了特定的'retriable_status'
  if ((retry_on_ & RetryPolicy::RETRY_ON_RETRIABLE_STATUS_CODES)) {
    for (auto code : retriable_status_codes_) {
      if (response_status == code) {
        if (!conn_pool_new_stream_with_early_data_and_http3_ ||
            static_cast<Http::Code>(code) != Http::Code::TooEarly) {
          return RetryDecision::RetryWithBackoff;
        }
      // ...
      }
    }
  }

  // 判断是否配置了需要retry 的头
  if (retry_on_ & RetryPolicy::RETRY_ON_RETRIABLE_HEADERS) {
    for (const auto& retriable_header : retriable_headers_) {
      if (retriable_header->matchesHeaders(response_headers)) {
        return RetryDecision::RetryWithBackoff;
      }
    }
  }

  // 判断是否配置了grpc返回码retry
  if (retry_on_ &
      (RetryPolicy::RETRY_ON_GRPC_CANCELLED | RetryPolicy::RETRY_ON_GRPC_DEADLINE_EXCEEDED |
       RetryPolicy::RETRY_ON_GRPC_RESOURCE_EXHAUSTED | RetryPolicy::RETRY_ON_GRPC_UNAVAILABLE |
       RetryPolicy::RETRY_ON_GRPC_INTERNAL)) {
    absl::optional<Grpc::Status::GrpcStatus> status = Grpc::Common::getGrpcStatus(response_headers);
    if (status) {
      if ((status.value() == Grpc::Status::Canceled &&
           (retry_on_ & RetryPolicy::RETRY_ON_GRPC_CANCELLED)) ||
          (status.value() == Grpc::Status::DeadlineExceeded &&
           (retry_on_ & RetryPolicy::RETRY_ON_GRPC_DEADLINE_EXCEEDED)) ||
          (status.value() == Grpc::Status::ResourceExhausted &&
           (retry_on_ & RetryPolicy::RETRY_ON_GRPC_RESOURCE_EXHAUSTED)) ||
          (status.value() == Grpc::Status::Unavailable &&
           (retry_on_ & RetryPolicy::RETRY_ON_GRPC_UNAVAILABLE)) ||
          (status.value() == Grpc::Status::Internal &&
           (retry_on_ & RetryPolicy::RETRY_ON_GRPC_INTERNAL))) {
        return RetryDecision::RetryWithBackoff;
      }
    }
  }
  // 什么都不满足
  return RetryDecision::NoRetry;
}
```
### shouldRetry
根据`response` header信息得到`RetryDecision`后,还需要再调用`shouldRetry`方法决定最终要不要retry
```cpp
RetryStatus RetryStateImpl::shouldRetry(RetryDecision would_retry, DoRetryCallback callback) {
  // If a callback is armed from a previous shouldRetry and we don't need to
  // retry this particular request, we can infer that we did a retry earlier
  // and it was successful.
  if ((backoff_callback_ || next_loop_callback_) && would_retry == RetryDecision::NoRetry) {
    cluster_.stats().upstream_rq_retry_success_.inc();
    if (vcluster_) {
      vcluster_->stats().upstream_rq_retry_success_.inc();
    }
    if (route_stats_context_.has_value()) {
      route_stats_context_->stats().upstream_rq_retry_success_.inc();
    }
  }

  resetRetry();

  if (would_retry == RetryDecision::NoRetry) {
    return RetryStatus::No;
  }

  // 判断设置的retry数是否已经到了
  if (retries_remaining_ == 0) {
    // ...
    return RetryStatus::NoRetryLimitExceeded;
  }

  retries_remaining_--;

  // 判断cluster的资源管理器是否允许再重试了
  if (!cluster_.resourceManager(priority_).retries().canCreate()) {
    // ...
    return RetryStatus::NoOverflow;
  }

  // ?? todo
  if (!runtime_.snapshot().featureEnabled("upstream.use_retry", 100)) {
    return RetryStatus::No;
  }

  ASSERT(!backoff_callback_ && !next_loop_callback_);
  cluster_.resourceManager(priority_).retries().inc();
  cluster_.stats().upstream_rq_retry_.inc();
  if (vcluster_) {
    vcluster_->stats().upstream_rq_retry_.inc();
  }
  if (route_stats_context_.has_value()) {
    route_stats_context_->stats().upstream_rq_retry_.inc();
  }
  if (would_retry == RetryDecision::RetryWithBackoff) {
    backoff_callback_ = callback;
    enableBackoffTimer();
  } else {
    next_loop_callback_ = dispatcher_.createSchedulableCallback(callback);
    next_loop_callback_->scheduleCallbackNextIteration();
  }
  return RetryStatus::Yes;
}
```