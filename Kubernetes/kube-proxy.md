# 版本
release-1.27 

# 序
kubeproxy 作用

# 源码
## Run
```go
// cmd/kube-proxy/app/server.go
func (s *ProxyServer) Run() error {
	// ...
	labelSelector := labels.NewSelector()
	labelSelector = labelSelector.Add(*noProxyName, *noHeadlessEndpoints)

	// Make informers that filter out objects that want a non-default service proxy.
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))

	// 注册监听k8s里的service资源
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)

    // 注册监听k8s里的endpointslice资源
	endpointSliceConfig := config.NewEndpointSliceConfig(informerFactory.Discovery().V1().EndpointSlices(), s.ConfigSyncPeriod)
	endpointSliceConfig.RegisterEventHandler(s.Proxier)
	go endpointSliceConfig.Run(wait.NeverStop)

	// This has to start after the calls to NewServiceConfig because that
	// function must configure its shared informer event handlers first.
	informerFactory.Start(wait.NeverStop)

	// Make an informer that selects for our nodename.
	currentNodeInformerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.FieldSelector = fields.OneTermEqualSelector("metadata.name", s.NodeRef.Name).String()
		}))
	nodeConfig := config.NewNodeConfig(currentNodeInformerFactory.Core().V1().Nodes(), s.ConfigSyncPeriod)
	// https://issues.k8s.io/111321
	if s.localDetectorMode == kubeproxyconfig.LocalModeNodeCIDR {
		nodeConfig.RegisterEventHandler(proxy.NewNodePodCIDRHandler(s.podCIDRs))
	}
	nodeConfig.RegisterEventHandler(s.Proxier)

	go nodeConfig.Run(wait.NeverStop)

	// This has to start after the calls to NewNodeConfig because that must
	// configure the shared informer event handler first.
	currentNodeInformerFactory.Start(wait.NeverStop)

	// Birth Cry after the birth is successful
	s.birthCry()

	go s.Proxier.SyncLoop()

	return <-errCh
}
```

### Service事件方法的注册与回调
#### 注册
```go
// pkg/proxy/config/config.go
func (c *ServiceConfig) RegisterEventHandler(handler ServiceHandler) {
	c.eventHandlers = append(c.eventHandlers, handler)
}
```
#### 回调
当有service资源的变动事件发生时会回调下面的方法 add,update,delete
```go
// pkg/proxy/config/config.go
func (c *ServiceConfig) handleAddService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
		return
	}
	for i := range c.eventHandlers {
		klog.V(4).InfoS("Calling handler.OnServiceAdd")
		c.eventHandlers[i].OnServiceAdd(service)
	}
}

func (c *ServiceConfig) handleUpdateService(oldObj, newObj interface{}) {
	oldService, ok := oldObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", oldObj))
		return
	}
	service, ok := newObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", newObj))
		return
	}
	for i := range c.eventHandlers {
		klog.V(4).InfoS("Calling handler.OnServiceUpdate")
		c.eventHandlers[i].OnServiceUpdate(oldService, service)
	}
}

func (c *ServiceConfig) handleDeleteService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
			return
		}
		if service, ok = tombstone.Obj.(*v1.Service); !ok {
			utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
			return
		}
	}
	for i := range c.eventHandlers {
		klog.V(4).InfoS("Calling handler.OnServiceDelete")
		c.eventHandlers[i].OnServiceDelete(service)
	}
}
```
#### 实际proxy实现的回调方法
add,update,delete最终都调用`proxier.serviceChanges.Update`方法
```go
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
	proxier.OnServiceUpdate(nil, service)
}

func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
    // proxier.serviceChanges.Update 在本地生成一条服务改变记录(但实际并未改变啥) #ref proxier.serviceChanges.Update
    // proxier.isInitialized() 判断proxy是否已经就绪 todo proxier.isInitialized()
	if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
        // 只有在有变动生成且proxyier已经就绪时才会把服务下发到系统内核
		proxier.Sync()
	}
}

func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
	proxier.OnServiceUpdate(service, nil)
}

```
##### proxier.serviceChanges.Update
```go
// pkg/proxy/service.go
func (sct *ServiceChangeTracker) Update(previous, current *v1.Service) bool {
	// ...

    // 删除操作时仍然需要以前服务的一些信息(名字,命名空间)
	svc := current
	if svc == nil {
		svc = previous
	}
	metrics.ServiceChangesTotal.Inc()
    // 获取变更后的服务名
	namespacedName := types.NamespacedName{Namespace: svc.Namespace, Name: svc.Name}

	sct.lock.Lock()
	defer sct.lock.Unlock()

    // 判断是否有同一个服务还没有完成的‘serviceChange’
	change, exists := sct.items[namespacedName]
	if !exists { // 没有则新建一个‘serviceChange’
		change = &serviceChange{}
        // 将原来的服务的内容格式化
		change.previous = sct.serviceToServiceMap(previous)
		sct.items[namespacedName] = change
	}
    // 将新服务的内容格式化
	change.current = sct.serviceToServiceMap(current)
	// 没有变化则直接删除掉此次变更
	if reflect.DeepEqual(change.previous, change.current) {
		delete(sct.items, namespacedName)
	} else {
		klog.V(4).InfoS("Service updated ports", "service", klog.KObj(svc), "portCount", len(change.current))
	}
	metrics.ServiceChangesPending.Set(float64(len(sct.items)))
	return len(sct.items) > 0
}

```
##### SyncLoop
```go
// pkg/proxy/iptables/proxier.go
func (proxier *Proxier) SyncLoop() {
	// ...
	proxier.syncRunner.Loop(wait.NeverStop)
}

```
```go
// pkg/util/async/bounded_frequency_runner.go
func (bfr *BoundedFrequencyRunner) Loop(stop <-chan struct{}) {
	klog.V(3).Infof("%s Loop running", bfr.name)
	bfr.timer.Reset(bfr.maxInterval)
	for {
		select {
		case <-stop:
			bfr.stop()
			klog.V(3).Infof("%s Loop stopping", bfr.name)
			return
		case <-bfr.timer.C():
			bfr.tryRun()
		case <-bfr.run:
			bfr.tryRun()
		case <-bfr.retry:
			bfr.doRetry()
		}
	}
}
func (bfr *BoundedFrequencyRunner) tryRun() {
	bfr.mu.Lock()
	defer bfr.mu.Unlock()

	if bfr.limiter.TryAccept() {
		// 执行回调,比如iptables 就是在这里把服务规则解析并根据解析结果设置转发规则给内核
        // 在iptable中就是 
		bfr.fn()
		bfr.lastRun = bfr.timer.Now()
		bfr.timer.Stop()
		bfr.timer.Reset(bfr.maxInterval)
		klog.V(3).Infof("%s: ran, next possible in %v, periodic in %v", bfr.name, bfr.minInterval, bfr.maxInterval)
		return
	}

	// It can't run right now, figure out when it can run next.
	elapsed := bfr.timer.Since(bfr.lastRun)   // how long since last run
	nextPossible := bfr.minInterval - elapsed // time to next possible run
	nextScheduled := bfr.timer.Remaining()    // time to next scheduled run
	klog.V(4).Infof("%s: %v since last run, possible in %v, scheduled in %v", bfr.name, elapsed, nextPossible, nextScheduled)

	// It's hard to avoid race conditions in the unit tests unless we always reset
	// the timer here, even when it's unchanged
	if nextPossible < nextScheduled {
		nextScheduled = nextPossible
	}
	bfr.timer.Stop()
	bfr.timer.Reset(nextScheduled)
}
```

### Iptable
#### iptables原理图
![avatar](resource/tables_traverse.jpeg)
无论是外部流量进入还是内部流量出去 都需要走同一套流程
k8s使用了iptables的filter表和nat表。filter表用于处理入站和出站的数据包，而nat表用于网络地址转换。
#### 同步规则到iptables的代码逻辑
```go
// pkg/proxy/iptables/proxier.go
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()

	// proxier 初始化好以前不需要同步任何规则
	if !proxier.isInitialized() {
		klog.V(2).InfoS("Not syncing iptables until Services and Endpoints have been received from master")
		return
	}


	tryPartialSync := !proxier.needFullSync && utilfeature.DefaultFeatureGate.Enabled(features.MinimizeIPTablesRestore)
	var serviceChanged, endpointsChanged sets.String
	if tryPartialSync {
		serviceChanged = proxier.serviceChanges.PendingChanges()
		endpointsChanged = proxier.endpointsChanges.PendingChanges()
	}
	serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

	// We need to detect stale connections to UDP Services so we
	// can clean dangling conntrack entries that can blackhole traffic.
	conntrackCleanupServiceIPs := serviceUpdateResult.DeletedUDPClusterIPs
	conntrackCleanupServiceNodePorts := sets.NewInt()
	// merge stale services gathered from updateEndpointsMap
	// an UDP service that changes from 0 to non-0 endpoints is considered stale.
	for _, svcPortName := range endpointUpdateResult.NewlyActiveUDPServices {
		if svcInfo, ok := proxier.svcPortMap[svcPortName]; ok {
			klog.V(4).InfoS("Newly-active UDP service may have stale conntrack entries", "servicePortName", svcPortName)
			conntrackCleanupServiceIPs.Insert(svcInfo.ClusterIP().String())
			for _, extIP := range svcInfo.ExternalIPStrings() {
				conntrackCleanupServiceIPs.Insert(extIP)
			}
			for _, lbIP := range svcInfo.LoadBalancerIPStrings() {
				conntrackCleanupServiceIPs.Insert(lbIP)
			}
			nodePort := svcInfo.NodePort()
			if svcInfo.Protocol() == v1.ProtocolUDP && nodePort != 0 {
				conntrackCleanupServiceNodePorts.Insert(nodePort)
			}
		}
	}


	success := false


	if !tryPartialSync {
        // 创建基本的chain 和跳转关系, iptablesJumpChains 和 iptablesKubeletJumpChains 见chain 跳转图
		for _, jump := range append(iptablesJumpChains, iptablesKubeletJumpChains...) {
			if _, err := proxier.iptables.EnsureChain(jump.table, jump.dstChain); err != nil {
				klog.ErrorS(err, "Failed to ensure chain exists", "table", jump.table, "chain", jump.dstChain)
				return
			}
			args := jump.extraArgs
			if jump.comment != "" {
				args = append(args, "-m", "comment", "--comment", jump.comment)
			}
			args = append(args, "-j", string(jump.dstChain))
			if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jump.table, jump.srcChain, args...); err != nil {
				klog.ErrorS(err, "Failed to ensure chain jumps", "table", jump.table, "srcChain", jump.srcChain, "dstChain", jump.dstChain)
				return
			}
		}
	}

	// 清理chain和规则的本地缓存,只用户层的缓存
	proxier.filterChains.Reset()
	proxier.filterRules.Reset()
	proxier.natChains.Reset()
	proxier.natRules.Reset()

	// 准备所有自定义chain的信息(后面直接使用来拼接iptable 语句)
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeExternalServicesChain, kubeForwardChain, kubeNodePortsChain, kubeProxyFirewallChain} {
        // filter 表
		proxier.filterChains.Write(utiliptables.MakeChainLine(chainName))
	}
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeNodePortsChain, kubePostroutingChain, kubeMarkMasqChain} {
        // nat 表
		proxier.natChains.Write(utiliptables.MakeChainLine(chainName))
	}

	// kubePostroutingChain 中设定不满足 masqueradeMark 标记时直接跳过该链(kubePostroutingChain)
    // 这个标记用来表示发出去的包需要隐藏源ip替换成出口ip
    // todo 这个标记在哪里打的
	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-m", "mark", "!", "--mark", fmt.Sprintf("%s/%s", proxier.masqueradeMark, proxier.masqueradeMark),
		"-j", "RETURN",
	)
	// 到这里说明没有返回,需要在kubePostroutingChain里继续执行规则,先清除已有的 masqueradeMark 标记
	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-j", "MARK", "--xor-mark", proxier.masqueradeMark,
	)

    // -j MASQUERADE 作用将源ip替换成出口ip
	masqRule := []string{
		"-A", string(kubePostroutingChain),
		"-m", "comment", "--comment", `"kubernetes service traffic requiring SNAT"`,
		"-j", "MASQUERADE",
	}
	proxier.natRules.Write(masqRule)

	// todo 这个chain 在流程的哪里??
	proxier.natRules.Write(
		"-A", string(kubeMarkMasqChain),
		"-j", "MARK", "--or-mark", proxier.masqueradeMark,
	)


	// Accumulate NAT chains to keep.
	activeNATChains := map[utiliptables.Chain]bool{} // use a map as a set

	// To avoid growing this slice, we arbitrarily set its size to 64,
	// there is never more than that many arguments for a single line.
	// Note that even if we go over 64, it will still be correct - it
	// is just for efficiency, not correctness.
	args := make([]string, 64)

	// Compute total number of endpoint chains across all services
	// to get a sense of how big the cluster is.
	totalEndpoints := 0
	for svcName := range proxier.svcPortMap {
		totalEndpoints += len(proxier.endpointsMap[svcName])
	}
	proxier.largeClusterMode = (totalEndpoints > largeClusterEndpointsThreshold)

	// These two variables are used to publish the sync_proxy_rules_no_endpoints_total
	// metric.
	serviceNoLocalEndpointsTotalInternal := 0
	serviceNoLocalEndpointsTotalExternal := 0

	// 遍历已知的服务列表
	for svcName, svc := range proxier.svcPortMap {
		svcInfo, ok := svc.(*servicePortInfo)
		if !ok {
			klog.ErrorS(nil, "Failed to cast serviceInfo", "serviceName", svcName)
			continue
		}
		protocol := strings.ToLower(string(svcInfo.Protocol()))
		svcPortNameString := svcInfo.nameString

		// 通过服务名找到该服务对应的endpointslice
		allEndpoints := proxier.endpointsMap[svcName]
        // 给所有endpoint分类
		clusterEndpoints, localEndpoints, allLocallyReachableEndpoints, hasEndpoints := proxy.CategorizeEndpoints(allEndpoints, svcInfo, proxier.nodeLabels)

		// Note the endpoint chains that will be used
		for _, ep := range allLocallyReachableEndpoints {
			if epInfo, ok := ep.(*endpointsInfo); ok {
				activeNATChains[epInfo.ChainName] = true
			}
		}

		// clusterPolicyChain contains the endpoints used with "Cluster" traffic policy
		clusterPolicyChain := svcInfo.clusterPolicyChainName
		usesClusterPolicyChain := len(clusterEndpoints) > 0 && svcInfo.UsesClusterEndpoints()
		if usesClusterPolicyChain {
			activeNATChains[clusterPolicyChain] = true
		}

		// localPolicyChain contains the endpoints used with "Local" traffic policy
		localPolicyChain := svcInfo.localPolicyChainName
		usesLocalPolicyChain := len(localEndpoints) > 0 && svcInfo.UsesLocalEndpoints()
		if usesLocalPolicyChain {
			activeNATChains[localPolicyChain] = true
		}

		// internalPolicyChain is the chain containing the endpoints for
		// "internal" (ClusterIP) traffic. internalTrafficChain is the chain that
		// internal traffic is routed to (which is always the same as
		// internalPolicyChain). hasInternalEndpoints is true if we should
		// generate rules pointing to internalTrafficChain, or false if there are
		// no available internal endpoints.
		internalPolicyChain := clusterPolicyChain
		hasInternalEndpoints := hasEndpoints
		if svcInfo.InternalPolicyLocal() {
			internalPolicyChain = localPolicyChain
			if len(localEndpoints) == 0 {
				hasInternalEndpoints = false
			}
		}
		internalTrafficChain := internalPolicyChain

		// Similarly, externalPolicyChain is the chain containing the endpoints
		// for "external" (NodePort, LoadBalancer, and ExternalIP) traffic.
		// externalTrafficChain is the chain that external traffic is routed to
		// (which is always the service's "EXT" chain). hasExternalEndpoints is
		// true if there are endpoints that will be reached by external traffic.
		// (But we may still have to generate externalTrafficChain even if there
		// are no external endpoints, to ensure that the short-circuit rules for
		// local traffic are set up.)
		externalPolicyChain := clusterPolicyChain
		hasExternalEndpoints := hasEndpoints
		if svcInfo.ExternalPolicyLocal() {
			externalPolicyChain = localPolicyChain
			if len(localEndpoints) == 0 {
				hasExternalEndpoints = false
			}
		}
		externalTrafficChain := svcInfo.externalChainName // eventually jumps to externalPolicyChain

		// usesExternalTrafficChain is based on hasEndpoints, not hasExternalEndpoints,
		// because we need the local-traffic-short-circuiting rules even when there
		// are no externally-usable endpoints.
		usesExternalTrafficChain := hasEndpoints && svcInfo.ExternallyAccessible()
		if usesExternalTrafficChain {
			activeNATChains[externalTrafficChain] = true
		}

		// Traffic to LoadBalancer IPs can go directly to externalTrafficChain
		// unless LoadBalancerSourceRanges is in use in which case we will
		// create a firewall chain.
		loadBalancerTrafficChain := externalTrafficChain
		fwChain := svcInfo.firewallChainName
		usesFWChain := hasEndpoints && len(svcInfo.LoadBalancerIPStrings()) > 0 && len(svcInfo.LoadBalancerSourceRanges()) > 0
		if usesFWChain {
			activeNATChains[fwChain] = true
			loadBalancerTrafficChain = fwChain
		}

		var internalTrafficFilterTarget, internalTrafficFilterComment string
		var externalTrafficFilterTarget, externalTrafficFilterComment string
		if !hasEndpoints {
			// The service has no endpoints at all; hasInternalEndpoints and
			// hasExternalEndpoints will also be false, and we will not
			// generate any chains in the "nat" table for the service; only
			// rules in the "filter" table rejecting incoming packets for
			// the service's IPs.
			internalTrafficFilterTarget = "REJECT"
			internalTrafficFilterComment = fmt.Sprintf(`"%s has no endpoints"`, svcPortNameString)
			externalTrafficFilterTarget = "REJECT"
			externalTrafficFilterComment = internalTrafficFilterComment
		} else {
			if !hasInternalEndpoints {
				// The internalTrafficPolicy is "Local" but there are no local
				// endpoints. Traffic to the clusterIP will be dropped, but
				// external traffic may still be accepted.
				internalTrafficFilterTarget = "DROP"
				internalTrafficFilterComment = fmt.Sprintf(`"%s has no local endpoints"`, svcPortNameString)
				serviceNoLocalEndpointsTotalInternal++
			}
			if !hasExternalEndpoints {
				// The externalTrafficPolicy is "Local" but there are no
				// local endpoints. Traffic to "external" IPs from outside
				// the cluster will be dropped, but traffic from inside
				// the cluster may still be accepted.
				externalTrafficFilterTarget = "DROP"
				externalTrafficFilterComment = fmt.Sprintf(`"%s has no local endpoints"`, svcPortNameString)
				serviceNoLocalEndpointsTotalExternal++
			}
		}

		// 找到cluster endpoint时,在 kubeServicesChain 中匹配了目标clusterip+端口时跳转到该服务特有的chain中处理
		if hasInternalEndpoints {
			proxier.natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", string(internalTrafficChain))
		} else {
			// cluster endpoint 为空时 跳转到 internalTrafficFilterTarget (REJECT)
			proxier.filterRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", internalTrafficFilterComment,
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", internalTrafficFilterTarget,
			)
		}

		// 同上,服务的外部ip也需要在 kubeServicesChain 中加上跳转处理,跳转到这个服务的专用chain
		for _, externalIP := range svcInfo.ExternalIPStrings() {
			if hasEndpoints {
				proxier.natRules.Write(
					"-A", string(kubeServicesChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s external IP"`, svcPortNameString),
					"-m", protocol, "-p", protocol,
					"-d", externalIP,
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", string(externalTrafficChain))
			}
			if !hasExternalEndpoints {
				// endpoint 为空时在kubeExternalServicesChain中也需要跳转到一个 externalTrafficFilterTarget 动作(REJECT)
				proxier.filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", protocol, "-p", protocol,
					"-d", externalIP,
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// LB设置
		for _, lbip := range svcInfo.LoadBalancerIPStrings() {
			if hasEndpoints {
				proxier.natRules.Write(
					"-A", string(kubeServicesChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s loadbalancer IP"`, svcPortNameString),
					"-m", protocol, "-p", protocol,
					"-d", lbip,
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", string(loadBalancerTrafficChain))

			}
			if usesFWChain {
				proxier.filterRules.Write(
					"-A", string(kubeProxyFirewallChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s traffic not accepted by %s"`, svcPortNameString, svcInfo.firewallChainName),
					"-m", protocol, "-p", protocol,
					"-d", lbip,
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", "DROP")
			}
		}
		if !hasExternalEndpoints {
			// Either no endpoints at all (REJECT) or no endpoints for
			// external traffic (DROP anything that didn't get short-circuited
			// by the EXT chain.)
			for _, lbip := range svcInfo.LoadBalancerIPStrings() {
				proxier.filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", protocol, "-p", protocol,
					"-d", lbip,
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// Capture nodeports.
		if svcInfo.NodePort() != 0 {
			if hasEndpoints {
				// Jump to the external destination chain.  For better or for
				// worse, nodeports are not subect to loadBalancerSourceRanges,
				// and we can't change that.
				proxier.natRules.Write(
					"-A", string(kubeNodePortsChain),
					"-m", "comment", "--comment", svcPortNameString,
					"-m", protocol, "-p", protocol,
					"--dport", strconv.Itoa(svcInfo.NodePort()),
					"-j", string(externalTrafficChain))
			}
			if !hasExternalEndpoints {
				// Either no endpoints at all (REJECT) or no endpoints for
				// external traffic (DROP anything that didn't get
				// short-circuited by the EXT chain.)
				proxier.filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", "addrtype", "--dst-type", "LOCAL",
					"-m", protocol, "-p", protocol,
					"--dport", strconv.Itoa(svcInfo.NodePort()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// Capture healthCheckNodePorts.
		if svcInfo.HealthCheckNodePort() != 0 {
			// no matter if node has local endpoints, healthCheckNodePorts
			// need to add a rule to accept the incoming connection
			proxier.filterRules.Write(
				"-A", string(kubeNodePortsChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s health check node port"`, svcPortNameString),
				"-m", "tcp", "-p", "tcp",
				"--dport", strconv.Itoa(svcInfo.HealthCheckNodePort()),
				"-j", "ACCEPT",
			)
		}

		// If the SVC/SVL/EXT/FW/SEP chains have not changed since the last sync
		// then we can omit them from the restore input. (We have already marked
		// them in activeNATChains, so they won't get deleted.)
		if tryPartialSync && !serviceChanged.Has(svcName.NamespacedName.String()) && !endpointsChanged.Has(svcName.NamespacedName.String()) {
			continue
		}

		// Set up internal traffic handling.
		if hasInternalEndpoints {
			args = append(args[:0],
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
			)
			if proxier.masqueradeAll {
				proxier.natRules.Write(
					"-A", string(internalTrafficChain),
					args,
					"-j", string(kubeMarkMasqChain))
			} else if proxier.localDetector.IsImplemented() {
				// This masquerades off-cluster traffic to a service VIP. The
				// idea is that you can establish a static route for your
				// Service range, routing to any node, and that node will
				// bridge into the Service for you. Since that might bounce
				// off-node, we masquerade here.
				proxier.natRules.Write(
					"-A", string(internalTrafficChain),
					args,
					proxier.localDetector.IfNotLocal(),
					"-j", string(kubeMarkMasqChain))
			}
		}

		// Set up external traffic handling (if any "external" destinations are
		// enabled). All captured traffic for all external destinations should
		// jump to externalTrafficChain, which will handle some special cases and
		// then jump to externalPolicyChain.
		if usesExternalTrafficChain {
			proxier.natChains.Write(utiliptables.MakeChainLine(externalTrafficChain))

			if !svcInfo.ExternalPolicyLocal() {
				// If we are using non-local endpoints we need to masquerade,
				// in case we cross nodes.
				proxier.natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"masquerade traffic for %s external destinations"`, svcPortNameString),
					"-j", string(kubeMarkMasqChain))
			} else {
				// If we are only using same-node endpoints, we can retain the
				// source IP in most cases.

				if proxier.localDetector.IsImplemented() {
					// Treat all locally-originated pod -> external destination
					// traffic as a special-case.  It is subject to neither
					// form of traffic policy, which simulates going up-and-out
					// to an external load-balancer and coming back in.
					proxier.natRules.Write(
						"-A", string(externalTrafficChain),
						"-m", "comment", "--comment", fmt.Sprintf(`"pod traffic for %s external destinations"`, svcPortNameString),
						proxier.localDetector.IfLocal(),
						"-j", string(clusterPolicyChain))
				}

				// Locally originated traffic (not a pod, but the host node)
				// still needs masquerade because the LBIP itself is a local
				// address, so that will be the chosen source IP.
				proxier.natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"masquerade LOCAL traffic for %s external destinations"`, svcPortNameString),
					"-m", "addrtype", "--src-type", "LOCAL",
					"-j", string(kubeMarkMasqChain))

				// Redirect all src-type=LOCAL -> external destination to the
				// policy=cluster chain. This allows traffic originating
				// from the host to be redirected to the service correctly.
				proxier.natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"route LOCAL traffic for %s external destinations"`, svcPortNameString),
					"-m", "addrtype", "--src-type", "LOCAL",
					"-j", string(clusterPolicyChain))
			}

			// Anything else falls thru to the appropriate policy chain.
			if hasExternalEndpoints {
				proxier.natRules.Write(
					"-A", string(externalTrafficChain),
					"-j", string(externalPolicyChain))
			}
		}

		// Set up firewall chain, if needed
		if usesFWChain {
			proxier.natChains.Write(utiliptables.MakeChainLine(fwChain))

			// The service firewall rules are created based on the
			// loadBalancerSourceRanges field. This only works for VIP-like
			// loadbalancers that preserve source IPs. For loadbalancers which
			// direct traffic to service NodePort, the firewall rules will not
			// apply.
			args = append(args[:0],
				"-A", string(fwChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s loadbalancer IP"`, svcPortNameString),
			)

			// firewall filter based on each source range
			allowFromNode := false
			for _, src := range svcInfo.LoadBalancerSourceRanges() {
				proxier.natRules.Write(args, "-s", src, "-j", string(externalTrafficChain))
				_, cidr, err := netutils.ParseCIDRSloppy(src)
				if err != nil {
					klog.ErrorS(err, "Error parsing CIDR in LoadBalancerSourceRanges, dropping it", "cidr", cidr)
				} else if cidr.Contains(proxier.nodeIP) {
					allowFromNode = true
				}
			}
			// For VIP-like LBs, the VIP is often added as a local
			// address (via an IP route rule).  In that case, a request
			// from a node to the VIP will not hit the loadbalancer but
			// will loop back with the source IP set to the VIP.  We
			// need the following rules to allow requests from this node.
			if allowFromNode {
				for _, lbip := range svcInfo.LoadBalancerIPStrings() {
					proxier.natRules.Write(
						args,
						"-s", lbip,
						"-j", string(externalTrafficChain))
				}
			}
			// If the packet was able to reach the end of firewall chain,
			// then it did not get DNATed, so it will match the
			// corresponding KUBE-PROXY-FIREWALL rule.
			proxier.natRules.Write(
				"-A", string(fwChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"other traffic to %s will be dropped by KUBE-PROXY-FIREWALL"`, svcPortNameString),
			)
		}

		// If Cluster policy is in use, create the chain and create rules jumping
		// from clusterPolicyChain to the clusterEndpoints
		if usesClusterPolicyChain {
			proxier.natChains.Write(utiliptables.MakeChainLine(clusterPolicyChain))
			proxier.writeServiceToEndpointRules(svcPortNameString, svcInfo, clusterPolicyChain, clusterEndpoints, args)
		}

		// If Local policy is in use, create the chain and create rules jumping
		// from localPolicyChain to the localEndpoints
		if usesLocalPolicyChain {
			proxier.natChains.Write(utiliptables.MakeChainLine(localPolicyChain))
			proxier.writeServiceToEndpointRules(svcPortNameString, svcInfo, localPolicyChain, localEndpoints, args)
		}

		// Generate the per-endpoint chains.
		for _, ep := range allLocallyReachableEndpoints {
			epInfo, ok := ep.(*endpointsInfo)
			if !ok {
				klog.ErrorS(nil, "Failed to cast endpointsInfo", "endpointsInfo", ep)
				continue
			}

			endpointChain := epInfo.ChainName

			// Create the endpoint chain
			proxier.natChains.Write(utiliptables.MakeChainLine(endpointChain))
			activeNATChains[endpointChain] = true

			args = append(args[:0], "-A", string(endpointChain))
			args = proxier.appendServiceCommentLocked(args, svcPortNameString)
			// Handle traffic that loops back to the originator with SNAT.
			proxier.natRules.Write(
				args,
				"-s", epInfo.IP(),
				"-j", string(kubeMarkMasqChain))
			// Update client-affinity lists.
			if svcInfo.SessionAffinityType() == v1.ServiceAffinityClientIP {
				args = append(args, "-m", "recent", "--name", string(endpointChain), "--set")
			}
			// DNAT to final destination.
			args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", epInfo.Endpoint)
			proxier.natRules.Write(args)
		}
	}

	// Delete chains no longer in use. Since "iptables-save" can take several seconds
	// to run on hosts with lots of iptables rules, we don't bother to do this on
	// every sync in large clusters. (Stale chains will not be referenced by any
	// active rules, so they're harmless other than taking up memory.)
	if !proxier.largeClusterMode || time.Since(proxier.lastIPTablesCleanup) > proxier.syncPeriod {
		var existingNATChains map[utiliptables.Chain]struct{}

		proxier.iptablesData.Reset()
		if err := proxier.iptables.SaveInto(utiliptables.TableNAT, proxier.iptablesData); err == nil {
			existingNATChains = utiliptables.GetChainsFromTable(proxier.iptablesData.Bytes())

			for chain := range existingNATChains {
				if !activeNATChains[chain] {
					chainString := string(chain)
					if !isServiceChainName(chainString) {
						// Ignore chains that aren't ours.
						continue
					}
					// We must (as per iptables) write a chain-line
					// for it, which has the nice effect of flushing
					// the chain. Then we can remove the chain.
					proxier.natChains.Write(utiliptables.MakeChainLine(chain))
					proxier.natRules.Write("-X", chainString)
				}
			}
			proxier.lastIPTablesCleanup = time.Now()
		} else {
			klog.ErrorS(err, "Failed to execute iptables-save: stale chains will not be deleted")
		}
	}

	// Finally, tail-call to the nodePorts chain.  This needs to be after all
	// other service portal rules.
	nodeAddresses, err := proxier.nodePortAddresses.GetNodeAddresses(proxier.networkInterfacer)
	if err != nil {
		klog.ErrorS(err, "Failed to get node ip address matching nodeport cidrs, services with nodeport may not work as intended", "CIDRs", proxier.nodePortAddresses)
	}
	// nodeAddresses may contain dual-stack zero-CIDRs if proxier.nodePortAddresses is empty.
	// Ensure nodeAddresses only contains the addresses for this proxier's IP family.
	for addr := range nodeAddresses {
		if utilproxy.IsZeroCIDR(addr) && isIPv6 == netutils.IsIPv6CIDRString(addr) {
			// if any of the addresses is zero cidr of this IP family, non-zero IPs can be excluded.
			nodeAddresses = sets.NewString(addr)
			break
		}
	}

	for address := range nodeAddresses {
		if utilproxy.IsZeroCIDR(address) {
			destinations := []string{"-m", "addrtype", "--dst-type", "LOCAL"}
			if isIPv6 {
				// For IPv6, Regardless of the value of localhostNodePorts is true
				// or false, we should disable access to the nodePort via localhost. Since it never works and always
				// cause kernel warnings.
				destinations = append(destinations, "!", "-d", "::1/128")
			}

			if !proxier.localhostNodePorts && !isIPv6 {
				// If set localhostNodePorts to "false"(route_localnet=0), We should generate iptables rules that
				// disable NodePort services to be accessed via localhost. Since it doesn't work and causes
				// the kernel to log warnings if anyone tries.
				destinations = append(destinations, "!", "-d", "127.0.0.0/8")
			}

			proxier.natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", `"kubernetes service nodeports; NOTE: this must be the last rule in this chain"`,
				destinations,
				"-j", string(kubeNodePortsChain))
			break
		}

		// Ignore IP addresses with incorrect version
		if isIPv6 && !netutils.IsIPv6String(address) || !isIPv6 && netutils.IsIPv6String(address) {
			klog.ErrorS(nil, "IP has incorrect IP version", "IP", address)
			continue
		}

		// For ipv6, Regardless of the value of localhostNodePorts is true or false, we should disallow access
		// to the nodePort via lookBack address.
		if isIPv6 && utilproxy.IsLoopBack(address) {
			klog.ErrorS(nil, "disallow nodePort services to be accessed via ipv6 localhost address", "IP", address)
			continue
		}

		// For ipv4, When localhostNodePorts is set to false, Ignore ipv4 lookBack address
		if !isIPv6 && utilproxy.IsLoopBack(address) && !proxier.localhostNodePorts {
			klog.ErrorS(nil, "disallow nodePort services to be accessed via ipv4 localhost address", "IP", address)
			continue
		}

		// create nodeport rules for each IP one by one
		proxier.natRules.Write(
			"-A", string(kubeServicesChain),
			"-m", "comment", "--comment", `"kubernetes service nodeports; NOTE: this must be the last rule in this chain"`,
			"-d", address,
			"-j", string(kubeNodePortsChain))
	}

	// Drop the packets in INVALID state, which would potentially cause
	// unexpected connection reset.
	// https://github.com/kubernetes/kubernetes/issues/74839
	proxier.filterRules.Write(
		"-A", string(kubeForwardChain),
		"-m", "conntrack",
		"--ctstate", "INVALID",
		"-j", "DROP",
	)

	// If the masqueradeMark has been added then we want to forward that same
	// traffic, this allows NodePort traffic to be forwarded even if the default
	// FORWARD policy is not accept.
	proxier.filterRules.Write(
		"-A", string(kubeForwardChain),
		"-m", "comment", "--comment", `"kubernetes forwarding rules"`,
		"-m", "mark", "--mark", fmt.Sprintf("%s/%s", proxier.masqueradeMark, proxier.masqueradeMark),
		"-j", "ACCEPT",
	)

	// The following rule ensures the traffic after the initial packet accepted
	// by the "kubernetes forwarding rules" rule above will be accepted.
	proxier.filterRules.Write(
		"-A", string(kubeForwardChain),
		"-m", "comment", "--comment", `"kubernetes forwarding conntrack rule"`,
		"-m", "conntrack",
		"--ctstate", "RELATED,ESTABLISHED",
		"-j", "ACCEPT",
	)

	metrics.IptablesRulesTotal.WithLabelValues(string(utiliptables.TableFilter)).Set(float64(proxier.filterRules.Lines()))
	metrics.IptablesRulesTotal.WithLabelValues(string(utiliptables.TableNAT)).Set(float64(proxier.natRules.Lines()))

	// Sync rules.
	proxier.iptablesData.Reset()
	proxier.iptablesData.WriteString("*filter\n")
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")
	proxier.iptablesData.WriteString("*nat\n")
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")

	klog.V(2).InfoS("Reloading service iptables data",
		"numServices", len(proxier.svcPortMap),
		"numEndpoints", totalEndpoints,
		"numFilterChains", proxier.filterChains.Lines(),
		"numFilterRules", proxier.filterRules.Lines(),
		"numNATChains", proxier.natChains.Lines(),
		"numNATRules", proxier.natRules.Lines(),
	)
	klog.V(9).InfoS("Restoring iptables", "rules", proxier.iptablesData.Bytes())

	// NOTE: NoFlushTables is used so we don't flush non-kubernetes chains in the table
	err = proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	if err != nil {
		if pErr, ok := err.(utiliptables.ParseError); ok {
			lines := utiliptables.ExtractLines(proxier.iptablesData.Bytes(), pErr.Line(), 3)
			klog.ErrorS(pErr, "Failed to execute iptables-restore", "rules", lines)
		} else {
			klog.ErrorS(err, "Failed to execute iptables-restore")
		}
		metrics.IptablesRestoreFailuresTotal.Inc()
		return
	}
	success = true
	proxier.needFullSync = false

	for name, lastChangeTriggerTimes := range endpointUpdateResult.LastChangeTriggerTimes {
		for _, lastChangeTriggerTime := range lastChangeTriggerTimes {
			latency := metrics.SinceInSeconds(lastChangeTriggerTime)
			metrics.NetworkProgrammingLatency.Observe(latency)
			klog.V(4).InfoS("Network programming", "endpoint", klog.KRef(name.Namespace, name.Name), "elapsed", latency)
		}
	}

	metrics.SyncProxyRulesNoLocalEndpointsTotal.WithLabelValues("internal").Set(float64(serviceNoLocalEndpointsTotalInternal))
	metrics.SyncProxyRulesNoLocalEndpointsTotal.WithLabelValues("external").Set(float64(serviceNoLocalEndpointsTotalExternal))
	if proxier.healthzServer != nil {
		proxier.healthzServer.Updated()
	}
	metrics.SyncProxyRulesLastTimestamp.SetToCurrentTime()

	// Update service healthchecks.  The endpoints list might include services that are
	// not "OnlyLocal", but the services list will not, and the serviceHealthServer
	// will just drop those endpoints.
	if err := proxier.serviceHealthServer.SyncServices(proxier.svcPortMap.HealthCheckNodePorts()); err != nil {
		klog.ErrorS(err, "Error syncing healthcheck services")
	}
	if err := proxier.serviceHealthServer.SyncEndpoints(proxier.endpointsMap.LocalReadyEndpoints()); err != nil {
		klog.ErrorS(err, "Error syncing healthcheck endpoints")
	}

	// Finish housekeeping.
	// Clear stale conntrack entries for UDP Services, this has to be done AFTER the iptables rules are programmed.
	// TODO: these could be made more consistent.
	klog.V(4).InfoS("Deleting conntrack stale entries for services", "IPs", conntrackCleanupServiceIPs.UnsortedList())
	for _, svcIP := range conntrackCleanupServiceIPs.UnsortedList() {
		if err := conntrack.ClearEntriesForIP(proxier.exec, svcIP, v1.ProtocolUDP); err != nil {
			klog.ErrorS(err, "Failed to delete stale service connections", "IP", svcIP)
		}
	}
	klog.V(4).InfoS("Deleting conntrack stale entries for services", "nodePorts", conntrackCleanupServiceNodePorts.UnsortedList())
	for _, nodePort := range conntrackCleanupServiceNodePorts.UnsortedList() {
		err := conntrack.ClearEntriesForPort(proxier.exec, nodePort, isIPv6, v1.ProtocolUDP)
		if err != nil {
			klog.ErrorS(err, "Failed to clear udp conntrack", "nodePort", nodePort)
		}
	}
	klog.V(4).InfoS("Deleting stale endpoint connections", "endpoints", endpointUpdateResult.DeletedUDPEndpoints)
	proxier.deleteUDPEndpointConnections(endpointUpdateResult.DeletedUDPEndpoints)
}
```

# chain 跳转图
## input
给filter 的input chain依次加上几个自定义跳转的chain,从外部进来的流量通过这3个自定义chain最终到达目标pod
![filter-input-chain](resource/kubeProxyInitInputChainJump.drawio.svg)

## output
给filter 的output chain 加上几个自定义跳转的chain,从内部出去的流量通过这几个自定义chain找到目标机器地址
![filter-output-chain](resource/kubeProxyInitOutputChainJump.drawio.svg)