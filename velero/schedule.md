# 版本
+ github: vmware-tanzu/velero
+ 分支: release-1.9

# 序
scheduleController 根据Schedule资源配置创建 Backup资源

# 源码
## TODO 监听
## Reconcile
```go
// pkg/controller/schedule_controller.go
func (c *scheduleReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	schedule := &velerov1.Schedule{}
	if err := c.Get(ctx, req.NamespacedName, schedule); err != nil {
		if apierrors.IsNotFound(err) {
			log.WithError(err).Error("schedule not found")
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, errors.Wrapf(err, "error getting schedule %s", req.String())
	}

	original := schedule.DeepCopy()

	// validation - even if the item is Enabled, we can't trust it
	// so re-validate
	currentPhase := schedule.Status.Phase

    // 解析下一次创建Backup资源的时间
	cronSchedule, errs := parseCronSchedule(schedule, c.logger)
	if len(errs) > 0 {
		schedule.Status.Phase = velerov1.SchedulePhaseFailedValidation
		schedule.Status.ValidationErrors = errs
	} else {
		schedule.Status.Phase = velerov1.SchedulePhaseEnabled
	}

	// update status if it's changed
	if currentPhase != schedule.Status.Phase {
		if err := c.Patch(ctx, schedule, client.MergeFrom(original)); err != nil {
			return ctrl.Result{}, errors.Wrapf(err, "error updating phase of schedule %s to %s", req.String(), schedule.Status.Phase)
		}
	}

	if schedule.Status.Phase != velerov1.SchedulePhaseEnabled {
		log.Debugf("the schedule's phase is %s, isn't %s, skip", schedule.Status.Phase, velerov1.SchedulePhaseEnabled)
		return ctrl.Result{}, nil
	}

	// 尝试根据schedule信息和 scheduleCron 决定是不是要创建一个 Backup资源 #ref submitBackupIfDue
	if err := c.submitBackupIfDue(ctx, schedule, cronSchedule); err != nil {
		return ctrl.Result{}, errors.Wrapf(err, "error running submitBackupIfDue for schedule %s", req.String())
	}

	return ctrl.Result{}, nil
}
```

### submitBackupIfDue
判断是否已经到时间创建新的`Backup`资源了
```go
// pkg/controller/schedule_controller.go
func (c *scheduleReconciler) submitBackupIfDue(ctx context.Context, item *velerov1.Schedule, cronSchedule cron.Schedule) error {
	var (
		now                = c.clock.Now()
        // 根据当前时间和算出来的下次运行时间 决定是否要开始创建‘Backup’了
		isDue, nextRunTime = getNextRunTime(item, cronSchedule, now)
		log                = c.logger.WithField("schedule", kubeutil.NamespaceAndName(item))
	)

	if !isDue { // 还没到时候,直接退出等待下次
		return nil
	}

	// 根据schedule信息创建Backup 资源
    // 注: 并没有判断是否有正在备份的backup任务
	backup := getBackup(item, now)
	if err := c.Create(ctx, backup); err != nil {
		return errors.Wrap(err, "error creating Backup")
	}

    // 更新Schedule资源信息
	original := item.DeepCopy()
	item.Status.LastBackup = &metav1.Time{Time: now}

	if err := c.Patch(ctx, item, client.MergeFrom(original)); err != nil {
		return errors.Wrapf(err, "error updating Schedule's LastBackup time to %v", item.Status.LastBackup)
	}

	return nil
}
```
