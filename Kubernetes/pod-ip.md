# 版本
release-1.20

# 序
node上的kubelet 监听到自己所在的node的pod,在当前node上创建pod时向cni插件申请ip
TODO 怎么把这个ip更新到k8s api server上??

# 示意图
![pod-ip-apply](pod-ip.apply.drawio.svg)

# 源码
## RunPodSandbox
```go
// pkg/kubelet/dockershim/docker_sandbox.go
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {
	config := r.GetConfig()

	// ...
    // 调用network插件的 SetUpPod 方法,为pod申请ip
	err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations, networkOptions)
	if err != nil {
		// ...
	}

	return resp, nil
}
```
