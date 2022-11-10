# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
UpstreamRequest 在 http filter的最后一个filter router中被生成并使用,用来将请求发送给上游

# 关键interface
```cpp
// source/common/router/upstream_request.h
class UpstreamRequest : public Logger::Loggable<Logger::Id::router>,
                        public UpstreamToDownstream,
                        public LinkedObject<UpstreamRequest>,
                        public GenericConnectionPoolCallbacks,
                        public Event::DeferredDeletable {
public:
  // 在router-filter中根据router-filter的信息和一个连接池(conn_pool)创建UpstreamRequest实例
  UpstreamRequest(RouterFilterInterface& parent, std::unique_ptr<GenericConnPool>&& conn_pool,
                  bool can_send_early_data, bool can_use_http3);
  ~UpstreamRequest() override;
  
  // router-filter处理完下游的请求信息后,通过这几个方法把数据交给当前UpstreamReqeust实例处理
  void acceptHeadersFromRouter(bool end_stream);
  void acceptDataFromRouter(Buffer::Instance& data, bool end_stream);
  void acceptTrailersFromRouter(Http::RequestTrailerMap& trailers);
  void acceptMetadataFromRouter(Http::MetadataMapPtr&& metadata_map_ptr);
  // ...

  // Http::StreamDecoder
  void decodeData(Buffer::Instance& data, bool end_stream) override;
  void decodeMetadata(Http::MetadataMapPtr&& metadata_map) override;

  // UpstreamToDownstream (Http::ResponseDecoder)
  void decode1xxHeaders(Http::ResponseHeaderMapPtr&& headers) override;
  void decodeHeaders(Http::ResponseHeaderMapPtr&& headers, bool end_stream) override;
  void decodeTrailers(Http::ResponseTrailerMapPtr&& trailers) override;
  void dumpState(std::ostream& os, int indent_level) const override;

  // 实例的conn_pool指向的连接池就绪或失败时回调
  void onPoolFailure(ConnectionPool::PoolFailureReason reason,
                     absl::string_view transport_failure_reason,
                     Upstream::HostDescriptionConstSharedPtr host) override;
  void onPoolReady(std::unique_ptr<GenericUpstream>&& upstream,
                   Upstream::HostDescriptionConstSharedPtr host,
                   const Network::Address::InstanceConstSharedPtr& upstream_local_address,
                   StreamInfo::StreamInfo& info, absl::optional<Http::Protocol> protocol) override;
  UpstreamToDownstream& upstreamToDownstream() override;
  // ...
private:
  // ...
  // 管理当前UpstreamRequest的RouterFilter的引用
  RouterFilterInterface& parent_;
  // 当前UpstreamReqeust实例用到的连接池
  std::unique_ptr<GenericConnPool> conn_pool_;
  // ...
  // 当前UpstreamRequest实例用到的Upstream
  std::unique_ptr<GenericUpstream> upstream_;
  // ...
};

```
# 实现
## 创建UpstreamRequest
```cpp
// source/common/router/upstream_request.cc
// 创建UpstreamRequest时需要把routerFilter信息与conn_pool作为参数穿进实例
UpstreamRequest::UpstreamRequest(RouterFilterInterface& parent,
                                 std::unique_ptr<GenericConnPool>&& conn_pool,
                                 bool can_send_early_data, bool can_use_http3)
    : parent_(parent), conn_pool_(std::move(conn_pool)), grpc_rq_success_deferred_(false),
      stream_info_(parent_.callbacks()->dispatcher().timeSource(), nullptr),
      start_time_(parent_.callbacks()->dispatcher().timeSource().monotonicTime()),
      calling_encode_headers_(false), upstream_canary_(false), router_sent_end_stream_(false),
      encode_trailers_(false), retried_(false), awaiting_headers_(true),
      outlier_detection_timeout_recorded_(false),
      create_per_try_timeout_on_request_complete_(false), paused_for_connect_(false),
      reset_stream_(false),
      record_timeout_budget_(parent_.cluster()->timeoutBudgetStats().has_value()),
      cleaned_up_(false), had_upstream_(false),
      // allow_upstream_filters_ 实验功能,暂时不表
      allow_upstream_filters_(
          Runtime::runtimeFeatureEnabled("envoy.reloadable_features.allow_upstream_filters")),
      stream_options_({can_send_early_data, can_use_http3}) {
  if (parent_.config().start_child_span_) {
    span_ = parent_.callbacks()->activeSpan().spawnChild(
        parent_.callbacks()->tracingConfig(), "router " + parent.cluster()->name() + " egress",
        parent.timeSource().systemTime());
    if (parent.attemptCount() != 1) {
      // This is a retry request, add this metadata to span.
      span_->setTag(Tracing::Tags::get().RetryCount, std::to_string(parent.attemptCount() - 1));
    }
  }
  stream_info_.setUpstreamInfo(std::make_shared<StreamInfo::UpstreamInfoImpl>());
  parent_.callbacks()->streamInfo().setUpstreamInfo(stream_info_.upstreamInfo());

  stream_info_.healthCheck(parent_.callbacks()->streamInfo().healthCheck());
  absl::optional<Upstream::ClusterInfoConstSharedPtr> cluster_info =
      parent_.callbacks()->streamInfo().upstreamClusterInfo();
  if (cluster_info.has_value()) {
    stream_info_.setUpstreamClusterInfo(*cluster_info);
  }
  if (!allow_upstream_filters_) { // 实验功能点,暂时认为没有开启
    return;
  }
  // ...
}
```
## acceptHeadersFromRouter
在router-filter对请求头经过处理后,生成了‘UpstreamRequest’实例,并调用‘acceptHeadersFromRouter’方法,表示已经接收好了从下游传过来的header信息,并交给UpstreamRequest实例处理
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::acceptHeadersFromRouter(bool end_stream) {
  // ...
  // 标记是否已经处理到了下游stream的末尾了(有可能整个下游请求就只有header,没有其他部分)
  router_sent_end_stream_ = end_stream;

  // 在与上游的连接池中根据‘this‘信息调用newStream方法创建新stream
  conn_pool_->newStream(this);
  // ...
}
```

## acceptDataFromRouter
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::acceptDataFromRouter(Buffer::Instance& data, bool end_stream) {
  // ...
  // 标记是否已经处理到了下游stream的末尾了(有可能整个下游请求就只有header,没有其他部分)
  router_sent_end_stream_ = end_stream;

  if (!allow_upstream_filters_) {// 实验功能点,暂时认为没有开启,调用完acceptDataFromRouterOld后就return了
    acceptDataFromRouterOld(data, end_stream);
    return;
  }

  // ...
}

void UpstreamRequest::acceptDataFromRouterOld(Buffer::Instance& data, bool end_stream) {
  if (!upstream_ || paused_for_connect_) { // 只有当UpstreamRequest实例的poolReady了,upstream_才会有值,还没有ready时需要先缓存一下数据, todo paused_for_connect_, todo #ref Upstream
    // 不存在buffer时新建
    if (!buffered_request_body_) {
      buffered_request_body_ = parent_.callbacks()->dispatcher().getWatermarkFactory().createBuffer(
          [this]() -> void { this->enableDataFromDownstreamForFlowControl(); },
          [this]() -> void { this->disableDataFromDownstreamForFlowControl(); },
          []() -> void { /* TODO(adisuissa): Handle overflow watermark */ });
      buffered_request_body_->setWatermarks(parent_.callbacks()->decoderBufferLimit());
    }
    // 将数据move至buffer中
    buffered_request_body_->move(data);
  } else {
    // 设置stream要传送的数据长度信息
    stream_info_.addBytesSent(data.length());
    // 调用Upstream 实例的encodeData方法将数据交给Upstream实例处理(并发送给上游) todo #ref Upstream
    upstream_->encodeData(data, end_stream);
    if (end_stream) {
      upstreamTiming().onLastUpstreamTxByteSent(parent_.callbacks()->dispatcher().timeSource());
    }
  }
}
```
## acceptTrailersFromRouter (略)
## acceptMetadataFromRouter (略)

## onPoolReady
UpstreamRequest实例中有一个指向连接池的conn_pool指针,当pool就绪(ready)时会回调Upstream实例的onPoolReady方法
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::onPoolReady( // todo Upstream 和 Host 的区别
    std::unique_ptr<GenericUpstream>&& upstream, Upstream::HostDescriptionConstSharedPtr host,
    const Network::Address::InstanceConstSharedPtr& upstream_local_address,
    StreamInfo::StreamInfo& info, absl::optional<Http::Protocol> protocol) {
  // 复制Upstream信息到本地,这样在后续Upstream调用acceptxxxFromRouter方法时,就可以认为upstream_可用,从而把信息传给upstream
  upstream_ = std::move(upstream);
  had_upstream_ = true;

  // 回调router-filter的onUpstreamHostSelected方法 todo
  onUpstreamHostSelected(host);

  // ...
  // todo ??
  for (auto* callback : upstream_callbacks_) {
    callback->onUpstreamConnectionEstablished();
    return;
  }

  // 调用upstream_的 encodeXXX方法,将下游的数据交给upstream处理
  const Http::Status status =
      upstream_->encodeHeaders(*parent_.downstreamHeaders(), shouldSendEndStream());
  calling_encode_headers_ = false;
  // ...

  if (!paused_for_connect_) {
    encodeBodyAndTrailers();
  }
}

```

## onPoolFailure
UpstreamRequest实例中有一个指向连接池的conn_pool指针,当pool失败(failure)时会回调Upstream实例的onPoolReady方法

```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::onPoolFailure(ConnectionPool::PoolFailureReason reason,
                                    absl::string_view transport_failure_reason,
                                    Upstream::HostDescriptionConstSharedPtr host) {
  // 封装连接池错误
  Http::StreamResetReason reset_reason = Http::StreamResetReason::ConnectionFailure;
  switch (reason) {
  case ConnectionPool::PoolFailureReason::Overflow:
    reset_reason = Http::StreamResetReason::Overflow;
    break;
  case ConnectionPool::PoolFailureReason::RemoteConnectionFailure:
    FALLTHRU;
  case ConnectionPool::PoolFailureReason::LocalConnectionFailure:
    reset_reason = Http::StreamResetReason::ConnectionFailure;
    break;
  case ConnectionPool::PoolFailureReason::Timeout:
    reset_reason = Http::StreamResetReason::ConnectionFailure;
  }
  // 设置stream信息
  stream_info_.upstreamInfo()->setUpstreamTransportFailureReason(transport_failure_reason);

  // todo
  onUpstreamHostSelected(host);
  onResetStream(reset_reason, transport_failure_reason);
}
```




