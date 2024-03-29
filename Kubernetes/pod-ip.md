# 版本
release-1.20

# 序
node上的kubelet 监听到自己所在的node的pod,在当前node上创建pod时向cni插件申请ip
创建好pod后,把pod的ip更新到k8s api server上

# 三份状态
一个pod有3份状态
1. api server 上的状态
2. kubelet上的statusManager上的状态
3. 容器运行时上的状态

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
