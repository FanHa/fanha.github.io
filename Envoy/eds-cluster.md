# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
对应envoy配置里的`EDS`类别的cluster

# 代码逻辑
## 创建eds cluster
```cpp
// source/common/upstream/eds.cc
std::pair<ClusterImplBaseSharedPtr, ThreadAwareLoadBalancerPtr>
EdsClusterFactory::createClusterImpl(
    Server::Configuration::ServerFactoryContext& server_context,
    const envoy::config::cluster::v3::Cluster& cluster, ClusterFactoryContext& context,
    Server::Configuration::TransportSocketFactoryContextImpl& socket_factory_context,
    Stats::ScopeSharedPtr&& stats_scope) {
  if (!cluster.has_eds_cluster_config()) {
    throw EnvoyException("cannot create an EDS cluster without an EDS config");
  }

  return std::make_pair(std::make_unique<EdsClusterImpl>(
                            server_context, cluster, context.runtime(), socket_factory_context,
                            std::move(stats_scope), context.addedViaApi()),
                        nullptr);
}

EdsClusterImpl::EdsClusterImpl(
    Server::Configuration::ServerFactoryContext& server_context,
    const envoy::config::cluster::v3::Cluster& cluster, Runtime::Loader& runtime,
    Server::Configuration::TransportSocketFactoryContextImpl& factory_context,
    Stats::ScopeSharedPtr&& stats_scope, bool added_via_api)
    : BaseDynamicClusterImpl(server_context, cluster, runtime, factory_context,
                             std::move(stats_scope), added_via_api,
                             factory_context.mainThreadDispatcher().timeSource()),
      Envoy::Config::SubscriptionBase<envoy::config::endpoint::v3::ClusterLoadAssignment>(
          factory_context.messageValidationVisitor(), "cluster_name"),
      factory_context_(factory_context), local_info_(factory_context.localInfo()),
      cluster_name_(cluster.eds_cluster_config().service_name().empty()
                        ? cluster.name()
                        : cluster.eds_cluster_config().service_name()) {
  Event::Dispatcher& dispatcher = factory_context.mainThreadDispatcher();
  assignment_timeout_ = dispatcher.createTimer([this]() -> void { onAssignmentTimeout(); });
  const auto& eds_config = cluster.eds_cluster_config().eds_config();
  if (Config::SubscriptionFactory::isPathBasedConfigSource(
          eds_config.config_source_specifier_case())) {
    initialize_phase_ = InitializePhase::Primary;
  } else {
    initialize_phase_ = InitializePhase::Secondary;
  }
  const auto resource_name = getResourceName();
  subscription_ =
      factory_context.clusterManager().subscriptionFactory().subscriptionFromConfigSource(
          eds_config, Grpc::Common::typeUrl(resource_name), info_->statsScope(), *this,
          resource_decoder_, {});
}
```

## host结构
一个host结构大致分为 HostDescription 与 Host两部分:
- HostDescription部分表示一个实际可连接的后端(比如一个IP:Port)
- Host部分表示在cluster视角这个host的`健康状态`和`权重`
```cpp
// source/common/upstream/upstream_impl.h
class HostImpl : public HostDescriptionImpl,
                 public Host,
                 public std::enable_shared_from_this<HostImpl> {
public:
  HostImpl(ClusterInfoConstSharedPtr cluster, const std::string& hostname,
           Network::Address::InstanceConstSharedPtr address, MetadataConstSharedPtr metadata,
           uint32_t initial_weight, const envoy::config::core::v3::Locality& locality,
           const envoy::config::endpoint::v3::Endpoint::HealthCheckConfig& health_check_config,
           uint32_t priority, const envoy::config::core::v3::HealthStatus health_status,
           TimeSource& time_source)
        // 实际可连接的后端Host描述(比如一个具体的IP:port)
      : HostDescriptionImpl(cluster, hostname, address, metadata, locality, health_check_config,
                            priority, time_source),
        used_(true) { 
    setEdsHealthFlag(health_status); // 健康状态
    HostImpl::weight(initial_weight); // 权重
  }
  // ...
};
```
### HostDescriptionImpl
HostDescriptionImpl 包裹了一个有具体地址的结构`Network::Address::InstanceConstSharedPtr`的变量address_,请求就是直接发到这个address_所代表的地址
```cpp
// source/common/upstream/upstream_impl.cc
HostDescriptionImpl::HostDescriptionImpl(
    ClusterInfoConstSharedPtr cluster, const std::string& hostname,
    Network::Address::InstanceConstSharedPtr dest_address, MetadataConstSharedPtr metadata,
    const envoy::config::core::v3::Locality& locality,
    const envoy::config::endpoint::v3::Endpoint::HealthCheckConfig& health_check_config,
    uint32_t priority, TimeSource& time_source)
    : cluster_(cluster), hostname_(hostname),
      health_checks_hostname_(health_check_config.hostname()), address_(dest_address),
      canary_(Config::Metadata::metadataValue(metadata.get(),
                                              Config::MetadataFilters::get().ENVOY_LB,
                                              Config::MetadataEnvoyLbKeys::get().CANARY)
                  .bool_value()),
      metadata_(metadata), locality_(locality),
      locality_zone_stat_name_(locality.zone(), cluster->statsScope().symbolTable()),
      priority_(priority),
      socket_factory_(resolveTransportSocketFactory(dest_address, metadata_.get())),
      creation_time_(time_source.monotonicTime()) {
  // ...
}
```

## 更新cluster里的host信息
```cpp
// source/common/upstream/eds.cc
bool EdsClusterImpl::updateHostsPerLocality(
    const uint32_t priority, const uint32_t overprovisioning_factor, const HostVector& new_hosts,
    LocalityWeightsMap& locality_weights_map, LocalityWeightsMap& new_locality_weights_map,
    PriorityStateManager& priority_state_manager, const HostMap& all_hosts,
    const absl::flat_hash_set<std::string>& all_new_hosts) {
  // todo priority的作用场合
  const auto& host_set = priority_set_.getOrCreateHostSet(priority, overprovisioning_factor);
  HostVectorSharedPtr current_hosts_copy(new HostVector(host_set.hosts()));

  HostVector hosts_added;
  HostVector hosts_removed;
  
  // 调用 #ref 'updateDynamicHostList'方法,比对新hosts列表'new_hosts' 和 当前hosts列表'current_host_copy'
  const bool hosts_updated = updateDynamicHostList(new_hosts, *current_hosts_copy, hosts_added,
                                                   hosts_removed, all_hosts, all_new_hosts);
  if (hosts_updated || host_set.overprovisioningFactor() != overprovisioning_factor ||
      locality_weights_map != new_locality_weights_map) {

    locality_weights_map = new_locality_weights_map;

    priority_state_manager.updateClusterPrioritySet(priority, std::move(current_hosts_copy),
                                                    hosts_added, hosts_removed, absl::nullopt,
                                                    overprovisioning_factor);
    return true;
  }
  return false;
}

// source/common/upstream/upstream_impl.cc
bool BaseDynamicClusterImpl::updateDynamicHostList(
    const HostVector& new_hosts, HostVector& current_priority_hosts,
    HostVector& hosts_added_to_current_priority, HostVector& hosts_removed_from_current_priority,
    const HostMap& all_hosts, const absl::flat_hash_set<std::string>& all_new_hosts) {
  uint64_t max_host_weight = 1;

  // 默认是没有变化
  bool hosts_changed = false;

  // Keep track of hosts we see in new_hosts that we are able to match up with an existing host.
  absl::flat_hash_set<std::string> existing_hosts_for_current_priority(
      current_priority_hosts.size());
  // Keep track of hosts we're adding (or replacing)
  absl::flat_hash_set<std::string> new_hosts_for_current_priority(new_hosts.size());
  // Keep track of hosts for which locality is changed.
  absl::flat_hash_set<std::string> hosts_with_updated_locality_for_current_priority(
      current_priority_hosts.size());
  HostVector final_hosts;
  for (const HostSharedPtr& host : new_hosts) { // 遍历新host列表
    // 寻找当前host是否存在新host
    auto existing_host = all_hosts.find(host->address()->asString());
    const bool existing_host_found = existing_host != all_hosts.end();

    // Clear any pending deletion flag on an existing host in case it came back while it was
    // being stabilized. We will set it again below if needed.
    // TODO second是结构里的什么作用 ?
    if (existing_host_found) {
      existing_host->second->healthFlagClear(Host::HealthFlag::PENDING_DYNAMIC_REMOVAL);
    }

    // 一些需要忽略处理的情况,如host虽然可以找到,但是地址变了;
    // TODO 这个Host 和 这个Address 的关系
    const bool health_check_address_changed =
        (health_checker_ != nullptr && existing_host_found &&
         *existing_host->second->healthCheckAddress() != *host->healthCheckAddress());
    bool locality_changed = false;
    if (Runtime::runtimeFeatureEnabled(
            "envoy.reloadable_features.support_locality_update_on_eds_cluster_endpoints")) {
      locality_changed =
          (existing_host_found &&
           (!LocalityEqualTo()(host->locality(), existing_host->second->locality())));
      if (locality_changed) {
        hosts_with_updated_locality_for_current_priority.emplace(existing_host->first);
      }
    }

    const bool skip_inplace_host_update = health_check_address_changed || locality_changed;

    // When there is a match and we decided to do in-place update, we potentially update the
    // host's health check flag and metadata. Afterwards, the host is pushed back into the
    // final_hosts, i.e. hosts that should be preserved in the current priority.
    // 能在已有的host中找到,且不属于需要忽略的变动
    if (existing_host_found && !skip_inplace_host_update) {
      existing_hosts_for_current_priority.emplace(existing_host->first);
      // If we find a host matched based on address, we keep it. However we do change weight inline
      // so do that here.
      if (host->weight() > max_host_weight) {
        max_host_weight = host->weight();
      }
      // 更新原host的权重weight变化
      if (existing_host->second->weight() != host->weight()) {
        existing_host->second->weight(host->weight());
        hosts_changed = true;
      }

      // 根据新host的的HealthFlag 更新原host 的flag
      hosts_changed |=
          updateHealthFlag(*host, *existing_host->second, Host::HealthFlag::FAILED_EDS_HEALTH);
      hosts_changed |=
          updateHealthFlag(*host, *existing_host->second, Host::HealthFlag::DEGRADED_EDS_HEALTH);

      // 判断这个host 的 metadata信息是否变更
      bool metadata_changed = true;
      if (host->metadata() && existing_host->second->metadata()) {
        metadata_changed = !Protobuf::util::MessageDifferencer::Equivalent(
            *host->metadata(), *existing_host->second->metadata());
      } else if (!host->metadata() && !existing_host->second->metadata()) {
        metadata_changed = false;
      }

      if (metadata_changed) {
        // 更新已有host的metadata信息 
        existing_host->second->metadata(host->metadata());

        // ... todo canary 的作用
        existing_host->second->canary(host->canary());

        hosts_changed = true;
      }

      // todo priority的作用
      if (host->priority() != existing_host->second->priority()) {
        // 更新原host的priority级别
        existing_host->second->priority(host->priority());
        // todo
        // host的priotiry变动需要更新信息到'hosts_added_to_current_priority'队列中
        hosts_added_to_current_priority.emplace_back(existing_host->second);
      }

      // 讲修改过信息的host push到最终的结果清单中
      final_hosts.push_back(existing_host->second);
    } else {
      // 在原有的host列表里找不到的host需要新push到队列'new_hosts_for_current_priority'
      new_hosts_for_current_priority.emplace(host->address()->asString());
      if (host->weight() > max_host_weight) {
        max_host_weight = host->weight();
      }

      // todo health_checker是什么
      if (health_checker_ != nullptr) {
        host->healthFlagSet(Host::HealthFlag::FAILED_ACTIVE_HC);

        // If we want to exclude hosts until they have been health checked, mark them with
        // a flag to indicate that they have not been health checked yet.
        if (info_->warmHosts()) {
          host->healthFlagSet(Host::HealthFlag::PENDING_ACTIVE_HC);
        }
      }
      // 新增的host需要push到最终结果里
      final_hosts.push_back(host);
      // todo
      hosts_added_to_current_priority.push_back(host);
    }
  }

  // 比对新host列表与旧host列表,删掉新host列表里没有的host
  auto erase_from =
      std::remove_if(current_priority_hosts.begin(), current_priority_hosts.end(),
                     [&existing_hosts_for_current_priority](const HostSharedPtr& p) {
                       auto existing_itr =
                           existing_hosts_for_current_priority.find(p->address()->asString());

                       if (existing_itr != existing_hosts_for_current_priority.end()) {
                         existing_hosts_for_current_priority.erase(existing_itr);
                         return true;
                       }

                       return false;
                     });
  current_priority_hosts.erase(erase_from, current_priority_hosts.end());

  // todo
  if (!existing_hosts_for_current_priority.empty()) {
    hosts_changed = true;
  }

  // TODO 
  // 应该是一种特殊情况,一些新列表里没有提及,但旧列表里有的host,可能需要更具情况决定删不删除
  const bool dont_remove_healthy_hosts =
      health_checker_ != nullptr && !info()->drainConnectionsOnHostRemoval();
  if (!current_priority_hosts.empty() && dont_remove_healthy_hosts) {
    erase_from = std::remove_if(
        current_priority_hosts.begin(), current_priority_hosts.end(),
        [&all_new_hosts, &new_hosts_for_current_priority,
         &hosts_with_updated_locality_for_current_priority, &final_hosts,
         &max_host_weight](const HostSharedPtr& p) {
          // This host has already been added as a new host in the
          // new_hosts_for_current_priority. Return false here to make sure that host
          // reference with older locality gets cleaned up from the priority.
          if (hosts_with_updated_locality_for_current_priority.contains(p->address()->asString())) {
            return false;
          }

          if (all_new_hosts.contains(p->address()->asString()) &&
              !new_hosts_for_current_priority.contains(p->address()->asString())) {
            // If the address is being completely deleted from this priority, but is
            // referenced from another priority, then we assume that the other
            // priority will perform an in-place update to re-use the existing Host.
            // We should therefore not mark it as PENDING_DYNAMIC_REMOVAL, but
            // instead remove it immediately from this priority.
            // Example: health check address changed and priority also changed
            return false;
          }

          if (!(p->healthFlagGet(Host::HealthFlag::FAILED_ACTIVE_HC) ||
                p->healthFlagGet(Host::HealthFlag::FAILED_EDS_HEALTH))) {
            if (p->weight() > max_host_weight) {
              max_host_weight = p->weight();
            }

            final_hosts.push_back(p);
            p->healthFlagSet(Host::HealthFlag::PENDING_DYNAMIC_REMOVAL);
            return true;
          }
          return false;
        });
    current_priority_hosts.erase(erase_from, current_priority_hosts.end());
  }


  // 将这次操作移除的host信息整理放到传入的一个参数中
  if (!hosts_added_to_current_priority.empty() || !current_priority_hosts.empty()) {
    hosts_removed_from_current_priority = std::move(current_priority_hosts);
    hosts_changed = true;
  }

  // 将最终结果复制到原来的current_priority_hosts 变量中
  current_priority_hosts = std::move(final_hosts);
  return hosts_changed;
}
```