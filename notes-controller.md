## 入口
cmd/kueue/main.go

## controller
Flavor, ClusterQueue, LocalQueue, Workloads这几个核心资源的controller
- 目录: pkg/controller/core

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

## cache
怎么有一个巨大的cache目录,在pkg/cache下, controller