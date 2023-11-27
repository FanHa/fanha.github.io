# 版本
https://github.com/chaos-mesh/chaos-mesh
2.6

# 序
stresschaos 用来给集群内部的pod 注入资源(cpu, 内存)故障

# 示意图
![stresschaos](stresschaos.drawio.svg)

# 源码解析
```go
// pkg/chaosdaemon/stress_server_linux.go
func (s *DaemonServer) ExecCPUStressors(ctx context.Context,
	req *pb.ExecStressRequest) (*bpm.Process, error) {
	// ...
    // 从容器ID获取PID
	pid, err := s.crClient.GetPidFromContainerID(ctx, req.Target)
	if err != nil {
		return nil, err
	}

    // 获取用于附加到 PID 的 cgroup(/proc/<pid>/cgroup)
	attachCGroup, err := cgroups.GetAttacherForPID(int(pid))
	if err != nil {
		return nil, err
	}

    // 构建用于执行 stress-ng 的进程
	processBuilder := bpm.DefaultProcessBuilder("stress-ng", strings.Fields(req.CpuStressors)...).
		EnablePause()
	if req.EnterNS {
		processBuilder = processBuilder.SetNS(pid, bpm.PidNS)
	}
	cmd := processBuilder.Build(ctx)

    // 启动进程
	proc, err := s.backgroundProcessManager.StartProcess(ctx, cmd)
	if err != nil {
		return nil, err
	}
    // ...
    // 将进程附加到 cgroup
	if err = attachCGroup.AttachProcess(proc.Pair.Pid); err != nil {
		if kerr := cmd.Process.Kill(); kerr != nil {
			log.Error(kerr, "kill stress-ng failed", "request", req)
		}
		return nil, err
	}

	// ...

	return proc, nil
}```