# 版本
+ github: vmware-tanzu/velero
+ 分支: release-1.9

# 序
restoreController 监听k8s 集群中的新 `Restore`资源

# 代码
## processRestore
监听到有新的`Restore`资源创建时触发
```go
func (c *restoreController) processRestore(restore *api.Restore) error {
    // 复制一份k8s资源到本地(便于修改和更新)
	original := restore.DeepCopy()

	// 检验‘Restore’资源的有效性
	info := c.validateAndComplete(restore, pluginManager)

	backupScheduleName := restore.Spec.ScheduleName

	if len(restore.Status.ValidationErrors) > 0 { // 如果校验中有报错
		restore.Status.Phase = api.RestorePhaseFailedValidation
        // ...
    } else {
        // 没有报错则更新本地restore实例状态
		restore.Status.StartTimestamp = &metav1.Time{Time: c.clock.Now()}
		restore.Status.Phase = api.RestorePhaseInProgress
	}

	// 将本地restore实例状态更新到k8s集群
	updatedRestore, err := patchRestore(original, restore, c.restoreClient)
	
	// 复制一份最新的k8s restore资源到本地
	original = updatedRestore
	restore = updatedRestore.DeepCopy()

	if restore.Status.Phase == api.RestorePhaseFailedValidation {
		return nil
	}

    // #ref runValidatedRestore 运行实际的恢复任务
	if err := c.runValidatedRestore(restore, info); err != nil {
		c.logger.WithError(err).Debug("Restore failed")
		restore.Status.Phase = api.RestorePhaseFailed
		restore.Status.FailureReason = err.Error()
		c.metrics.RegisterRestoreFailed(backupScheduleName)
	} else if restore.Status.Errors > 0 {
		c.logger.Debug("Restore partially failed")
		restore.Status.Phase = api.RestorePhasePartiallyFailed
		c.metrics.RegisterRestorePartialFailure(backupScheduleName)
	} else {
		c.logger.Debug("Restore completed")
		restore.Status.Phase = api.RestorePhaseCompleted
		c.metrics.RegisterRestoreSuccess(backupScheduleName)
	}

	restore.Status.CompletionTimestamp = &metav1.Time{Time: c.clock.Now()}
	c.logger.Debug("Updating restore's final status")
	if _, err = patchRestore(original, restore, c.restoreClient); err != nil {
		c.logger.WithError(errors.WithStack(err)).Info("Error updating restore's final status")
	}

	return nil
}
```

### runValidatedRestore
接收一个已经校验了参数的Restore实例,并执行恢复
```go
func (c *restoreController) runValidatedRestore(restore *api.Restore, info backupInfo) error {
	// ...

    // 根据‘Restore’资源里的BackupName,先把备份的文件下载到本地
	backupFile, err := downloadToTempFile(restore.Spec.BackupName, info.backupStore, restoreLog)
    // ...

    // 构建本地参数
	restoreReq := pkgrestore.Request{
		Log:              restoreLog,
		Restore:          restore,
		Backup:           info.backup,
		PodVolumeBackups: podVolumeBackups,
		VolumeSnapshots:  volumeSnapshots,
		BackupReader:     backupFile,
	}
    // #ref RestoreWithResolvers 
	restoreWarnings, restoreErrors := c.restorer.RestoreWithResolvers(restoreReq, actionsResolver, snapshotItemResolver,
		c.snapshotLocationLister, pluginManager)
    // ...

	return nil
}
```

### RestoreWithResolvers
- 入参
    - req Restore的参数
```go
// pkg/restore/restore.go
func (kr *kubernetesRestorer) RestoreWithResolvers(
	req Request,
	restoreItemActionResolver framework.RestoreItemActionResolver,
	itemSnapshotterResolver framework.ItemSnapshotterResolver,
	snapshotLocationLister listers.VolumeSnapshotLocationLister,
	volumeSnapshotterGetter VolumeSnapshotterGetter,
) (Result, Result) {
    // ...
    // 整理好参数后创建 restoreContext
	restoreCtx := &restoreContext{
		backup:                         req.Backup,
		backupReader:                   req.BackupReader,
		restore:                        req.Restore,
		resourceIncludesExcludes:       resourceIncludesExcludes,
		resourceStatusIncludesExcludes: restoreStatusIncludesExcludes,
		namespaceIncludesExcludes:      namespaceIncludesExcludes,
		chosenGrpVersToRestore:         make(map[string]ChosenGroupVersion),
		selector:                       selector,
		OrSelectors:                    OrSelectors,
		log:                            req.Log,
		dynamicFactory:                 kr.dynamicFactory,
		fileSystem:                     kr.fileSystem,
		namespaceClient:                kr.namespaceClient,
		restoreItemActions:             resolvedActions,
		itemSnapshotterActions:         resolvedItemSnapshotterActions,
		volumeSnapshotterGetter:        volumeSnapshotterGetter,
		resticRestorer:                 resticRestorer,
		resticErrs:                     make(chan error),
		pvsToProvision:                 sets.NewString(),
		pvRestorer:                     pvRestorer,
		volumeSnapshots:                req.VolumeSnapshots,
		podVolumeBackups:               req.PodVolumeBackups,
		resourceTerminatingTimeout:     kr.resourceTerminatingTimeout,
		resourceClients:                make(map[resourceClientKey]client.Dynamic),
		restoredItems:                  make(map[velero.ResourceIdentifier]struct{}),
		renamedPVs:                     make(map[string]string),
		pvRenamer:                      kr.pvRenamer,
		discoveryHelper:                kr.discoveryHelper,
		resourcePriorities:             kr.resourcePriorities,
		resourceRestoreHooks:           resourceRestoreHooks,
		hooksErrs:                      make(chan error),
		waitExecHookHandler:            waitExecHookHandler,
		hooksContext:                   hooksCtx,
		hooksCancelFunc:                hooksCancelFunc,
		restoreClient:                  kr.restoreClient,
	}
    // #ref restoreCtx.execute
	return restoreCtx.execute()
}

func (ctx *restoreContext) execute() (Result, Result) {
	warnings, errs := Result{}, Result{}

    // 解压备份文件
	dir, err := archive.NewExtractor(ctx.log, ctx.fileSystem).UnzipAndExtractBackup(ctx.backupReader)
	ctx.restoreDir = dir
    // 解析解压后的文件夹,将资源格式化成本地的数据结构实例
	backupResources, err := archive.NewParser(ctx.log, ctx.fileSystem).Parse(ctx.restoreDir)
    // ...

    // 另启一个goroutine,接收‘restore’过程中的进度信息,并更新到k8s 的restore资源
	update := make(chan progressUpdate)
	quit := make(chan struct{})
	go func() {
		ticker := time.NewTicker(1 * time.Second)
		var lastUpdate *progressUpdate
		for {
			select {
			case <-quit:
				ticker.Stop()
				return
			case val := <-update:
				lastUpdate = &val
			case <-ticker.C:
                // 周期更新k8s集群内的restore资源item的进度
				if lastUpdate != nil {
					patch := fmt.Sprintf(
						`{"status":{"progress":{"totalItems":%d,"itemsRestored":%d}}}`,
						lastUpdate.totalItems,
						lastUpdate.itemsRestored,
					)
					_, err := ctx.restoreClient.Restores(ctx.restore.Namespace).Patch(
						go_context.TODO(),
						ctx.restore.Name,
						types.MergePatchType,
						[]byte(patch),
						metav1.PatchOptions{},
					)
					// ...
					lastUpdate = nil
				}
			}
		}
	}()

	// totalItems: previously discovered items, i: iteration counter.
	totalItems, processedItems, existingNamespaces := 0, 0, sets.NewString()

	// 根据备份的资源选出要先恢复的与这些资源相关的‘CRD’, backupResources是所有备份的资源,但返回的结果会过滤出满足restore参数里选择的过滤条件的资源
    // 参数中的 Priorities{HighPriorities: []string{"customresourcedefinitions"}} 和 false 表明只从 backupResources中取出customresourcedefinitions这一种资源类型 
	crdResourceCollection, processedResources, w, e := ctx.getOrderedResourceCollection(
		backupResources,
		make([]restoreableResource, 0),
		sets.NewString(),
		Priorities{HighPriorities: []string{"customresourcedefinitions"}},
		false,
	)
	warnings.Merge(&w)
	errs.Merge(&e)

	for _, selectedResource := range crdResourceCollection {
		totalItems += selectedResource.totalItems
	}

    // 执行‘CRD’资源的恢复
	for _, selectedResource := range crdResourceCollection {
		var w, e Result
		// 执行单个选中资源的恢复 #ref processSelectedResource
		processedItems, w, e = ctx.processSelectedResource(
			selectedResource,
			totalItems,
			processedItems,
			existingNamespaces,
			update,
		)
		warnings.Merge(&w)
		errs.Merge(&e)
	}

	// 准备好其他备份的资源
	selectedResourceCollection, _, w, e := ctx.getOrderedResourceCollection(
		backupResources,
		crdResourceCollection,
		processedResources,
		ctx.resourcePriorities,
		true,
	)
	warnings.Merge(&w)
	errs.Merge(&e)

	// reset processedItems and totalItems before processing full resource list
	processedItems = 0
	totalItems = 0
	for _, selectedResource := range selectedResourceCollection {
		totalItems += selectedResource.totalItems
	}

	for _, selectedResource := range selectedResourceCollection {
		var w, e Result
		// 执行单个选中资源的恢复 #ref processSelectedResource
		processedItems, w, e = ctx.processSelectedResource(
			selectedResource,
			totalItems,
			processedItems,
			existingNamespaces,
			update,
		)
		warnings.Merge(&w)
		errs.Merge(&e)
	}

	// Close the progress update channel.
	quit <- struct{}{}

    // 更新k8s集群内的‘Restore’资源的状态进度
	patch := fmt.Sprintf(
		`{"status":{"progress":{"totalItems":%d,"itemsRestored":%d}}}`,
		len(ctx.restoredItems),
		len(ctx.restoredItems),
	)
	_, err = ctx.restoreClient.Restores(ctx.restore.Namespace).Patch(
		go_context.TODO(),
		ctx.restore.Name,
		types.MergePatchType,
		[]byte(patch),
		metav1.PatchOptions{},
	)
    // ...
	return warnings, errs
}
```

### getOrderedResourceCollection
按照优先级从完整的备份数据里取出满足条件的资源,并按优先级排序
- 入参:
    - backupResources 传入完整的格式化好了的资源备份数据
    - restoreResourceCollection 既是入参也是返回,过滤出的资源就是加入到这个集合里再返回
    - processedResources 已经运行过的资源类型,本方法调用结束前也会更新这个参数
    - resourcePriorities 指定的优先级资源(`high`, `low`)
    - includeAllResources(bool) 为false时只过滤返回`resourcePriorities`参数里的高优先级的备份数据
- 返回:
    - []restoreableResource 根据条件过滤出的备份数据集合

```go
// pkg/restore/restore.go
func (ctx *restoreContext) getOrderedResourceCollection(
	backupResources map[string]*archive.ResourceItems,
	restoreResourceCollection []restoreableResource,
	processedResources sets.String,
	resourcePriorities Priorities,
	includeAllResources bool,
) ([]restoreableResource, sets.String, Result, Result) {
	var warnings, errs Result
	var resourceList []string
	if includeAllResources {
		resourceList = getOrderedResources(resourcePriorities, backupResources)
	} else {
        // ‘includeAllResources’参数为‘false’时 只遍历‘resourcePriorities’参数里指定的高优先级资源类型
		resourceList = resourcePriorities.HighPriorities
	}
    // 遍历每种资源类型
	for _, resource := range resourceList {
		// 资源类型有很多种名字,找到资源的官方名字,便于统一处理
		gvr, _, err := ctx.discoveryHelper.ResourceFor(schema.ParseGroupResource(resource).WithVersion(""))
		groupResource := gvr.GroupResource()

		// 如果该资源类型已经处理过了,则跳过
		if processedResources.Has(groupResource.String()) {
			// ...
			continue
		}

		// 如果该资源不在配置的要恢复的资源类型里,则跳过
		if !ctx.resourceIncludesExcludes.ShouldInclude(groupResource.String()) {
			// ...
			continue
		}

		// 如果资源类型就是命名空间,则跳过, 命名空间的恢复有特殊的处理逻辑
		if groupResource == kuberesource.Namespaces {
			continue
		}

		// 如果备份的数据里压根没有这个资源,跳过
		resourceList := backupResources[groupResource.String()]
		if resourceList == nil {
			// ...
			continue
		}

		// 遍历该资源的所有命名空间(如果是cluster scope资源则只有一个namespace==“”的)
		for namespace, items := range resourceList.ItemsByNamespace {
            // 资源是namespaced类型的,且命名空间不在restore参数的‘include’里时,则跳过
			if namespace != "" && !ctx.namespaceIncludesExcludes.ShouldInclude(namespace) {
				// ...
				continue
			}

			// 配置了命名空间映射的需要转下
			targetNamespace := namespace
			if target, ok := ctx.restore.Spec.NamespaceMapping[namespace]; ok {
				targetNamespace = target
			}

            // 资源类型是‘cluster-scoped’ 且‘restore’参数明确指定了‘IncludeClusterResources’为false时,跳过
			if targetNamespace == "" && boolptr.IsSetToFalse(ctx.restore.Spec.IncludeClusterResources) {
                // ...
                continue
			}

            // 资源类型是‘cluster-scoped’ 
            // 且‘restore’没有明确指定‘IncludeClusterResources’
            // 且‘restore’没有明确指定 include 所有命名空间时跳过
			if targetNamespace == "" && !boolptr.IsSetToTrue(ctx.restore.Spec.IncludeClusterResources) && !ctx.namespaceIncludesExcludes.IncludeEverything() {
				// ...
				continue
			}

            // 到这里代表该资源类型该命名空间下的资源满足条件了,整理信息,append到返回结果‘restoreResourceCollection’中
			res, w, e := ctx.getSelectedRestoreableItems(groupResource.String(), targetNamespace, namespace, items)
			warnings.Merge(&w)
			errs.Merge(&e)

			restoreResourceCollection = append(restoreResourceCollection, res)
		}

		// 更新‘processedResources’ 表明该资源类型已被处理过
		processedResources.Insert(groupResource.String())
	}
	return restoreResourceCollection, processedResources, warnings, errs
}
```

### processSelectedResource
执行一个资源类型的恢复(资源类型下有多个资源实例)
- 入参:
    - selectedResource 要执行restore的资源(资源类型下有多个资源实例)
    - existingNamespaces 已经存在的命名空间,用于记录已经处理过哪个命名空间的恢复,方法内也可以添加

```go
// pkg/restore/restore.go
func (ctx *restoreContext) processSelectedResource(
	selectedResource restoreableResource,
	totalItems int,
	processedItems int,
	existingNamespaces sets.String,
	update chan progressUpdate,
) (int, Result, Result) {
	warnings, errs := Result{}, Result{}
	groupResource := schema.ParseGroupResource(selectedResource.resource)

    // 资源类型下的资源实例按命名空间存放的,cluster-scoped的资源就一个为“”的命名空间
	for namespace, selectedItems := range selectedResource.selectedItemsByNamespace {
        // 遍历该命名空间下的每一个资源实例
		for _, selectedItem := range selectedItems {
			// namespaced 资源需要先确认要恢复的目标命名空间的存在情况(不存在需要走一套创建命名空间的逻辑)
			if namespace != "" && !existingNamespaces.Has(selectedItem.targetNamespace) {
				ns := getNamespace(
					logger,
					archive.GetItemFilePath(ctx.restoreDir, "namespaces", "", namespace),
					selectedItem.targetNamespace,
				)
                // 先尝试根据备份数据里的命名空间信息(比如labels)创建新命名空间,(如果已经存在就不会再创建了)
				_, nsCreated, err := kube.EnsureNamespaceExistsAndIsReady(
					ns,
					ctx.namespaceClient,
					ctx.resourceTerminatingTimeout,
				)

				// 表明目标命名空间自身已经处理过了
				existingNamespaces.Insert(selectedItem.targetNamespace)
			}

            // 从文件中取出资源
			obj, err := archive.Unmarshal(ctx.fileSystem, selectedItem.path)
			// ...
            // #ref restoreItem
			w, e := ctx.restoreItem(obj, groupResource, selectedItem.targetNamespace)
			warnings.Merge(&w)
			errs.Merge(&e)
			processedItems++

			// 通过channel更新进度
			actualTotalItems := len(ctx.restoredItems) + (totalItems - processedItems)
			update <- progressUpdate{
				totalItems:    actualTotalItems,
				itemsRestored: len(ctx.restoredItems),
			}
		}
	}

	return processedItems, warnings, errs
}
```

### restoreItem
- 入参
    - obj 结构化的资源
    - groupResource
    - namespace
```go
// pkg/restore/restore.go
func (ctx *restoreContext) restoreItem(obj *unstructured.Unstructured, groupResource schema.GroupResource, namespace string) (Result, Result) {
	warnings, errs := Result{}, Result{}
	resourceID := getResourceID(groupResource, namespace, obj.GetName())

    // 判断资源是否满足‘restore’配置的‘resource’规则
	if !ctx.resourceIncludesExcludes.ShouldInclude(groupResource.String()) {
		// ...
		return warnings, errs
	}

	if namespace != "" { // namespace-scoped 资源
		if !ctx.namespaceIncludesExcludes.ShouldInclude(obj.GetNamespace()) { // 判断是否满足‘restore’配置里的命名空间筛选
			return warnings, errs
		}

		// 从备份文件中取中命名空间资源详情(并把名字换成remapping后的名字)
		nsToEnsure := getNamespace(ctx.log, archive.GetItemFilePath(ctx.restoreDir, "namespaces", "", obj.GetNamespace()), namespace)
        // 确保k8s集群中命名空间已经就绪(有则略过,无则创建)
		if _, nsCreated, err := kube.EnsureNamespaceExistsAndIsReady(nsToEnsure, ctx.namespaceClient, ctx.resourceTerminatingTimeout); err != nil {
			// ...
			return warnings, errs
		} else {
			// ...
		}
	} else { // cluster-scoped 资源
        // 如果没有规则表明可以恢复这个‘cluster-scoped’资源,则返回
		if boolptr.IsSetToFalse(ctx.restore.Spec.IncludeClusterResources) {
			// ...
			return warnings, errs
		}
	}

	
	itemFromBackup := obj.DeepCopy()

    // 特殊资源‘pod’和‘job’的状态判断,pod.status.phase的PodFaile或podSucceeded 以及 job.status.completionTime不为空时,代表这个资源已经完成使命,不需要恢复
	complete, err := isCompleted(obj, groupResource)
	// ...
	if complete {
        //...
		return warnings, errs
	}

	name := obj.GetName()

    // 当前资源item实例已经恢复过了则略过
	itemKey := velero.ResourceIdentifier{
		GroupResource: groupResource,
		Namespace:     namespace,
		Name:          name,
	}
	if _, exists := ctx.restoredItems[itemKey]; exists {
		//...
		return warnings, errs
	}
	ctx.restoredItems[itemKey] = struct{}{}


	resourceClient, err := ctx.getResourceClient(groupResource, obj, namespace)
	if err != nil {
		errs.AddVeleroError(fmt.Errorf("error getting resource client for namespace %q, resource %q: %v", namespace, &groupResource, err))
		return warnings, errs
	}

	if groupResource == kuberesource.PersistentVolumes {
		// ...
	}

	objStatus, statusFieldExists, statusFieldErr := unstructured.NestedFieldCopy(obj.Object, "status")
	// 清除掉要复原的资源item实例里的一些信息
    // ‘metadata’只保留 ‘name’,‘namespace’,‘labels’,‘annotations’
    // 清除‘status’
	if obj, err = resetMetadataAndStatus(obj); err != nil {
		errs.Add(namespace, err)
		return warnings, errs
	}

	// 资源数据的修正做完了,最后实际还原前需要把资源的‘namespace’换成配置的要替换成的‘namespace’
	originalNamespace := obj.GetNamespace()
	if namespace != "" {
		obj.SetNamespace(namespace)
	}

	// 给要恢复的资源item实例打两个label,便于管理和review
	addRestoreLabels(obj, ctx.restore.Name, ctx.restore.Spec.BackupName)

	// ...
    // 向k8s集群提交创建新资源item
	createdObj, restoreErr := resourceClient.Create(obj)
	isAlreadyExistsError, err := isAlreadyExistsError(ctx, obj, restoreErr, resourceClient)
	if err != nil {
		errs.Add(namespace, err)
		return warnings, errs
	}

	// 判断资源是已经在集群里存在
	objectExists := false
	var fromCluster *unstructured.Unstructured
	if restoreErr != nil && !isAlreadyExistsError {
		fromCluster, err = resourceClient.Get(name, metav1.GetOptions{})
		if err == nil {
			objectExists = true
		}
	}

    // 如果提交时资源已经存在了
	if isAlreadyExistsError || objectExists {
		// do a get call if we did not run this previously i.e.
		// we've only run this for errors other than isAlreadyExistError
		if fromCluster == nil {
			fromCluster, err = resourceClient.Get(name, metav1.GetOptions{})
			if err != nil {
				ctx.log.Errorf("Error retrieving cluster version of %s: %v", kube.NamespaceAndName(obj), err)
				errs.Add(namespace, err)
				return warnings, errs
			}
		}
		// 清除掉从集群获取的资源item实例里的一些信息
        // ‘metadata’只保留 ‘name’,‘namespace’,‘labels’,‘annotations’
        // 清除‘status’
		fromCluster, err = resetMetadataAndStatus(fromCluster)
		labels := obj.GetLabels()
        // 加上一些恢复的label(使得下面深度对比时,不会因为一些无关项干扰)
		addRestoreLabels(fromCluster, labels[velerov1api.RestoreNameLabel], labels[velerov1api.BackupNameLabel])
		fromClusterWithLabels := fromCluster.DeepCopy() // saving the in-cluster object so that we can create label patch if overall patch fails

        // 对比集群里的资源item 与 本地要改动的资源item,如果发现不同,则需要做额外处理
		if !equality.Semantic.DeepEqual(fromCluster, obj) {
			switch groupResource {
            // serviceaccount资源需要合并 patch提交(无论ExistingResourcePolicy有没有设置)
			case kuberesource.ServiceAccounts:
				desired, err := mergeServiceAccounts(fromCluster, obj)
                // ...
				patchBytes, err := generatePatch(fromCluster, desired)
                // ...
				_, err = resourceClient.Patch(name, patchBytes)
				// ...
			default:
				// 非serviceaccount资源item需要先确认‘restore’的同资源名策略配置
				if len(ctx.restore.Spec.ExistingResourcePolicy) > 0 { // 如果配置了
					resourcePolicy := ctx.restore.Spec.ExistingResourcePolicy
					
					if resourcePolicy == velerov1api.PolicyTypeNone { // 设置了‘none’则生成一条warning
						e := errors.Errorf("could not restore, %s %q already exists. Warning: the in-cluster version is different than the backed-up version.",
							obj.GetKind(), obj.GetName())
						warnings.Add(namespace, e)
 					} else if resourcePolicy == velerov1api.PolicyTypeUpdate { // 设置了‘update’则需要更新资源item的信息
						// processing update as existingResourcePolicy
						warningsFromUpdateRP, errsFromUpdateRP := ctx.processUpdateResourcePolicy(fromCluster, fromClusterWithLabels, obj, namespace, resourceClient)
						warnings.Merge(&warningsFromUpdateRP)
						errs.Merge(&errsFromUpdateRP)
					}
				} else { // 没有设置就是默认none
					e := errors.Errorf("could not restore, %s %q already exists. Warning: the in-cluster version is different than the backed-up version.",
						obj.GetKind(), obj.GetName())
					warnings.Add(namespace, e)
				}
			}
			return warnings, errs
		}

		//update backup/restore labels on the unchanged resources if existingResourcePolicy is set as update
		if ctx.restore.Spec.ExistingResourcePolicy == velerov1api.PolicyTypeUpdate {
			resourcePolicy := ctx.restore.Spec.ExistingResourcePolicy
			
			removeRestoreLabels(fromCluster)
			// try updating the backup/restore labels for the in-cluster object
			warningsFromUpdate, errsFromUpdate := ctx.updateBackupRestoreLabels(fromCluster, obj, namespace, resourceClient)
			warnings.Merge(&warningsFromUpdate)
			errs.Merge(&errsFromUpdate)
		}
		return warnings, errs
	}

	// 当创建的‘CRD’时需要强同步,等带‘CRD’状态可用后再继续(后面有的资源恢复可能会依赖CRD)
	if groupResource == kuberesource.CustomResourceDefinitions {
		available, err := ctx.crdAvailable(name, resourceClient)
		if err != nil {
			errs.Add(namespace, errors.Wrapf(err, "error verifying custom resource definition is ready to use"))
		} else if !available {
			errs.Add(namespace, fmt.Errorf("CRD %s is not available to use for custom resources.", name))
		}
	}

	return warnings, errs
}
```
