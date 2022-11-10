# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
router filter是filter chain的最后一个filter,用于将下游的请求最终转发给上游,将上游的回复返回给下游

# 关键interface
```cpp
// source/common/router/router.h
class Filter : Logger::Loggable<Logger::Id::router>,
               public Http::StreamDecoderFilter,
               public Upstream::LoadBalancerContextBase,
               public RouterFilterInterface {
public:
  // ...
  // 解码从下游发过来的请求(已经被其他filter处理过,最终来到了router-filter)的处理
  Http::FilterHeadersStatus decodeHeaders(Http::RequestHeaderMap& headers,
                                          bool end_stream) override;
  Http::FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) override;
  Http::FilterTrailersStatus decodeTrailers(Http::RequestTrailerMap& trailers) override;
  Http::FilterMetadataStatus decodeMetadata(Http::MetadataMap& metadata_map) override;
  void onRequestComplete();


  // 收到上游的回复时的处理
  void onUpstream1xxHeaders(Http::ResponseHeaderMapPtr&& headers,
                            UpstreamRequest& upstream_request) override;
  void onUpstreamHeaders(uint64_t response_code, Http::ResponseHeaderMapPtr&& headers,
                         UpstreamRequest& upstream_request, bool end_stream) override;
  void onUpstreamData(Buffer::Instance& data, UpstreamRequest& upstream_request,
                      bool end_stream) override;
  void onUpstreamTrailers(Http::ResponseTrailerMapPtr&& trailers,
                          UpstreamRequest& upstream_request) override;
  void onUpstreamMetadata(Http::MetadataMapPtr&& metadata_map) override;

  // ...
               
               }
```
## 下游请求处理接口
### decodeHeaders
```cpp
// source/common/router/router.cc
Http::FilterHeadersStatus Filter::decodeHeaders(Http::RequestHeaderMap& headers, bool end_stream) {
  downstream_headers_ = &headers;

  // ...
  // 判断是否grpc请求
  grpc_request_ = Grpc::Common::isGrpcRequestHeaders(headers);
  // 判断是否需要统计grpc请求的http 错误码
  exclude_http_code_stats_ = grpc_request_ && config_.suppress_grpc_request_failure_code_stats_;

  // 统计
  config_.stats_.rq_total_.inc();

  // Initialize the `modify_headers` function as a no-op (so we don't have to remember to check it
  // against nullptr before calling it), and feed it behavior later if/when we have cluster info
  // headers to append.
  std::function<void(Http::ResponseHeaderMap&)> modify_headers = [](Http::ResponseHeaderMap&) {};

  // 判断当前请求能否路由或直接返回
  // todo callbacks_ 内部
  route_ = callbacks_->route();
  if (!route_) {
    // ...
    // 找不到可以路由的目的地,返回错误给下游
    callbacks_->streamInfo().setResponseFlag(StreamInfo::ResponseFlag::NoRouteFound);
    callbacks_->sendLocalReply(Http::Code::NotFound, "", modify_headers, absl::nullopt,
                               StreamInfo::ResponseCodeDetails::get().RouteNotFound);
    return Http::FilterHeadersStatus::StopIteration;
  }

  // 配置好的直接返回
  const auto* direct_response = route_->directResponseEntry();
  if (direct_response != nullptr) {
    config_.stats_.rq_direct_response_.inc();
    direct_response->rewritePathHeader(headers, !config_.suppress_envoy_headers_);
    callbacks_->streamInfo().setRouteName(direct_response->routeName());
    callbacks_->sendLocalReply(
        direct_response->responseCode(), direct_response->responseBody(),
        [this, direct_response,
         &request_headers = headers](Http::ResponseHeaderMap& response_headers) -> void {
          std::string new_path;
          if (request_headers.Path()) {
            new_path = direct_response->newPath(request_headers);
          }
          // See https://tools.ietf.org/html/rfc7231#section-7.1.2.
          const auto add_location =
              direct_response->responseCode() == Http::Code::Created ||
              Http::CodeUtility::is3xx(enumToInt(direct_response->responseCode()));
          if (!new_path.empty() && add_location) {
            response_headers.addReferenceKey(Http::Headers::get().Location, new_path);
          }
          direct_response->finalizeResponseHeaders(response_headers, callbacks_->streamInfo());
        },
        absl::nullopt, StreamInfo::ResponseCodeDetails::get().DirectResponse);
    return Http::FilterHeadersStatus::StopIteration;
  }

  // 代码到这里就是根据请求信息匹配到了route
  route_entry_ = route_->routeEntry();
  // ...
  // 设置请求的routeName
  callbacks_->streamInfo().setRouteName(route_entry_->routeName());
  // ... 
  // 根据匹配的route的clusterName 得到将要接管数据的cluster结构实例
  Upstream::ThreadLocalCluster* cluster =
      config_.cm_.getThreadLocalCluster(route_entry_->clusterName());
  if (!cluster) {
    // 错误处理
    return Http::FilterHeadersStatus::StopIteration;
  }
  cluster_ = cluster->info();

  // Set up stat prefixes, etc.
  // todo
  request_vcluster_ = route_entry_->virtualCluster(headers);
  if (request_vcluster_ != nullptr) {
    callbacks_->streamInfo().setVirtualClusterName(request_vcluster_->name());
  }
  route_stats_context_ = route_entry_->routeStatsContext();
  
  // 如果配置了严格的header检测,需要限过一遍
  if (config_.strict_check_headers_ != nullptr) {
    for (const auto& header : *config_.strict_check_headers_) {
      const auto res = FilterUtility::StrictHeaderChecker::checkHeader(headers, header);
      if (!res.valid_) {
        // ...
        // header严格检测不通过的错误处理
        return Http::FilterHeadersStatus::StopIteration;
      }
    }
  }

  // todo
  const Http::HeaderEntry* request_alt_name = headers.EnvoyUpstreamAltStatName();
  if (request_alt_name) {
    alt_stat_prefix_ = std::make_unique<Stats::StatNameDynamicStorage>(
        request_alt_name->value().getStringView(), config_.scope_.symbolTable());
    headers.removeEnvoyUpstreamAltStatName();
  }

  // 配置了maintainenceMode 强制请求返回失败的情形处理
  if (cluster_->maintenanceMode()) {
    // ...
    return Http::FilterHeadersStatus::StopIteration;
  }

  // 获取上游cluster的http协议配置
  const auto& upstream_http_protocol_options = cluster_->upstreamHttpProtocolOptions();

  if (upstream_http_protocol_options.has_value() &&
      (upstream_http_protocol_options.value().auto_sni() ||
       upstream_http_protocol_options.value().auto_san_validation())) {
    // ...
    // 上游sni 和 san认证的处理
  }


  // todo ??
  transport_socket_options_ = Network::TransportSocketOptionsUtility::fromFilterState(
      *callbacks_->streamInfo().filterState());

  // 下游socket的option信息按需带给上游
  if (auto downstream_connection = downstreamConnection(); downstream_connection != nullptr) {
    if (auto typed_state = downstream_connection->streamInfo()
                               .filterState()
                               .getDataReadOnly<Network::UpstreamSocketOptionsFilterState>(
                                   Network::UpstreamSocketOptionsFilterState::key());
        typed_state != nullptr) {
      // ...
      Network::Socket::appendOptions(upstream_options_, downstream_options);
    }
  }

  // 配置的要发给上游socket的信息也要带上
  if (upstream_options_ && callbacks_->getUpstreamSocketOptions()) {
    Network::Socket::appendOptions(upstream_options_, callbacks_->getUpstreamSocketOptions());
  }

  // todo createConnPool
  // 根据cluster信息创建一个链接池
  std::unique_ptr<GenericConnPool> generic_conn_pool = createConnPool(*cluster);

  if (!generic_conn_pool) {
    sendNoHealthyUpstreamResponse();
    return Http::FilterHeadersStatus::StopIteration;
  }

  // 根据headers 和配置 算出当前请求的 超时时间
  timeout_ = FilterUtility::finalTimeout(*route_entry_, headers, !config_.suppress_envoy_headers_,
                                         grpc_request_, hedging_params_.hedge_on_per_try_timeout_,
                                         config_.respect_expected_rq_timeout_);

  const Http::HeaderEntry* header_max_stream_duration_entry =
      headers.EnvoyUpstreamStreamDurationMs();
  if (header_max_stream_duration_entry) {
    dynamic_max_stream_duration_ =
        FilterUtility::tryParseHeaderTimeout(*header_max_stream_duration_entry);
    headers.removeEnvoyUpstreamStreamDurationMs();
  }

  // ...

  // 根据配置决定要不要设置重试信息
  if (route_entry_->includeAttemptCountInResponse()) {
    modify_headers = [modify_headers, this](Http::ResponseHeaderMap& headers) {
      modify_headers(headers);
      headers.setEnvoyAttemptCount(attempt_count_);
    };
  }
  callbacks_->streamInfo().setAttemptCount(attempt_count_);

  // 根据所有已知信息,设置好路由到上游的请求头
  route_entry_->finalizeRequestHeaders(headers, callbacks_->streamInfo(),
                                       !config_.suppress_envoy_headers_);
  FilterUtility::setUpstreamScheme(
      headers, callbacks_->streamInfo().downstreamAddressProvider().sslConnection() != nullptr);

  // 根据已有信息创建`retry_state`实例,后续用来做重试决定
  retry_state_ =
      createRetryState(route_entry_->retryPolicy(), headers, *cluster_, request_vcluster_,
                       route_stats_context_, config_.runtime_, config_.random_,
                       callbacks_->dispatcher(), config_.timeSource(), route_entry_->priority());
  // ...
  // 根据前面创建的连接池generic_conn_pool创建一个新的UpstreamRequest实例
  UpstreamRequestPtr upstream_request =
      std::make_unique<UpstreamRequest>(*this, std::move(generic_conn_pool), can_send_early_data,
                                        /*can_use_http3=*/true);
  // todo?? 将新建的UpstreamRequest实例放到upstream_requests_的队列头
  LinkedList::moveIntoList(std::move(upstream_request), upstream_requests_);
  // 调用新建的 UpstreamRequest 的acceptHeadersFromRouter方法,这个方法会最终将请求发送给目的地
  // 见下方 #ref upstream-request.md
  upstream_requests_.front()->acceptHeadersFromRouter(end_stream);
  // 当已经到了请求的结束时,直接调用 onRequestComplete 完成请求

  if (end_stream) {
    onRequestComplete();
  }

  return Http::FilterHeadersStatus::StopIteration;
}

```
[UpstreamRequest](./upstream-request.md)



### decodeData
```cpp
// source/common/router/router.cc
Http::FilterDataStatus Filter::decodeData(Buffer::Instance& data, bool end_stream) {
  // upstream_requests_.size() cannot be > 1 because that only happens when a per
  // try timeout occurs with hedge_on_per_try_timeout enabled but the per
  // try timeout timer is not started until onRequestComplete(). It could be zero
  // if the first request attempt has already failed and a retry is waiting for
  // a backoff timer.
  ASSERT(upstream_requests_.size() <= 1);

  bool buffering = (retry_state_ && retry_state_->enabled()) || !active_shadow_policies_.empty() ||
                   (route_entry_ && route_entry_->internalRedirectPolicy().enabled());
  if (buffering &&
      getLength(callbacks_->decodingBuffer()) + data.length() > retry_shadow_buffer_limit_) {
    ENVOY_LOG(debug,
              "The request payload has at least {} bytes data which exceeds buffer limit {}. Give "
              "up on the retry/shadow.",
              getLength(callbacks_->decodingBuffer()) + data.length(), retry_shadow_buffer_limit_);
    cluster_->stats().retry_or_shadow_abandoned_.inc();
    retry_state_.reset();
    buffering = false;
    active_shadow_policies_.clear();
    request_buffer_overflowed_ = true;

    // If we had to abandon buffering and there's no request in progress, abort the request and
    // clean up. This happens if the initial upstream request failed, and we are currently waiting
    // for a backoff timer before starting the next upstream attempt.
    if (upstream_requests_.empty()) {
      cleanup();
      callbacks_->sendLocalReply(
          Http::Code::InsufficientStorage, "exceeded request buffer limit while retrying upstream",
          modify_headers_, absl::nullopt,
          StreamInfo::ResponseCodeDetails::get().RequestPayloadExceededRetryBufferLimit);
      return Http::FilterDataStatus::StopIterationNoBuffer;
    }
  }

  // If we aren't buffering and there is no active request, an abort should have occurred
  // already.
  ASSERT(buffering || !upstream_requests_.empty());

  if (buffering) {
    // If we are going to buffer for retries or shadowing, we need to make a copy before encoding
    // since it's all moves from here on.
    if (!upstream_requests_.empty()) {
      Buffer::OwnedImpl copy(data);
      upstream_requests_.front()->acceptDataFromRouter(copy, end_stream);
    }

    // If we are potentially going to retry or shadow this request we need to buffer.
    // This will not cause the connection manager to 413 because before we hit the
    // buffer limit we give up on retries and buffering. We must buffer using addDecodedData()
    // so that all buffered data is available by the time we do request complete processing and
    // potentially shadow.
    callbacks_->addDecodedData(data, true);
  } else {
    upstream_requests_.front()->acceptDataFromRouter(data, end_stream);
  }

  if (end_stream) {
    onRequestComplete();
  }

  return Http::FilterDataStatus::StopIterationNoBuffer;
}

```

### onRequestComplete
下游的请求完全接收后,最终都是调用这个方法(前面的decodeXXX方法里的end_stream参数表计下游请求接收完了)
```cpp
// source/common/router/router.cc
void Filter::onRequestComplete() {
  // 标记
  downstream_end_stream_ = true;
  // todo ??
  Event::Dispatcher& dispatcher = callbacks_->dispatcher();
  downstream_request_complete_time_ = dispatcher.timeSource().monotonicTime();

  // Possible that we got an immediate reset.
  if (!upstream_requests_.empty()) {
    // 如果配置了全局超时时间,设置超时回调 onResponseTimeout() todo
    if (timeout_.global_timeout_.count() > 0) {
      response_timeout_ = dispatcher.createTimer([this]() -> void { onResponseTimeout(); });
      response_timeout_->enableTimer(timeout_.global_timeout_);
    }

    // 遍历所有要发送给上游的request
    // 设置每个request的timeout
    // todo 具体发送是在哪儿??
    for (auto& upstream_request : upstream_requests_) {
      if (upstream_request->createPerTryTimeoutOnRequestComplete()) {
        upstream_request->setupPerTryTimeout();
      }
    }
  }
}
```