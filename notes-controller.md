## 入口
cmd/kueue/main.go

## controller
Flavor, ClusterQueue, LocalQueue, Workloads这几个核心资源的controller
- 目录: pkg/controller/core
- 核心是pkg/controller/core/core.go? 因为main.go调用了SetupControllers
## controller特别的地方
可以识别是Create/Update等方法, 不像别的operator都是全部写在Reconcile里的, 这里的controller会根据事件类型进行不同的处理. 是通过每个资源的`SetupWithManager`方法做到的,比如
```go
func (r *ResourceFlavorReconciler) SetupWithManager(mgr ctrl.Manager, cfg *config.Configuration) error {
	h := cqHandler{
		cache: r.cache,
	}
	return builder.TypedControllerManagedBy[reconcile.Request](mgr).
		Named("resourceflavor_controller").
		WatchesRawSource(source.TypedKind(
			mgr.GetCache(),
			&kueue.ResourceFlavor{},
			&handler.TypedEnqueueRequestForObject[*kueue.ResourceFlavor]{},
			r,
		)).
		WithOptions(controller.Options{
			NeedLeaderElection:      ptr.To(false),
			MaxConcurrentReconciles: mgr.GetControllerOptions().GroupKindConcurrency[kueue.GroupVersion.WithKind("ResourceFlavor").GroupKind().String()],
		}).
		WatchesRawSource(source.Channel(r.cqUpdateCh, &h)).
		Complete(WithLeadingManager(mgr, r, &kueue.ResourceFlavor{}, cfg))
}
```
### TypedControllerManagedBy
controller-runtime里有个这样的例子: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/builder#example-Builder, 虽然用的ControllerManagedBy   
因为这里用的TypedControllerManagedBy[reconcile.Request], 所以跟ControllerManagedBy是一样的   

## CohortReconciler
用qoder解释`CohortReconciler`
```golang
func (r *CohortReconciler) SetupWithManager(mgr ctrl.Manager, cfg *config.Configuration) error {
	cqHandler := &cohortCqHandler{
		cache: r.cache,
	}
	return ctrl.NewControllerManagedBy(mgr).
		Named("cohort_controller").
		WatchesRawSource(source.TypedKind(
			mgr.GetCache(),
			&kueue.Cohort{},
			&handler.TypedEnqueueRequestForObject[*kueue.Cohort]{},
			r,
		)).
		WithOptions(controller.Options{
			NeedLeaderElection:      ptr.To(false),
			MaxConcurrentReconciles: mgr.GetControllerOptions().GroupKindConcurrency[kueue.GroupVersion.WithKind("Cohort").GroupKind().String()],
		}).
		WatchesRawSource(source.Channel(r.cqUpdateCh, cqHandler)).
		Complete(WithLeadingManager(mgr, r, &kueue.Cohort{}, cfg))
}
```
WatchesRawSource(source.Channel(r.cqUpdateCh, cqHandler)), 设置了对ClusterQueue更新的监视，当ClusterQueue发生变化时，会通过r.cqUpdateCh通道通知控制器。
### 实际上是通过以下机制来监视ClusterQueue更新的：
- cqUpdateCh通道: 这是一个chan event.GenericEvent类型的通道，在CohortReconciler结构体中定义。当ClusterQueue发生变化时，相关的控制器会向这个通道发送事件。
- NotifyClusterQueueUpdate方法: 这个方法负责向cqUpdateCh通道发送事件：
- 调用来源: NotifyClusterQueueUpdate方法在其他控制器（如ClusterQueueReconciler、ResourceFlavorReconciler等）中被调用。例如，在ClusterQueueReconciler中：
```golang
func (r *ClusterQueueReconciler) notifyWatchers(oldCQ, newCQ *kueue.ClusterQueue) {
    for _, w := range r.watchers {
        w.NotifyClusterQueueUpdate(oldCQ, newCQ)
    }
}
```
- 事件处理: 当事件通过cqUpdateCh通道发送时，cohortCqHandler的Generic方法会被调用：
- 触发协调: cohortCqHandler.Generic方法会获取ClusterQueue的祖先Cohort，然后为每个祖先Cohort添加一个协调请求，这会导致CohortReconciler.Reconcile方法被调用。
## cache
怎么有一个巨大的cache目录,在pkg/cache下, controller