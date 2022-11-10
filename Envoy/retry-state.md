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
## shouldRetryHeaders
通过上游`header`和下游`header`判断是否需要`retry`
```cpp
// source/common/router/retry_state_impl.cc
RetryStatus RetryStateImpl::shouldRetryHeaders(const Http::ResponseHeaderMap& response_headers,
                                               const Http::RequestHeaderMap& original_request,
                                               DoRetryHeaderCallback callback) {
  // This may be overridden in wouldRetryFromHeaders().
  bool disable_early_data = !conn_pool_new_stream_with_early_data_and_http3_;

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
        if (original_request.get(Http::Headers::get().EarlyData).empty()) {
          // Retry if the downstream request wasn't received as early data. Otherwise, regardless if
          // the request was sent as early data in upstream or not, don't retry. Instead, forward
          // the response to downstream.
          disable_early_data = true;
          return RetryDecision::RetryImmediately;
        }
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

  // The request has exhausted the number of retries allotted to it by the retry policy configured
  // (or the x-envoy-max-retries header).
  if (retries_remaining_ == 0) {
    cluster_.stats().upstream_rq_retry_limit_exceeded_.inc();
    if (vcluster_) {
      vcluster_->stats().upstream_rq_retry_limit_exceeded_.inc();
    }
    if (route_stats_context_.has_value()) {
      route_stats_context_->stats().upstream_rq_retry_limit_exceeded_.inc();
    }
    return RetryStatus::NoRetryLimitExceeded;
  }

  retries_remaining_--;

  if (!cluster_.resourceManager(priority_).retries().canCreate()) {
    cluster_.stats().upstream_rq_retry_overflow_.inc();
    if (vcluster_) {
      vcluster_->stats().upstream_rq_retry_overflow_.inc();
    }
    if (route_stats_context_.has_value()) {
      route_stats_context_->stats().upstream_rq_retry_overflow_.inc();
    }
    return RetryStatus::NoOverflow;
  }

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