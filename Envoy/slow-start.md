- [版本](#版本)
- [序](#序)
- [原理](#原理)
- [代码](#代码)
  - [hostWeight(const Host\& host) 计算一个host在lb中的加权值](#hostweightconst-host-host-计算一个host在lb中的加权值)
    - [EdfLoadBalancerBase::applySlowStartFactor(double host\_weight, const Host\& host) 在原有加权值上加上slow-start的修正](#edfloadbalancerbaseapplyslowstartfactordouble-host_weight-const-host-host-在原有加权值上加上slow-start的修正)

# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
slow-start配置可以让新加入的endpoint不立刻得到全部流量,设定一个窗口时间,流量慢慢涨上来(渐进式流量)

# 原理
envoy lb 通过`ScheduleEdf`的`pickAndAdd`方法挑出下个请求要转发的`endpoint`,然后又按照配置的生成一个给这个`endpoint`生成一个新的加权值重新加入到`ScheduleEdf`中,`slow-start`就是通过在窗口期调整生成新加权值时的数值来实现`渐进式流量`

# 代码
```cpp
    // edf 调用pickAndAdd方法挑出一个host,然后调用‘hostWeight’重新计算该host的加权值加回edf
    scheduler.edf_->pickAndAdd([this](const Host& host) { return hostWeight(host); });
```
## hostWeight(const Host& host) 计算一个host在lb中的加权值
```cpp
// source/common/upstream/load_balancer_impl.h
double hostWeight(const Host& host) override {
    if (!noHostsAreInSlowStart()) {
      // 在原有weight值的基础上计算slow-start加权 #ref  EdfLoadBalancerBase::applySlowStartFactor(double host_weight, const Host& host)
      return applySlowStartFactor(host.weight(), host);
    }
    return host.weight();
  }
```
###  EdfLoadBalancerBase::applySlowStartFactor(double host_weight, const Host& host) 在原有加权值上加上slow-start的修正
```cpp
// source/common/upstream/load_balancer_impl.cc
double EdfLoadBalancerBase::applySlowStartFactor(double host_weight, const Host& host) {
  // 算出host已经创建了多长时间
  auto host_create_duration = std::chrono::duration_cast<std::chrono::milliseconds>(
      time_source_.monotonicTime() - host.creationTime());
  if (host_create_duration < slow_start_window_ && // 已创建时间 < slow-start窗口时间说明host还在窗口期,需要继续使用slow-start修正加权值
      host.health() == Upstream::Host::Health::Healthy) { 
    // 流量渐进增长的曲率设置(由快到慢,由慢到快,线性增长)
    aggression_ = aggression_runtime_ != absl::nullopt ? aggression_runtime_.value().value() : 1.0;
    //... 
    // time_factor是已创建时间与窗口时间的比例
    auto time_factor = static_cast<double>(std::max(std::chrono::milliseconds(1).count(),
                                                    host_create_duration.count())) /
                       slow_start_window_.count();

    // 通过当前‘time_factor’与前面计算得到的‘aggression_’得到一个小于‘100%’的值,乘以原来的host_weight就得到了slow-start修正后的加权值
    // applyAggressionFactor见下方
    return host_weight *
           std::max(applyAggressionFactor(time_factor), slow_start_min_weight_percent_);
  } else {
    return host_weight;
  }
}

// 通过时间比例与曲率值算出修正因子
double EdfLoadBalancerBase::applyAggressionFactor(double time_factor) {
  if (aggression_ == 1.0 || time_factor == 1.0) {
    return time_factor;
  } else {
    return std::pow(time_factor, 1.0 / aggression_);
  }
}

```