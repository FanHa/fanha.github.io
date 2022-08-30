# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
local_ratelimit 是envoy官方提供的本地限流filter,不需要外部依赖,所有配置都加载在envoy自身

# 核心代码逻辑
## decodeHeaders 收到下游请求时的处理
```cpp
// source/extensions/filters/http/local_ratelimit/local_ratelimit.cc
Http::FilterHeadersStatus Filter::decodeHeaders(Http::RequestHeaderMap& headers, bool) {
  const auto* config = getConfig();
  // ...

  // filter通过descriptor的配置 更精细的限流(descriptor描述了更精准的 请求来源 -》 请求目的地的限流信息)
  // 这里先找出当前请求是否满足某个配置好的descriptor
  std::vector<RateLimit::LocalDescriptor> descriptors;
  if (config->hasDescriptors()) {
    populateDescriptors(descriptors, headers);
  }

  // 保存descriptor信息,后面触发限流有设置时需要返回给客户端,告诉客户端发生了什么
  stored_descriptors_ = descriptors;

  // #ref requestAllowd 根据 descriptor决定请求要不要放行
  if (requestAllowed(descriptors)) {
    // 放行直接回Continue
    return Http::FilterHeadersStatus::Continue;
  }

  // 虽然根据规则触发了限流,但插件仍然可以通过配置enforced参数放行请求(此举应该是为了测试与观察)
  if (!config->enforced()) {
    config->requestHeadersParser().evaluateHeaders(headers, decoder_callbacks_->streamInfo());
    return Http::FilterHeadersStatus::Continue;
  }

  // 代码走到这里就是确实不能放行了,直接返回给下游,表明已被限流
  decoder_callbacks_->sendLocalReply(
      config->status(), "local_rate_limited",
      [this, config](Http::HeaderMap& headers) {
        config->responseHeadersParser().evaluateHeaders(headers, decoder_callbacks_->streamInfo());
      },
      absl::nullopt, "local_rate_limited");
  decoder_callbacks_->streamInfo().setResponseFlag(StreamInfo::ResponseFlag::RateLimited);

  return Http::FilterHeadersStatus::StopIteration;
}
```
## encodeHeaders 收到上游回复时的处理
```cpp
// source/extensions/filters/http/local_ratelimit/local_ratelimit.cc
Http::FilterHeadersStatus Filter::encodeHeaders(Http::ResponseHeaderMap& headers, bool) {
  const auto* config = getConfig();
  // 根据配置对返回给下游的回复增加一些与限流相关的表头信息
  if (config->enabled() && config->enableXRateLimitHeaders()) {
    auto limit = maxTokens(stored_descriptors_.value());
    auto remaining = remainingTokens(stored_descriptors_.value());
    auto reset = remainingFillInterval(stored_descriptors_.value());

    headers.addReferenceKey(
        HttpFilters::Common::RateLimit::XRateLimitHeaders::get().XRateLimitLimit, limit);
    headers.addReferenceKey(
        HttpFilters::Common::RateLimit::XRateLimitHeaders::get().XRateLimitRemaining, remaining);
    headers.addReferenceKey(
        HttpFilters::Common::RateLimit::XRateLimitHeaders::get().XRateLimitReset, reset);
  }

  return Http::FilterHeadersStatus::Continue;
}
```
## requestAllowed 决定是否触发限流
通过比对限流配置,当前请求满足的descriptor,以及当前运行数据决定请求是否放行
```cpp
// source/extensions/filters/http/local_ratelimit/local_ratelimit.cc
bool Filter::requestAllowed(absl::Span<const RateLimit::LocalDescriptor> request_descriptors) {
  const auto* config = getConfig();
  // TODO 根据限流是 perconnection 来决定调用哪个限流器(PerConnectionRateLimiter/LocalRateLimiter)的requestAllowed 方法
  return config->rateLimitPerConnection()
             ? getPerConnectionRateLimiter().requestAllowed(request_descriptors)
             : config->requestAllowed(request_descriptors);
}
```

### LocalRateLimiter 本地限流器
#### 是否触发限流的判断
```cpp
// source/extensions/filters/common/local_ratelimit/local_ratelimit_impl.cc
bool LocalRateLimiterImpl::requestAllowed(
    absl::Span<const RateLimit::LocalDescriptor> request_descriptors) const {
  // 当配置了必须通过所有的descriptor的检测才能放行时
  // todo 这个 runtimeFeatureEnabled 从哪里来??

  if (Runtime::runtimeFeatureEnabled(
          "envoy.reloadable_features.local_ratelimit_match_all_descriptors")) {
    // 先查看是否已经全局限流(注:这个“全局限流”是相对于具体的descriptor的)
    // #ref requestAllowedHelper
    bool allow = requestAllowedHelper(tokens_);
    // 全局就已经触发限流了,不必再对比具体的descriptor了,直接返回false
    if (!allow) {
      return allow;
    }

    if (!descriptors_.empty() && !request_descriptors.empty()) {
      // todo 这个双循环会不会有性能影响
      for (const auto& descriptor : sorted_descriptors_) {
        for (const auto& request_descriptor : request_descriptors) {
          // 同一个请求可能会匹配到多个配置的descriptor,必须都同意放行才能真正放行
          if (descriptor == request_descriptor) {
            // 匹配到了当前请求的descriptor 与 配置的descriptor,根据配置的descriptor的当前状态信息决定放不放行
            // #ref requestAllowedHelper

            allow &= requestAllowedHelper(*descriptor.token_state_);
            if (!allow) {
              return allow;
            }
            break;
          }
        }
      }
    }
    
    return allow;
  }
  // 简单路径,只比对匹配到的第一个descriptor
  auto descriptor = descriptorHelper(request_descriptors);

  // 没有匹配到descriptor就调用全局token的数据,匹配到了就调用该descriptor的token数据决定放不放行
  return descriptor.has_value() ? requestAllowedHelper(*descriptor.value().get().token_state_)
                                : requestAllowedHelper(tokens_);
}

// 判断是否触发限流并减少剩余token计数
bool LocalRateLimiterImpl::requestAllowedHelper(const TokenState& tokens) const {
  // 原子操作 tokens
  uint32_t expected_tokens = tokens.tokens_.load(std::memory_order_relaxed);
  do {
    // token不够了,返回false表示要限流
    if (expected_tokens == 0) {
      return false;
    }

    // 锁,准备给token 减1时发现数据已被其他地方改过,需要循环调用,直到改成功为止
  } while (!tokens.tokens_.compare_exchange_weak(expected_tokens, expected_tokens - 1,
                                                 std::memory_order_relaxed));

  return true;
}
```

#### 周期性补充tokens
```cpp
// source/extensions/filters/common/local_ratelimit/local_ratelimit_impl.cc
LocalRateLimiterImpl::LocalRateLimiterImpl(
    const std::chrono::milliseconds fill_interval, const uint32_t max_tokens,
    const uint32_t tokens_per_fill, Event::Dispatcher& dispatcher,
    const Protobuf::RepeatedPtrField<
        envoy::extensions::common::ratelimit::v3::LocalRateLimitDescriptor>& descriptors)
    : fill_timer_(fill_interval > std::chrono::milliseconds(0)
                      ? dispatcher.createTimer([this] { onFillTimer(); })
                      : nullptr), // 设置fill_timer_ 的回调 onFillTimer todo #ref
      time_source_(dispatcher.timeSource()) {
  if (fill_timer_ && fill_interval < std::chrono::milliseconds(50)) {
    throw EnvoyException("local rate limit token bucket fill timer must be >= 50ms");
  }

  token_bucket_.max_tokens_ = max_tokens;
  token_bucket_.tokens_per_fill_ = tokens_per_fill;
  token_bucket_.fill_interval_ = absl::FromChrono(fill_interval);
  tokens_.tokens_ = max_tokens;
  tokens_.fill_time_ = time_source_.monotonicTime();

  if (fill_timer_) {
    fill_timer_->enableTimer(fill_interval); // 更新全局限流信息的timer, 注:这里的全局是相对descriptor
  }

  for (const auto& descriptor : descriptors) { // 每个descriptor创建一个独特的限流数据结构
    LocalDescriptorImpl new_descriptor;
    for (const auto& entry : descriptor.entries()) {
      new_descriptor.entries_.push_back({entry.key(), entry.value()});
    }
    RateLimit::TokenBucket per_descriptor_token_bucket;
    per_descriptor_token_bucket.fill_interval_ =
        absl::Milliseconds(PROTOBUF_GET_MS_OR_DEFAULT(descriptor.token_bucket(), fill_interval, 0));
    // descriptor 的补充限流token数的间隔必须是全局间隔的整数倍(以便统一使用fill_timer_的回调来处理,减少不必要的性能损耗)
    if (per_descriptor_token_bucket.fill_interval_ % token_bucket_.fill_interval_ !=
        absl::ZeroDuration()) {
      throw EnvoyException(
          "local rate descriptor limit is not a multiple of token bucket fill timer");
    }
    // 简化descriptor补充token的间隔为 每几次onFillTimer回调补充一次(multipiler_);
    // descriptor max_tokens 和 tokens_per_fill 与全局
    new_descriptor.multiplier_ =
        per_descriptor_token_bucket.fill_interval_ / token_bucket_.fill_interval_;
    per_descriptor_token_bucket.max_tokens_ = descriptor.token_bucket().max_tokens();
    per_descriptor_token_bucket.tokens_per_fill_ =
        PROTOBUF_GET_WRAPPED_OR_DEFAULT(descriptor.token_bucket(), tokens_per_fill, 1);
    new_descriptor.token_bucket_ = per_descriptor_token_bucket;

    auto token_state = std::make_shared<TokenState>();
    token_state->tokens_ = per_descriptor_token_bucket.max_tokens_;
    token_state->fill_time_ = time_source_.monotonicTime();
    new_descriptor.token_state_ = token_state;

    // descriptor去重
    auto result = descriptors_.emplace(new_descriptor);
    if (!result.second) {
      throw EnvoyException(absl::StrCat("duplicate descriptor in the local rate descriptor: ",
                                        result.first->toString()));
    }
    // 保存descriptor便于后面排序
    sorted_descriptors_.push_back(new_descriptor);
  }
  // descriptor排序
  if (!sorted_descriptors_.empty()) {
    std::sort(sorted_descriptors_.begin(), sorted_descriptors_.end(),
              [this](LocalDescriptorImpl a, LocalDescriptorImpl b) -> bool {
                const int a_token_fill_per_second = tokensFillPerSecond(a);
                const int b_token_fill_per_second = tokensFillPerSecond(b);
                return a_token_fill_per_second < b_token_fill_per_second;
              });
  }
}

// 根据配置补充tokens
void LocalRateLimiterImpl::onFillTimer() {
  // 补充全局tokens
  onFillTimerHelper(tokens_, token_bucket_);
  // 补充descriptor的tokens
  onFillTimerDescriptorHelper();
  refill_counter_++;
  // 设置下一次触发onFillTimer的时间
  fill_timer_->enableTimer(absl::ToChronoMilliseconds(token_bucket_.fill_interval_));
}
```

