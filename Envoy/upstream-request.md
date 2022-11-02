# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
UpstreamRequest 在 http filter的最后一个filter router中被生成并使用,用来将请求发送给上游

# 关键interface
## 创建UpstreamRequest
```cpp
// source/common/router/upstream_request.cc
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
  if (!allow_upstream_filters_) {
    return;
  }
  // ...
}
```
## acceptHeadersFromRouter
在router-filter对请求头经过处理后,生成了‘UpstreamRequest’实例,并调用‘acceptHeadersFromRouter’方法
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::acceptHeadersFromRouter(bool end_stream) {
  // ...

  // 在与上游的连接池中根据‘this;信息创建stream(并开始传输数据), todo 不同类型的conn_pool以及他们的newstream方法
  conn_pool_->newStream(this);
  // ...
}
```

### conn_pool_->newStream(this)
举例假设upstream-request是http类型的
```cpp
// source/extensions/upstreams/http/http/upstream_request.cc
void HttpConnPool::newStream(GenericConnectionPoolCallbacks* callbacks) {
  callbacks_ = callbacks;
  // 调用poolData 的 newStream方法创建一个handle,后续对这个新stream的处理都交由这个handle
  Envoy::Http::ConnectionPool::Cancellable* handle =
      pool_data_.value().newStream(callbacks->upstreamToDownstream(), *this,
                                   callbacks->upstreamToDownstream().upstreamStreamOptions());
  if (handle) {
    conn_pool_stream_handle_ = handle;
  }
}
```

### HttpPoolData.newStream方法
以HttpPoolData类型举例阐释newStream方法
```cpp
// envoy/upstream/thread_local_cluster.h
class HttpPoolData {
public:
  using OnNewStreamFn = std::function<void()>;

  // HttpPoolData 在新建时需要传入一个OnNewStreamFn方法,用来在调用newStream方法时触发
  HttpPoolData(OnNewStreamFn on_new_stream, Http::ConnectionPool::Instance* pool)
      : on_new_stream_(on_new_stream), pool_(pool) {}

  /**
   * See documentation of Http::ConnectionPool::Instance.
   */
  Envoy::Http::ConnectionPool::Cancellable*
  newStream(Http::ResponseDecoder& response_decoder,
            Envoy::Http::ConnectionPool::Callbacks& callbacks,
            const Http::ConnectionPool::Instance::StreamOptions& stream_options) {
    // 触发新建HttpPoolData实例时注入的 on_new_stream_ 函数 #ref on_new_stream_
    on_new_stream_();
    // 调用注入的pool 的newStream方法
    return pool_->newStream(response_decoder, callbacks, stream_options);
  }
  //...
}
```

### on_new_stream_
httpConnPool 创建时,会创建HttpPoolData, 并将一个匿名函数传入当作httpPoolData的 on_new_stream_参数
```cpp
// source/common/upstream/cluster_manager_impl.cc
absl::optional<HttpPoolData>
ClusterManagerImpl::ThreadLocalClusterManagerImpl::ClusterEntry::httpConnPool(
    ResourcePriority priority, absl::optional<Http::Protocol> protocol,
    LoadBalancerContext* context) {
  // 新建pool #ref 
  auto pool = httpConnPoolImpl(priority, protocol, context, false);
  if (pool == nullptr) {
    return absl::nullopt;
  }

  HttpPoolData data(
      // 这个方法就是前面会使用的on_new_stream_函数
      [this, priority, protocol, context]() -> void {
        // 在新建与上游的stream时会调用这里(并开始发送数据给上游) #ref maybePreconnect
        // 先通过调用传入的回调创建一个Pool,然后调用pool的‘maybePreConnect’方法
        maybePreconnect(*this, parent_.cluster_manager_state_,
                        [this, &priority, &protocol, &context]() {
                          // 构造一个httpConnPool #ref httpConnPoolImpl
                          return httpConnPoolImpl(priority, protocol, context, true);
                        });
      },
      pool);
  return data;
}
```
### maybePreconnect 调用参数中的函数创建pool实例,然后调用实例的'maybePreconnect'方法
```cpp
// source/common/upstream/cluster_manager_impl.cc
void ClusterManagerImpl::maybePreconnect(
    ThreadLocalClusterManagerImpl::ClusterEntry& cluster_entry,
    const ClusterConnectivityState& state,
    std::function<ConnectionPool::Instance*()> pick_preconnect_pool) {
  // ...
  // 写死最多三次
  for (int i = 0; i < 3; ++i) {
    // See if adding this one new connection
    // would put the cluster over desired capacity. If so, stop preconnecting.
    //
    // We anticipate the incoming stream here, because maybePreconnect is called
    // before a new stream is established.
    if (!ConnectionPool::ConnPoolImplBase::shouldConnect(
            state.pending_streams_, state.active_streams_,
            state.connecting_and_connected_stream_capacity_, peekahead_ratio, true)) {
      return;
    }
    ConnectionPool::Instance* preconnect_pool = pick_preconnect_pool();
    if (!preconnect_pool || !preconnect_pool->maybePreconnect(peekahead_ratio)) {
      // Given that the next preconnect pick may be entirely different, we could
      // opt to try again even if the first preconnect fails. Err on the side of
      // caution and wait for the next attempt.
      return;
    }
  }
}
```
```cpp
// source/common/http/conn_pool_base.h
class HttpConnPoolImplBase : public Envoy::ConnectionPool::ConnPoolImplBase,
                             public Http::ConnectionPool::Instance {
public:
  // ...
  bool maybePreconnect(float ratio) override { return maybePreconnectImpl(ratio); }
  // ...
                             }
```

```cpp
// source/common/conn_pool/conn_pool_base.cc
bool ConnPoolImplBase::maybePreconnectImpl(float global_preconnect_ratio) {
  ASSERT(!deferred_deleting_);
  return tryCreateNewConnection(global_preconnect_ratio) == ConnectionResult::CreatedNewConnection;
}
```
#### tryCreateNewConnection
```cpp
ConnPoolImplBase::ConnectionResult
ConnPoolImplBase::tryCreateNewConnection(float global_preconnect_ratio) {
  // There are already enough Connecting connections for the number of queued streams.
  if (!shouldCreateNewConnection(global_preconnect_ratio)) {
    ENVOY_LOG(trace, "not creating a new connection, shouldCreateNewConnection returned false.");
    return ConnectionResult::ShouldNotConnect;
  }

  const bool can_create_connection = host_->canCreateConnection(priority_);

  if (!can_create_connection) {
    host_->cluster().stats().upstream_cx_overflow_.inc();
  }
  // If we are at the connection circuit-breaker limit due to other upstreams having
  // too many open connections, and this upstream has no connections, always create one, to
  // prevent pending streams being queued to this upstream with no way to be processed.
  if (can_create_connection || (ready_clients_.empty() && busy_clients_.empty() &&
                                connecting_clients_.empty() && early_data_clients_.empty())) {
    ENVOY_LOG(debug, "creating a new connection (connecting={})", connecting_clients_.size());
    ActiveClientPtr client = instantiateActiveClient();
    if (client.get() == nullptr) {
      ENVOY_LOG(trace, "connection creation failed");
      return ConnectionResult::FailedToCreateConnection;
    }
    ASSERT(client->state() == ActiveClient::State::Connecting);
    ASSERT(std::numeric_limits<uint64_t>::max() - connecting_stream_capacity_ >=
           static_cast<uint64_t>(client->currentUnusedCapacity()));
    ASSERT(client->real_host_description_);
    // Increase the connecting capacity to reflect the streams this connection can serve.
    incrConnectingAndConnectedStreamCapacity(client->currentUnusedCapacity(), *client);
    LinkedList::moveIntoList(std::move(client), owningList(client->state()));
    return can_create_connection ? ConnectionResult::CreatedNewConnection
                                 : ConnectionResult::CreatedButRateLimited;
  } else {
    ENVOY_LOG(trace, "not creating a new connection: connection constrained");
    return ConnectionResult::NoConnectionRateLimited;
  }
}
```

#### httpConnPoolImpl
创建到上游的http连接池
```cpp
// source/common/upstream/cluster_manager_impl.cc
Http::ConnectionPool::Instance*
ClusterManagerImpl::ThreadLocalClusterManagerImpl::ClusterEntry::httpConnPoolImpl(
    ResourcePriority priority, absl::optional<Http::Protocol> downstream_protocol,
    LoadBalancerContext* context, bool peek) {
  // 从当前cluster的lb规则中挑出一个host
  HostConstSharedPtr host = (peek ? lb_->peekAnotherHost(context) : lb_->chooseHost(context));
  // ...
 
  // 根据host找到该host的连接池容器(后面的true参数表示找不到则新建一个)
  // #ref getHttpConnPoolsContainer
  ConnPoolsContainer& container = *parent_.getHttpConnPoolsContainer(host, true);

  // 从连接池容器中获取一个连接池(如果容器中没有,则调用传入的factory_.allocateConnPool新建一个)
  ConnPoolsContainer::ConnPools::PoolOptRef pool =
      container.pools_->getPool(priority, hash_key, [&]() {
        auto pool = parent_.parent_.factory_.allocateConnPool(
            parent_.thread_local_dispatcher_, host, priority, upstream_protocols,
            alternate_protocol_options, !upstream_options->empty() ? upstream_options : nullptr,
            have_transport_socket_options ? context->upstreamTransportSocketOptions() : nullptr,
            parent_.parent_.time_source_, parent_.cluster_manager_state_, quic_info_);

        pool->addIdleCallback([&parent = parent_, host, priority, hash_key]() {
          parent.httpConnPoolIsIdle(host, priority, hash_key);
        });

        return pool;
      });

  if (pool.has_value()) {
    return &(pool.value().get());
  } else {
    return nullptr;
  }
}

```

#### getHttpConnPoolsContainer 获取(或创建)一个host的连接池容器
```cpp
// source/common/upstream/cluster_manager_impl.cc
ClusterManagerImpl::ThreadLocalClusterManagerImpl::ConnPoolsContainer*
ClusterManagerImpl::ThreadLocalClusterManagerImpl::getHttpConnPoolsContainer(
    const HostConstSharedPtr& host, bool allocate) {
  // 从当前‘clusterManager’的‘host_http_conn_pool_map_’中寻找参数‘host’的连接池容器
  auto container_iter = host_http_conn_pool_map_.find(host);
  if (container_iter == host_http_conn_pool_map_.end()) {
    if (!allocate) {
      return nullptr;
    }
    // 找不到则新建一个连接池容器,并加入到‘host_http_conn_pool_map_’中
    ConnPoolsContainer container{thread_local_dispatcher_, host};
    container_iter = host_http_conn_pool_map_.emplace(host, std::move(container)).first;
  }

  return &container_iter->second;
}
```
