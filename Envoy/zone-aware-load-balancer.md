
- [版本](#版本)
- [序](#序)
  - [与`ZoneAwareLoadBalancerBase`有关的的lb关系](#与zoneawareloadbalancerbase有关的的lb关系)
  - [host保存结构](#host保存结构)
    - [PrioritySet todo](#priorityset-todo)
    - [HostSet todo](#hostset-todo)
    - [HostsPerlocality todo](#hostsperlocality-todo)
    - [HostVector todo](#hostvector-todo)
- [源码逻辑](#源码逻辑)
  - [接口](#接口)
  - [实现](#实现)
    - [RoundRobinLoadBalancer 创建](#roundrobinloadbalancer-创建)
    - [ChooseHost](#choosehost)
      - [hostSourceToUse](#hostsourcetouse)
    - [host\_set的更新 todo](#host_set的更新-todo)


# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
ZoneAwareLoadBalancer 使envoy可以把流量优先转发到同region同zone的服务,减少网络损耗

## 与`ZoneAwareLoadBalancerBase`有关的的lb关系
- RandomLoadBalancer
- RoundRobinLoadBalancerBase
- LeastRequestLoadBalancer

![ZoneAwareLoadBalancerBase](resource/ZoneAwareLoadBalancerBase.drawio.svg)

## host保存结构
- 一个cluster拥有了一个`PrioriySet`
- 一个`PrioritySet`拥有若干个优先级不同的`HostSet`
- 一个`HostSet`拥有不同种类的`HostVector`或`HostsPerLocality`
  - SourceType::AllHosts
  - SourceType::HealthyHosts
  - SourceType::DegradedHosts
  - SourceType::LocalityHealthyHosts(`HostsPerLocality`)
  - SourceType::LocalityDegradedHosts(`HostsPerLocality`)
- `HostsPerLocality`中有若干个优先级不同的`HostVector`
- `HostVector`可以认为是一个host数组

![PriorityHost](resource/PriorityHost.drawio.svg)
### PrioritySet todo
### HostSet todo
### HostsPerlocality todo
### HostVector todo

# 源码逻辑
以 `RoundRobinLoadBalancerBase`为例子
## 接口
```cpp
// source/common/upstream/load_balancer_impl.h
class RoundRobinLoadBalancer : public EdfLoadBalancerBase {
public:
  // 创建‘RoundRobinLoadBalancer’
  RoundRobinLoadBalancer(
      const PrioritySet& priority_set, const PrioritySet* local_priority_set, ClusterStats& stats,
      Runtime::Loader& runtime, Random::RandomGenerator& random,
      const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config,
      const absl::optional<envoy::config::cluster::v3::Cluster::RoundRobinLbConfig>
          round_robin_config,
      TimeSource& time_source)
      : EdfLoadBalancerBase(
            priority_set, local_priority_set, stats, runtime, random, common_config,
            (round_robin_config.has_value() && round_robin_config.value().has_slow_start_config())
                ? absl::optional<envoy::config::cluster::v3::Cluster::SlowStartConfig>(
                      round_robin_config.value().slow_start_config())
                : absl::nullopt,
            time_source) {
    initialize();
  }

private:
  // 刷新一个存放host的容器信息
  void refreshHostSource(const HostsSource& source) override {
    rr_indexes_.insert({source, seed_});
    peekahead_index_ = 0;
  }
  double hostWeight(const Host& host) override {
    if (!noHostsAreInSlowStart()) {
      return applySlowStartFactor(host.weight(), host);
    }
    return host.weight();
  }

  HostConstSharedPtr unweightedHostPeek(const HostVector& hosts_to_use,
                                        const HostsSource& source) override {
    auto i = rr_indexes_.find(source);
    if (i == rr_indexes_.end()) {
      return nullptr;
    }
    return hosts_to_use[(i->second + (peekahead_index_)++) % hosts_to_use.size()];
  }

  HostConstSharedPtr unweightedHostPick(const HostVector& hosts_to_use,
                                        const HostsSource& source) override {
    if (peekahead_index_ > 0) {
      --peekahead_index_;
    }
    // To avoid storing the RR index in the base class, we end up using a second map here with
    // host source as the key. This means that each LB decision will require two map lookups in
    // the unweighted case. We might consider trying to optimize this in the future.
    ASSERT(rr_indexes_.find(source) != rr_indexes_.end());
    return hosts_to_use[rr_indexes_[source]++ % hosts_to_use.size()];
  }

  uint64_t peekahead_index_{};
  absl::node_hash_map<HostsSource, uint64_t, HostsSourceHash> rr_indexes_;
};
```
## 实现
### RoundRobinLoadBalancer 创建
```cpp
  // source/common/upstream/load_balancer_impl.cc
  RoundRobinLoadBalancer(
      const PrioritySet& priority_set, const PrioritySet* local_priority_set, ClusterStats& stats,
      Runtime::Loader& runtime, Random::RandomGenerator& random,
      const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config,
      const absl::optional<envoy::config::cluster::v3::Cluster::RoundRobinLbConfig>
          round_robin_config,
      TimeSource& time_source)
      : EdfLoadBalancerBase( // 调用父类’EdfLoadBalancerBase‘的创建 todo EdfLoadBalancerBase
            priority_set, local_priority_set, stats, runtime, random, common_config,
            (round_robin_config.has_value() && round_robin_config.value().has_slow_start_config())
                ? absl::optional<envoy::config::cluster::v3::Cluster::SlowStartConfig>(
                      round_robin_config.value().slow_start_config())
                : absl::nullopt,
            time_source) {
    // 初始化当前‘cluster’的所有优先级的host设置 #ref initialize
    initialize();
  }

// 初始化所有优先级的host设置
void EdfLoadBalancerBase::initialize() {
  // 遍历‘cluster’的每一个优先级
  for (uint32_t priority = 0; priority < priority_set_.hostSetsPerPriority().size(); ++priority) {
    // #ref refresh
    refresh(priority);
  }
}

// 刷新一个优先级(入参)的信息 
void EdfLoadBalancerBase::refresh(uint32_t priority) {
  // 创建一个‘add_hosts_source’函数将若干host放入指定的容器
  // 入参 source 是目标容器
  // 入参 hosts 是要放入目标的hosts们
  const auto add_hosts_source = [this](HostsSource source, const HostVector& hosts) {
    // 直接清空原有的source
    auto& scheduler = scheduler_[source] = Scheduler{};
    // 更新这个source容器的host信息, 这个是由继承了‘EdfLoadBalancerBase’类的‘RoundRobinLoadBalancer’来实现, #ref refreshHostSource
    refreshHostSource(source);
    // todo
    if (isSlowStartEnabled()) {
      recalculateHostsInSlowStart(hosts);
    }

    // 没有配置百分比分配流量时,直接跳过edf机制创建,因为有更省资源的方式实现 todo
    if (hostWeightsAreEqual(hosts) && noHostsAreInSlowStart()) {
      // Skip edf creation.
      return;
    }
    // ...
  };
  // 根据优先级‘priority’取出cluster在这个优先级的所有host信息
  const auto& host_set = priority_set_.hostSetsPerPriority()[priority];
  // 通过‘HostsSource’方法找到相应的容器,然后根据host类型放到当前优先级的相应容器
  add_hosts_source(HostsSource(priority, HostsSource::SourceType::AllHosts), host_set->hosts());
  add_hosts_source(HostsSource(priority, HostsSource::SourceType::HealthyHosts),
                   host_set->healthyHosts());
  add_hosts_source(HostsSource(priority, HostsSource::SourceType::DegradedHosts),
                   host_set->degradedHosts());
  // 遍历每一个优先级的‘healthyHostsPerLocality’里的host,把ta们按顺序放到当前优先级的‘LocalityHealthyHosts’容器里
  for (uint32_t locality_index = 0;
       locality_index < host_set->healthyHostsPerLocality().get().size(); ++locality_index) {
    add_hosts_source(
        HostsSource(priority, HostsSource::SourceType::LocalityHealthyHosts, locality_index),
        host_set->healthyHostsPerLocality().get()[locality_index]);
  }
  for (uint32_t locality_index = 0;
       locality_index < host_set->degradedHostsPerLocality().get().size(); ++locality_index) {
    add_hosts_source(
        HostsSource(priority, HostsSource::SourceType::LocalityDegradedHosts, locality_index),
        host_set->degradedHostsPerLocality().get()[locality_index]);
  }
}


void refreshHostSource(const HostsSource& source) override {
    // insert() is used here on purpose so that we don't overwrite the index if the host source
    // already exists. Note that host sources will never be removed, but given how uncommon this
    // is it probably doesn't matter.
    rr_indexes_.insert({source, seed_});
    // If the list of hosts changes, the order of picks change. Discard the
    // index.
    peekahead_index_ = 0;
  }

```
### ChooseHost
lb通过`chooseHost`方法根据`context`挑选出目标`host`
```cpp
// source/common/upstream/load_balancer_impl.cc
HostConstSharedPtr ZoneAwareLoadBalancerBase::chooseHost(LoadBalancerContext* context) {
  // todo
  HostConstSharedPtr host = LoadBalancerContextBase::selectOverrideHost(
      cross_priority_host_map_.get(), override_host_status_, context);
  if (host != nullptr) {
    return host;
  }

  const size_t max_attempts = context ? context->hostSelectionRetryCount() + 1 : 1;
  for (size_t i = 0; i < max_attempts; ++i) {
    // #ref chooseHostOnce
    host = chooseHostOnce(context);

    // If host selection failed or the host is accepted by the filter, return.
    // Otherwise, try again.
    // Note: in the future we might want to allow retrying when chooseHostOnce returns nullptr.
    if (!host || !context || !context->shouldSelectAnotherHost(*host)) {
      return host;
    }
  }

  // If we didn't find anything, return the last host.
  return host;
}

HostConstSharedPtr EdfLoadBalancerBase::chooseHostOnce(LoadBalancerContext* context) {
  // 调用hostSourceToUse方法根据context得到一个‘HostSource’结构实例,后面需要根据这个结构实例的信息确认从哪个具体HostVector里获取host
  // #ref hostSourceToUse todo
  const absl::optional<HostsSource> hosts_source = hostSourceToUse(context, random(false));
  // ...
  // 根据‘hosts_source’信息从’schedule_‘找到相应‘scheduler’ todo
  auto scheduler_it = scheduler_.find(*hosts_source);
  auto& scheduler = scheduler_it->second;

  if (scheduler.edf_ != nullptr) {
    // 配置了加权
    auto host = scheduler.edf_->pickAndAdd([this](const Host& host) { return hostWeight(host); });
    return host;
  } else { // 没有配置加权时
    // 根据‘hosts_source’信息,找到最终要使用的‘HostVector’ todo
    const HostVector& hosts_to_use = hostSourceToHosts(*hosts_source);
    if (hosts_to_use.empty()) {
      return nullptr;
    }
    // 从‘HostVector’中取出一个最终的host todo
    return unweightedHostPick(hosts_to_use, *hosts_source);
  }
}
```
#### hostSourceToUse
```cpp
// source/common/upstream/load_balancer_impl.cc
absl::optional<ZoneAwareLoadBalancerBase::HostsSource>
ZoneAwareLoadBalancerBase::hostSourceToUse(LoadBalancerContext* context, uint64_t hash) const {
  // 按优先级选出prioritySet里的HostSet,以及set的健康状态
  auto host_set_and_source = chooseHostSet(context, hash);

  // The second argument tells us which availability we should target from the selected host set.
  const auto host_availability = host_set_and_source.second;
  auto& host_set = host_set_and_source.first;

  // 构造要返回的‘HostSource’实例
  HostsSource hosts_source;
  hosts_source.priority_ = host_set.priority(); // 填入priority信息

  // ...

  // 判断是否存在locality相关的设置
  // 最终返回locality是 'HostsPerLocality'的优先级index
  absl::optional<uint32_t> locality;
  if (host_availability == HostAvailability::Degraded) {
    locality = host_set.chooseDegradedLocality();
  } else {
    locality = host_set.chooseHealthyLocality();
  }
  // 如果存在locality设置,需要对hosts_source做些调整
  if (locality.has_value()) {
    // 转换SourceType类型
    // HostAvailability::Healthy ==》 HostsSource::SourceType::LocalityHealthyHosts
    // HostAvailability::Degraded ==》 HostsSource::SourceType::LocalityDegradedHosts
    auto source_type = localitySourceType(host_availability);
    if (!source_type) {
      return absl::nullopt;
    }
    // 设定source_type_ 和 locality_index_ 属性并返回
    hosts_source.source_type_ = source_type.value();
    hosts_source.locality_index_ = locality.value();
    return hosts_source;
  }
  // 其他非locality的情况
  // ...
}
```

### host_set的更新 todo
