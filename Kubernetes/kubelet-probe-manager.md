# 版本
release-1.27 

# 序
probe manager 管理一个pod(或者container?? todo)的所有probe,k8s pod资源的container中包含livenessProbe,readinessProbe, startupProbe,probeManager负责管理这些probe,并且在pod状态变化时更新probe状态

# 源码 todo
```go
// pkg/kubelet/prober/prober_manager.go
```