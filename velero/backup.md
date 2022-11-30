# 版本
+ github: vmware-tanzu/velero
+ 分支: release-1.9

# 序
BackupController 监听k8s 集群中的新 Backup资源

# 代码
## processBackup
有新的Backup资源创建时触发
```go
// pkg/controller/backup_controller.go
func (c *backupController) processBackup(key string) error {
	// 先找到对应的Backup资源
	ns, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		log.WithError(err).Errorf("error splitting key")
		return nil
	}

	log.Debug("Getting backup")
	original, err := c.lister.Backups(ns).Get(name)
	// ...

	// backup不是初始状态直接跳过 todo ?? 什么情况会触发这种情形
	switch original.Status.Phase {
	case "", velerov1api.BackupPhaseNew:
		// only process new backups
	default:
		return nil
	}

	// 做些检测工作,判断’backup‘资源的各个参数是否正确
	request := c.prepareBackupRequest(original)

    // 根据错误情况更新 Phase,这里的更新的是本地副本中的状态,还没patch到k8s服务上
	if len(request.Status.ValidationErrors) > 0 {
        // 更新
		request.Status.Phase = velerov1api.BackupPhaseFailedValidation
	} else {
        // 没有报错则更新’Phase‘状态,开始做事
		request.Status.Phase = velerov1api.BackupPhaseInProgress
		request.Status.StartTimestamp = &metav1.Time{Time: c.clock.Now()}
	}

	// 这里把本地修改了状态后的backup资源信息patch到‘k8s集群’上,
    // 返回更新后的的‘backup’资源(patch到k8s集群后,集群也会加上一些信息)
	updatedBackup, err := patchBackup(original, request.Backup, c.client)
	if err != nil {
		return errors.Wrapf(err, "error updating Backup status to %s", request.Status.Phase)
	}

	original = updatedBackup
    // 复制一份最新的backup资源信息 到reqeust
	request.Backup = updatedBackup.DeepCopy()

    // 现在来处理前面校验backup资源参数的错误,有错直接返回,不继续折腾
	if request.Status.Phase == velerov1api.BackupPhaseFailedValidation {
		return nil
	}


    // todo 这个tracker的作用
	c.backupTracker.Add(request.Namespace, request.Name)
	defer c.backupTracker.Delete(request.Namespace, request.Name)


	backupScheduleName := request.GetLabels()[velerov1api.ScheduleNameLabel]
	c.metrics.RegisterBackupAttempt(backupScheduleName)

	// 开始运行 #ref runBackup
	if err := c.runBackup(request); err != nil {
		// ...
		request.Status.Phase = velerov1api.BackupPhaseFailed
		request.Status.FailureReason = err.Error()
	}

	switch request.Status.Phase {
	case velerov1api.BackupPhaseCompleted:
		c.metrics.RegisterBackupSuccess(backupScheduleName)
	case velerov1api.BackupPhasePartiallyFailed:
		c.metrics.RegisterBackupPartialFailure(backupScheduleName)
	case velerov1api.BackupPhaseFailed:
		c.metrics.RegisterBackupFailed(backupScheduleName)
	case velerov1api.BackupPhaseFailedValidation:
		c.metrics.RegisterBackupValidationFailure(backupScheduleName)
	}

	log.Debug("Updating backup's final status")
	if _, err := patchBackup(original, request.Backup, c.client); err != nil {
		log.WithError(err).Error("error updating backup's final status")
	}

	return nil
}

```

### runBackup
根据backup资源的设置运行实际的备份
```go
func (c *backupController) runBackup(backup *pkgbackup.Request) error {
	// 创建本地log文件
	logFile, err := ioutil.TempFile("", "")
	// ...
    // 将日志文件的writer改为’gz‘(最终存到远端文件存储的就是gz格式)
	gzippedLogFile := gzip.NewWriter(logFile)
	
    // 将
	logger := logging.DefaultLogger(c.backupLogLevel, c.formatFlag)
	logger.Out = io.MultiWriter(os.Stdout, gzippedLogFile)

	logCounter := logging.NewLogCounterHook()
	logger.Hooks.Add(logCounter)

	backupLog := logger.WithField(Backup, kubeutil.NamespaceAndName(backup))

    // 创建备份文件
	backupFile, err := ioutil.TempFile("", "")
	// ...

	pluginManager := c.newPluginManager(backupLog)
	defer pluginManager.CleanupClients()

    // todo 用什么方法获取所有k8s资源到本地并存好
    actions, err := pluginManager.GetBackupItemActions()
	if err != nil {
		return err
	}

	// 确认Backup资源配置的‘BackupStore’的有效性
	backupStore, err := c.backupStoreGetter.Get(backup.StorageLocation, pluginManager, backupLog)
	// ...
    // 确认远端‘BackupStore’是否存在同名’Backup‘
	exists, err := backupStore.BackupExists(backup.StorageLocation.Spec.StorageType.ObjectStorage.Bucket, backup.Name)
	// ...

	var fatalErrs []error
    // 根据Backup实例配置从集群中获取资源的信息并保存到本地文件中 #ref BackupWithResolvers
	if err := c.backupper.BackupWithResolvers(backupLog, backup, backupFile, backupItemActionsResolver,
		itemSnapshottersResolver, pluginManager); err != nil {
		fatalErrs = append(fatalErrs, err)
	}

	// 更新本地Backup实例备份过程中遇到的告警与错误信息
	backup.Status.CompletionTimestamp = &metav1.Time{Time: c.clock.Now()}
	backup.Status.Warnings = logCounter.GetCount(logrus.WarnLevel)
	backup.Status.Errors = logCounter.GetCount(logrus.ErrorLevel)

	// 更新本地Backup实例的状态信息
	switch {
	case len(fatalErrs) > 0:
		backup.Status.Phase = velerov1api.BackupPhaseFailed
	case logCounter.GetCount(logrus.ErrorLevel) > 0:
		backup.Status.Phase = velerov1api.BackupPhasePartiallyFailed
	default:
		backup.Status.Phase = velerov1api.BackupPhaseCompleted
	}
    //...
    // 将保存到本地的备份信息持久化道远端存储服务器上
	if errs := persistBackup(backup, backupFile, logFile, backupStore, volumeSnapshots, volumeSnapshotContents, volumeSnapshotClasses); len(errs) > 0 {
		fatalErrs = append(fatalErrs, errs...)
	}

	return kerrors.NewAggregate(fatalErrs)
}
```

#### BackupWithResolvers
从集群中获取备份资源,并格式化保存到本地文件
- 入参
    - backupRequest 备份的过滤条件
    - backupFile  资源解析后需要通过这个io.Writer写到对应的文件中
```go
// pkg/backup/backup.go
func (kb *kubernetesBackupper) BackupWithResolvers(log logrus.FieldLogger,
	backupRequest *Request,
	backupFile io.Writer,
	backupItemActionResolver framework.BackupItemActionResolver,
	itemSnapshotterResolver framework.ItemSnapshotterResolver,
	volumeSnapshotterGetter VolumeSnapshotterGetter) error {
        
	gzippedData := gzip.NewWriter(backupFile)
	defer gzippedData.Close()

	tw := tar.NewWriter(gzippedData)
	defer tw.Close()

	// ...
	backupRequest.NamespaceIncludesExcludes = getNamespaceIncludesExcludes(backupRequest.Backup)

	backupRequest.ResourceIncludesExcludes = collections.GetResourceIncludesExcludes(kb.discoveryHelper, backupRequest.Spec.IncludedResources, backupRequest.Spec.ExcludedResources)
    // ...
    // 创建一个map结构保存所有要备份的资源详情(itemKey 包括 namespace,resource,name )
	backupRequest.BackedUpItems = map[itemKey]struct{}{}
    // ...

	// 创建临时文件保存所有从集群获取的资源
	tempDir, err := ioutil.TempDir("", "")
	if err != nil {
		return errors.Wrap(err, "error creating temp dir for backup")
	}
	defer os.RemoveAll(tempDir)

	collector := &itemCollector{
		log:                   log,
		backupRequest:         backupRequest,
		discoveryHelper:       kb.discoveryHelper,
		dynamicFactory:        kb.dynamicFactory,
		cohabitatingResources: cohabitatingResources(),
		dir:                   tempDir,
		pageSize:              kb.clientPageSize,
	}
    // 从k8s集群获取所有要备份的资源,按照资源类型整理好
	items := collector.getAllItems()
    // ...
    // 更新本地Backup实例中的 totalItem信息
	backupRequest.Status.Progress = &velerov1api.BackupProgress{TotalItems: len(items)}
	patch := fmt.Sprintf(`{"status":{"progress":{"totalItems":%d}}}`, len(items))
    // 更新k8s集群中的Backup资源的TotalItem信息
	if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
		log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress.totalItems")
	}

	itemBackupper := &itemBackupper{
		backupRequest:           backupRequest,
		tarWriter:               tw,
		dynamicFactory:          kb.dynamicFactory,
		discoveryHelper:         kb.discoveryHelper,
		resticBackupper:         resticBackupper,
		resticSnapshotTracker:   newPVCSnapshotTracker(),
		volumeSnapshotterGetter: volumeSnapshotterGetter,
		itemHookHandler: &hook.DefaultItemHookHandler{
			PodCommandExecutor: kb.podCommandExecutor,
		},
	}

    // 用来表示进度的结构体
	type progressUpdate struct {
		totalItems, itemsBackedUp int
	}
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
                // 收到新的进度,更新本地临时变量(等下一次ticker触发时把最新值更新到k8s集群)
				lastUpdate = &val
			case <-ticker.C:
				if lastUpdate != nil {
                    // 每隔一段时间更新k8s集群的Backup资源的备份进度
					backupRequest.Status.Progress.TotalItems = lastUpdate.totalItems
					backupRequest.Status.Progress.ItemsBackedUp = lastUpdate.itemsBackedUp

					patch := fmt.Sprintf(`{"status":{"progress":{"totalItems":%d,"itemsBackedUp":%d}}}`, lastUpdate.totalItems, lastUpdate.itemsBackedUp)
					if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
						log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress")
					}
					lastUpdate = nil
				}
			}
		}
	}()

	backedUpGroupResources := map[schema.GroupResource]bool{}
	totalItems := len(items)

	for i, item := range items {
        // 把从临时文件获取的item信息格式化存储到本地临时文件中,每个item就是一个资源类型
		func() {
			var unstructured unstructured.Unstructured

			f, err := os.Open(item.path)
			if err != nil {
				log.WithError(errors.WithStack(err)).Error("Error opening file containing item")
				return
			}
			defer f.Close()
			defer os.Remove(f.Name())

			if err := json.NewDecoder(f).Decode(&unstructured); err != nil {
				log.WithError(errors.WithStack(err)).Error("Error decoding JSON from file")
				return
			}
			if backedUp := kb.backupItem(log, item.groupResource, itemBackupper, &unstructured, item.preferredGVR); backedUp {
				backedUpGroupResources[item.groupResource] = true
			}
		}()
		totalItems = len(backupRequest.BackedUpItems) + (len(items) - (i + 1))
		// 发送进度信息到channel(channel接收方会把进度定时更新到k8s集群Backup资源)
		update <- progressUpdate{
			totalItems:    totalItems,
			itemsBackedUp: len(backupRequest.BackedUpItems),
		}

	}
	quit <- struct{}{}
	// ...
	// 最后总结把最终结果更新到k8s集群的Backup资源
	backupRequest.Status.Progress.TotalItems = len(backupRequest.BackedUpItems)
	backupRequest.Status.Progress.ItemsBackedUp = len(backupRequest.BackedUpItems)

	patch = fmt.Sprintf(`{"status":{"progress":{"totalItems":%d,"itemsBackedUp":%d}}}`, len(backupRequest.BackedUpItems), len(backupRequest.BackedUpItems))
	if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
		log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress")
	}

	return nil
}
```