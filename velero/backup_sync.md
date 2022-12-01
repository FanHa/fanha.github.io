# 版本
+ github: vmware-tanzu/velero
+ 分支: release-1.9

# 序
BackupSyncController 监听BackupStorageLocation资源,将BackupStorageLocation里的信息同步到本地,
比如:新建了一个BackupStorageLocation,这个BackupStorageLocation本身里面有很多别的方式创建的backup,这时需要将里面的backup的元信息同步到本地(即创建本地backup资源),后续velero就可以把这些backup当成是自己备份的东西使用了

# 代码
## backupSyncController.run
```go
// pkg/controller/backup_sync_controller.go
func (c *backupSyncController) run() {
    // 获取k8s集群里的所有BackupStorageLocation资源
	locationList, err := storage.ListBackupStorageLocations(context.Background(), c.kbClient, c.namespace)
	// ...

	// 排序(挑出默认的‘BackupStorageLocation’,后面第一个处理)
	for _, location := range locationList.Items {
		if location.Spec.Default {
			c.defaultBackupLocation = location.Name
			break
		}
	}
	locations := orderedBackupLocations(&locationList, c.defaultBackupLocation)

    // ...
    // 遍历所有‘BackupStorageLocation’
	for i, location := range locations {
		// ...
        // 从‘backupStorageLocation’的信息指向的存储地址中取出所有backups
		backupStore, err := c.backupStoreGetter.Get(&locations[i], pluginManager, log)
		res, err := backupStore.ListBackups()
		backupStoreBackups := sets.NewString(res...)

		// 从‘k8s’集群中取出所有本地有记录的backups
		clusterBackups, err := c.backupLister.Backups(c.namespace).List(labels.Everything())
		clusterBackupsSet := sets.NewString()
		for _, b := range clusterBackups {
			clusterBackupsSet.Insert(b.Name)
		}

        // 比对,返回所有需要同步到本地的backups
		backupsToSync := backupStoreBackups.Difference(clusterBackupsSet)


		// 遍历所有需要同步信息的backups
		for backupName := range backupsToSync {

			backup, err := backupStore.GetBackupMetadata(backupName)

			backup.Namespace = c.namespace
			backup.ResourceVersion = ""
			backup.Spec.StorageLocation = location.Name
			if backup.Labels == nil {
				backup.Labels = make(map[string]string)
			}
			backup.Labels[velerov1api.StorageLocationLabel] = label.GetValidName(backup.Spec.StorageLocation)

			// 根据backupStorageLocation里的信息创建本地backup
			backup, err = c.backupClient.Backups(backup.Namespace).Create(context.TODO(), backup, metav1.CreateOptions{})
			switch {
			case err != nil && kuberrs.IsAlreadyExists(err):
				log.Debug("Backup already exists in cluster")
				continue
			case err != nil && !kuberrs.IsAlreadyExists(err):
				log.WithError(errors.WithStack(err)).Error("Error syncing backup into cluster")
				continue
			default:
				log.Info("Successfully synced backup into cluster")
			}

		}

		// 更新‘backupStorageLocation’资源本身的状态
		statusPatch := client.MergeFrom(location.DeepCopy())
		location.Status.LastSyncedTime = &metav1.Time{Time: time.Now().UTC()}
		if err := c.kbClient.Patch(context.Background(), &locations[i], statusPatch); err != nil {
			log.WithError(errors.WithStack(err)).Error("Error patching backup location's last-synced time")
			continue
		}
	}
}

```