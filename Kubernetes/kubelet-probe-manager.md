# 版本
release-1.27 

# 序
probeManager负责管理pod的每个container的probe,并且在根据配置检测到container状态变化时更新probe状态
当前一个container的probe有:
1. livenessProbe
2. readinessProbe
3. startupProbe

# 源码 
## probeManager todo
```go
// pkg/kubelet/prober/prober_manager.go
```
## 注册一个pod的probe
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) SyncPod(_ context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
    // ...
    // 注册pod的probe管理
	kl.probeManager.AddPod(pod)
    // ...
}
```
```go
// pkg/kubelet/prober/prober_manager.go
func (m *manager) AddPod(pod *v1.Pod) {
	m.workerLock.Lock()
	defer m.workerLock.Unlock()

	key := probeKey{podUID: pod.UID}
    // 遍历pod的每个container
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name
        // startupProbe
		if c.StartupProbe != nil {
			key.probeType = startup
			// ...
            // 每个pod 的每个container 的每个probe都有个key, key相同的probe只会注册一次
			w := newWorker(m, startup, pod, c)
			m.workers[key] = w
            // 新开用一个goroutine跑probe的探测代码
			go w.run()
		}
        // readinessProbe
		if c.ReadinessProbe != nil {
			key.probeType = readiness
			// ...
			w := newWorker(m, readiness, pod, c)
			m.workers[key] = w
			go w.run()
		}
        // livenessProbe
		if c.LivenessProbe != nil {
			key.probeType = liveness
			// ...
			w := newWorker(m, liveness, pod, c)
			m.workers[key] = w
			go w.run()
		}
	}
}
```
```go
// pkg/kubelet/prober/worker.go
func (w *worker) run() {
	ctx := context.Background()
	probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second
    // ...

	probeTicker := time.NewTicker(probeTickerPeriod)
    // ...

probeLoop:
    // 无限循环doProbe,直到遇到stopCh信号, todo 这个信号谁发的?
	for w.doProbe(ctx) {
		// Wait for next probe tick.
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
		case <-w.manualTriggerCh:
			// continue
		}
	}
}
```
```go
// pkg/kubelet/prober/worker.go
func (w *worker) doProbe(ctx context.Context) (keepGoing bool) {
	// ...
	// 根据配置的规则执行probe逻辑
	result, err := w.probeManager.prober.probe(ctx, w.probeType, w.pod, status, w.container, w.containerID)
	if err != nil {
		// Prober error, throw away the result.
		return true
	}

	// ...
    // 没有触发阈值,直接返回(等待下一次循环)
	if (result == results.Failure && w.resultRun < int(w.spec.FailureThreshold)) ||
		(result == results.Success && w.resultRun < int(w.spec.SuccessThreshold)) {
		// Success or failure is below threshold - leave the probe state unchanged.
		return true
	}

    // 到这里代表probe出现了状况,需要更新probe状态
    // todo
	w.resultsManager.Set(w.containerID, result, w.pod)
    // liveness,starup probe失败的情况下,container需要重启, 将onHold置换成true,避免下一次循环再次执行probe
	if (w.probeType == liveness || w.probeType == startup) && result == results.Failure {
		// The container fails a liveness/startup check, it will need to be restarted.
		// Stop probing until we see a new container ID. This is to reduce the
		// chance of hitting #21751, where running `docker exec` when a
		// container is being stopped may lead to corrupted container state.
		w.onHold = true
		w.resultRun = 0
	}

	return true
}
```
