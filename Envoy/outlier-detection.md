# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
outlier_detection 用来配置envoy 的 cluster里的host的检测机制,配置在什么情况下,cluster里的某个host应该被认为不健康,下次有请求时不转给它
>> 注: istio 的 DestinationRule的Subset里的outlier配置,最终也是变成了envoy里的cluster outlier配置

# 核心代码逻辑
## route 向outlierDetector push结果的逻辑 todo

## detector 初始化
```cpp
// source/common/upstream/outlier_detection_impl.cc
td::shared_ptr<DetectorImpl> DetectorImpl::create(
    const Cluster& cluster, const envoy::config::cluster::v3::OutlierDetection& config,
    Event::Dispatcher& dispatcher, Runtime::Loader& runtime, TimeSource& time_source,
    EventLoggerSharedPtr event_logger, Random::RandomGenerator& random) {
  std::shared_ptr<DetectorImpl> detector(
      new DetectorImpl(cluster, config, dispatcher, runtime, time_source, event_logger, random));

  if (detector->config().maxEjectionTimeMs() < detector->config().baseEjectionTimeMs()) {
    throw EnvoyException(
        "outlier detector's max_ejection_time cannot be smaller than base_ejection_time");
  }

  detector->initialize(cluster);

  return detector;
}
void DetectorImpl::initialize(const Cluster& cluster) {
  for (auto& host_set : cluster.prioritySet().hostSetsPerPriority()) {
    for (const HostSharedPtr& host : host_set->hosts()) {
      addHostMonitor(host);
    }
  }
  member_update_cb_ = cluster.prioritySet().addMemberUpdateCb(
      [this](const HostVector& hosts_added, const HostVector& hosts_removed) -> void {
        for (const HostSharedPtr& host : hosts_added) {
          addHostMonitor(host);
        }

        for (const HostSharedPtr& host : hosts_removed) {
          ASSERT(host_monitors_.count(host) == 1);
          if (host->healthFlagGet(Host::HealthFlag::FAILED_OUTLIER_CHECK)) {
            ASSERT(ejections_active_helper_.value() > 0);
            ejections_active_helper_.dec();
          }

          host_monitors_.erase(host);
        }
      });

  armIntervalTimer();
}
```
## ejectHost 将一个host暂时驱逐出cluster
cluster触发需要标记一个host不可用时调用 ejctHost方法,传入要驱逐的host和触发的驱逐规则类型
```cpp
// source/common/upstream/outlier_detection_impl.cc
void DetectorImpl::ejectHost(HostSharedPtr host,
                             envoy::data::cluster::v3::OutlierEjectionType type) {
  uint64_t max_ejection_percent = std::min<uint64_t>(
      100, runtime_.snapshot().getInteger(MaxEjectionPercentRuntime, config_.maxEjectionPercent()));
  double ejected_percent = 100.0 * ejections_active_helper_.value() / host_monitors_.size();
  // 如果配置了最大可驱逐host的比例,只有当前已驱逐的host的比例低于此值才会真正触发驱逐
  if (ejected_percent < max_ejection_percent) {
    // ...
    if (enforceEjection(type)) {
      // ...
      // 调用当前host列表里的host_monitor触发eject方法,驱逐该host todo #ref eject
      host_monitors_[host]->eject(time_source_.monotonicTime());
      // ...

      // 调用注册的回调 todo #ref runCallbacks
      runCallbacks(host);
      // ...
    } else {
      // ...
    }
  } else {
    // ...
  }
}
```