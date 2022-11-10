# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
ThreadLocalCluster 是每个工作线程都有的一个cluster的本地实例,主要工作:
1. 与clusterManager交流,缓存一个cluster的信息(减少同一个cluster需要在不同工作线程间的信息同步);
2. 提供本地方法给routerFilter等

# interface
```cpp
// source/common/upstream/cluster_manager_impl.h
    class ClusterEntry : public ThreadLocalCluster {
    public:
      ClusterEntry(ThreadLocalClusterManagerImpl& parent, ClusterInfoConstSharedPtr cluster,
                   const LoadBalancerFactorySharedPtr& lb_factory);
      ~ClusterEntry() override;

      // Upstream::ThreadLocalCluster
      const PrioritySet& prioritySet() override { return priority_set_; }
      ClusterInfoConstSharedPtr info() override { return cluster_info_; }
      // 负载平衡
      LoadBalancer& loadBalancer() override { return *lb_; }
      // 从当前cluster中获取一个‘Http’连接池
      absl::optional<HttpPoolData> httpConnPool(ResourcePriority priority,
                                                absl::optional<Http::Protocol> downstream_protocol,
                                                LoadBalancerContext* context) override;
      // 从当前cluster中获取一个‘TCP’连接池
      absl::optional<TcpPoolData> tcpConnPool(ResourcePriority priority,
                                              LoadBalancerContext* context) override;
      Host::CreateConnectionData tcpConn(LoadBalancerContext* context) override;
      Http::AsyncClient& httpAsyncClient() override;

      // 下面这些方法应该都是哥clusterManager调用来更新信息的 todo
      // Updates the hosts in the priority set.
      void updateHosts(const std::string& name, uint32_t priority,
                       PrioritySet::UpdateHostsParams&& update_hosts_params,
                       LocalityWeightsConstSharedPtr locality_weights,
                       const HostVector& hosts_added, const HostVector& hosts_removed,
                       absl::optional<uint32_t> overprovisioning_factor,
                       HostMapConstSharedPtr cross_priority_host_map);

      // Drains any connection pools associated with the removed hosts.
      void drainConnPools(const HostVector& hosts_removed);
      // Drains idle clients in connection pools for all hosts.
      void drainConnPools();
      // Drain all clients in connection pools for all hosts.
      void drainAllConnPools(DrainConnectionsHostPredicate predicate);

    private:
      Http::ConnectionPool::Instance*

      // ThreadLocalClusterManager实例,方便回调manager的方法
      ThreadLocalClusterManagerImpl& parent_;
      // 负载均衡器,用来在获取连接池前从cluster列表中挑选出一个具体的Host
      LoadBalancerPtr lb_;
     
    };
```

# 逻辑实现
## httpConnPool
根据当前cluster信息选择一个host创建http连接池
```cpp
// source/common/upstream/cluster_manager_impl.cc
absl::optional<HttpPoolData>
ClusterManagerImpl::ThreadLocalClusterManagerImpl::ClusterEntry::httpConnPool(
    ResourcePriority priority, absl::optional<Http::Protocol> protocol,
    LoadBalancerContext* context) {
  // 调用私有方法根据当前cluster信息创建http连接池 #ref httpConnPoolImpl
  auto pool = httpConnPoolImpl(priority, protocol, context, false);
  if (pool == nullptr) {
    return absl::nullopt;
  }
  // 封装连接池到数据结构‘HttpPoolData’实例中, TODO
  HttpPoolData data(
      [this, priority, protocol, context]() -> void {
        // Now that a new stream is being established, attempt to preconnect.
        maybePreconnect(*this, parent_.cluster_manager_state_,
                        [this, &priority, &protocol, &context]() {
                          return httpConnPoolImpl(priority, protocol, context, true);
                        });
      },
      pool);
  return data;
}

Http::ConnectionPool::Instance*
ClusterManagerImpl::ThreadLocalClusterManagerImpl::ClusterEntry::httpConnPoolImpl(
    ResourcePriority priority, absl::optional<Http::Protocol> downstream_protocol,
    LoadBalancerContext* context, bool peek) {
  // 由从当前cluster的lb规则挑选出要创建连接池的目标host
  HostConstSharedPtr host = (peek ? lb_->peekAnotherHost(context) : lb_->chooseHost(context));
  // ...

  // 先通过cluster信息得到与上游通信的协议,http1,http2等
  auto upstream_protocols = host->cluster().upstreamHttpProtocol(downstream_protocol);
  std::vector<uint8_t> hash_key;
  // 根据协议信息为连接池生成一个hash_key,便于复用连接池
  hash_key.reserve(upstream_protocols.size());
  for (auto protocol : upstream_protocols) {
    hash_key.push_back(uint8_t(protocol));
  }

  // ...
  // 调用parent_(指向ThreadLocalClusterManager实例)的方法‘getHttpConnPoolsContainer’获取一个host的连接池容器
  ConnPoolsContainer& container = *parent_.getHttpConnPoolsContainer(host, true);

  // 从连接池容器中取出连接池
  // 传入了两个回调,一个用来在找不到hash_key符合的连接池时新建连接池
  // 一个用来在连接池空闲时回收
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
