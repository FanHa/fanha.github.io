# 版本
release-1.27 

# 序
kubelet

# 源码
## 永久事件轮询处理
通过`syncLoopIteration`方法轮询各个`chan`,作出响应处理
- configCh Pod的增删改 事件处理
- plegCh pod的生命周期变化事件
- syncCh 周期性调用同步pod状态
- kl.livenessManager.Updates todo
- housekeepingCh 周期性做清理
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// ...
		switch u.Op {
		case kubetypes.ADD:
			// ...
            // 新增pod
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			// ...
            // 更新pod
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			// ...
            // 移除pod
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			// todo
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			// todo
			handler.HandlePodUpdates(u.Pods)
			// ...
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}
        // todo
		kl.sourcesReady.AddSource(u.Source)

	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// 更新pod 状态
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
                // todo
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} 
		}
        // ContainerDied 事件清理pod
		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
                // todo
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// 周期性同步pod状态
		podsToSync := kl.getPodsToSync()
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
	case <-housekeepingCh:
		// ...
			if err := handler.HandlePodCleanups(ctx); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
			
		// ...
	}
	return true
}
```
### HandlePodAdditions 
处理pod新增event, 一个event可能会新增多个pod
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	// ...
	for _, pod := range pods {
        // 获取当前manager管理的pod信息
		existingPods := kl.podManager.GetPods()
		// podManager增加pod信息
		kl.podManager.AddPod(pod)

		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			activePods := kl.filterOutInactivePods(existingPods)

			//...
				// 判断在当前node上创建这个pod能不能被接受
                // todo 判断什么
				if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
					kl.rejectPod(pod, reason, message)
					continue
				}
			// ...
		}
		// ...
        // 调用podWorker 实际在本地创建pod
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)

	}
}
```