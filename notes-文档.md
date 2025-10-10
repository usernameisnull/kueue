## 文档
- 官方文档: https://kueue.sigs.k8s.io/docs/overview/, 官方文档站在`site/content`目录里
- DeepWiki: https://deepwiki.com/kubernetes-sigs/kueue
- 内部解读: [kueue源码摘要.md](kueue%E6%BA%90%E7%A0%81%E6%91%98%E8%A6%81.md)
### 翻译
- fungibility
```txt
Resource fungibility: if a resource flavor is fully utilized, Kueue can admit the job using a different flavor.
资源可互换性：如果某种资源类型已被完全使用，Kueue 可以使用另一种资源类型来接收作业。  
这里的 utilized 在上下文里就是指 “被占用/使用完”。  

```
- vanilla
```txt
You can install Kueue on top of a vanilla Kubernetes cluster.
在 Kueue 的安装说明中，"vanilla" 指的是原始的、未经修改的标准 Kubernetes 发行版。它不像 OpenShift、RKE 或 Tanzu 那样被特定厂商额外封装或深度定制。
```
- core design principle
```txt
A core design principle for Kueue is to avoid duplicating mature functionality in Kubernetes components and well-established third-party controllers. Autoscaling, pod-to-node scheduling and job lifecycle management are the responsibility of cluster-autoscaler, kube-scheduler and kube-controller-manager, respectively. Advanced admission control can be delegated to controllers such as gatekeeper.
Kueue 的一个核心设计原则是避免重复造轮子，不去实现 Kubernetes 组件和成熟的第三方控制器里已经具备的功能。自动扩缩容、Pod 到节点的调度以及作业生命周期管理，分别由 cluster-autoscaler、kube-scheduler 和 kube-controller-manager 负责。更高级的准入控制可以交给像 gatekeeper 这样的控制器来处理。
```
- cohort
  Cohort的意思是一群具有共同特征的人或事物, 在这个项目里
```txt
ClusterQueues can be grouped in cohorts. ClusterQueues that belong to the same cohort can borrow unused quota from each other.
To add a ClusterQueue to a cohort, specify the name of the cohort in the .spec.cohort field. All ClusterQueues that have a matching spec.cohort are part of the same cohort. If the spec.cohort field is empty, the ClusterQueue doesn’t belong to any cohort, and thus it cannot borrow quota from any other ClusterQueue

ClusterQueue 可以分组到队列集团（cohort）中。属于同一队列集团的 ClusterQueue 可以相互借用未使用的配额.
若要将 ClusterQueue 添加到某个队列集团，请在其规范（.spec.cohort）字段中指定该队列集团的名称。所有具有相同 spec.cohort值的 ClusterQueue 都属于同一个队列集团。如果 spec.cohort字段为空，则该 ClusterQueue 不属于任何队列集团，因此它无法从任何其他 ClusterQueue 借用配额.
```