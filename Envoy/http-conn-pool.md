# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
router filter 需要根据路由的cluster创建一个到目标地址的连接池(当cluster是http时,使用HttpConnPool),然后使用连接池中的连接将UpstreamRequest最终发送给目标地址

# 接口
## HttpConnPool
```cpp
// source/extensions/upstreams/http/http/upstream_request.h
class HttpConnPool : public Router::GenericConnPool, public Envoy::Http::ConnectionPool::Callbacks {
public:
  // GenericConnPool
  HttpConnPool(Upstream::ThreadLocalCluster& thread_local_cluster, bool is_connect,
               const Router::RouteEntry& route_entry,
               absl::optional<Envoy::Http::Protocol> downstream_protocol,
               Upstream::LoadBalancerContext* ctx) {
    // 通过传进来的cluster实例的httpConnPool方法创建pool_data实例
    pool_data_ =
        thread_local_cluster.httpConnPool(route_entry.priority(), downstream_protocol, ctx);
  }
  ~HttpConnPool() override {
    // ...
  }
  // 在连接池中创建一条新stream用来发送数据
  void newStream(Router::GenericConnectionPoolCallbacks* callbacks) override;
  bool cancelAnyPendingStream() override;

  // Http::ConnectionPool::Callbacks
  void onPoolFailure(ConnectionPool::PoolFailureReason reason,
                     absl::string_view transport_failure_reason,
                     Upstream::HostDescriptionConstSharedPtr host) override;
  void onPoolReady(Envoy::Http::RequestEncoder& callbacks_encoder,
                   Upstream::HostDescriptionConstSharedPtr host, StreamInfo::StreamInfo& info,
                   absl::optional<Envoy::Http::Protocol> protocol) override;
  // 在这里可以看出一个conn_pool的目标对象是一个host(而不是一个cluster)
  Upstream::HostDescriptionConstSharedPtr host() const override {
    return pool_data_.value().host();
  }

  bool valid() { return pool_data_.has_value(); }

protected:
  // Points to the actual connection pool to create streams from.
  absl::optional<Envoy::Upstream::HttpPoolData> pool_data_{};
  Envoy::Http::ConnectionPool::Cancellable* conn_pool_stream_handle_{};
  Router::GenericConnectionPoolCallbacks* callbacks_{};
};
```

## HttpPoolData
```cpp
class HttpPoolData {
public:
  using OnNewStreamFn = std::function<void()>;

  HttpPoolData(OnNewStreamFn on_new_stream, Http::ConnectionPool::Instance* pool)
      : on_new_stream_(on_new_stream), pool_(pool) {}

  /**
   * See documentation of Http::ConnectionPool::Instance.
   */
  Envoy::Http::ConnectionPool::Cancellable*
  newStream(Http::ResponseDecoder& response_decoder,
            Envoy::Http::ConnectionPool::Callbacks& callbacks,
            const Http::ConnectionPool::Instance::StreamOptions& stream_options) {
    on_new_stream_();
    return pool_->newStream(response_decoder, callbacks, stream_options);
  }
  bool hasActiveConnections() const { return pool_->hasActiveConnections(); };

  /**
   * See documentation of Envoy::ConnectionPool::Instance.
   */
  void addIdleCallback(ConnectionPool::Instance::IdleCb cb) { pool_->addIdleCallback(cb); };
  void drainConnections(ConnectionPool::DrainBehavior drain_behavior) {
    pool_->drainConnections(drain_behavior);
  };

  Upstream::HostDescriptionConstSharedPtr host() const { return pool_->host(); }

private:
  friend class HttpPoolDataPeer;

  OnNewStreamFn on_new_stream_;
  Http::ConnectionPool::Instance* pool_;
};

```

# 逻辑
## newStream
```cpp
```


# 代码
## 
```cpp
// source/extensions/upstreams/http/http/upstream_request.h
```
### conn_pool_->newStream(this)
举例假设upstream-request是http类型的
```cpp
// source/extensions/upstreams/http/http/upstream_request.cc
void HttpConnPool::newStream(GenericConnectionPoolCallbacks* callbacks) {
  callbacks_ = callbacks;
  // 调用poolData 的 newStream方法创建一个handle,后续对这个新stream的处理都交由这个handle
  Envoy::Http::ConnectionPool::Cancellable* handle =
      // upstreamToDownstream变成了pool_data实例的response_decoder,后续收到了请求的回复会交由这个decoder处理
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
  return tryCreateNewConnection(global_preconnect_ratio) == ConnectionResult::CreatedNewConnection;
}
```
#### tryCreateNewConnection
```cpp
// source/common/conn_pool/conn_pool_base.cc
ConnPoolImplBase::ConnectionResult
ConnPoolImplBase::tryCreateNewConnection(float global_preconnect_ratio) {
  // 先判断是不是需要创建新的连接
  if (!shouldCreateNewConnection(global_preconnect_ratio)) {
    ENVOY_LOG(trace, "not creating a new connection, shouldCreateNewConnection returned false.");
    return ConnectionResult::ShouldNotConnect;
  }

  // 根据host所在的cluster的各种资源限制配置,判断可不可以创建新的连接
  const bool can_create_connection = host_->canCreateConnection(priority_);

  if (!can_create_connection) {
    host_->cluster().stats().upstream_cx_overflow_.inc();
  }
  
  // can_create_connection 为真时当然需要创建新连接
  // 但另一种情况,当前upstream一个连接也没有时,也需要强行创建一个
  if (can_create_connection || (ready_clients_.empty() && busy_clients_.empty() &&
                                connecting_clients_.empty() && early_data_clients_.empty())) {
    // 新建一个client连接 todo ??
    ActiveClientPtr client = instantiateActiveClient();
    // ...
    // 更新这个连接client的处理能力信息
    incrConnectingAndConnectedStreamCapacity(client->currentUnusedCapacity(), *client);
    // 根据client状态,将连接加入到当前连接池的相应list中
    LinkedList::moveIntoList(std::move(client), owningList(client->state()));
    return can_create_connection ? ConnectionResult::CreatedNewConnection
                                 : ConnectionResult::CreatedButRateLimited;
  } else {
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
