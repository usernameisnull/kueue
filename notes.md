## 部署
- baize只有一个kueue-controller-manager的deployment
### 部署原生的kueue
- 需要先把baize安装的所有的kueque的crd全部删除了,不然会失败     
    ```bash
     helm install kueue kueue/ --namespace kueue-system --create-namespace
    Error: INSTALLATION FAILED: Unable to continue with install: CustomResourceDefinition "clusterqueues.kueue.x-k8s.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-name" must equal "kueue": current value is "baize-agent"
    ```
- 安装命令   
    ```bash
    helm upgrade --install kueue kueue/ --namespace kueue-system --create-namespace \
    --set enableKueueViz=true \
    --set controllerManager.manager.image.repository="release-ci.daocloud.io/demo/kueue/kueue" \
    --set kueueViz.backend.image.repository="release-ci.daocloud.io/demo/kueue/kueueviz-backend" \
    --set kueueViz.frontend.image.repository="release-ci.daocloud.io/demo/kueue/kueueviz-frontend"
    ```
#### 安装kueueViz  
分成backend, frontend2个部分, 还有个ingress
- charts/kueue/values.yaml里设置为true
    ```cgo
    enableKueueViz: false
    ```
- 后端报错
    ```cgo
    2025/09/26 17:56:51 Starting pprof server on localhost:6060
    2025/09/26 17:56:51 Warning: Invalid origin 'frontend.kueueviz.local' rejected
    2025/09/26 17:56:51 Running in development mode
    panic: conflict settings: all origins disabled
    
    goroutine 1 [running]:
    github.com/gin-contrib/cors.newCors({0x0, {0x0, 0x0, 0x0}, 0x0, 0x0, {0xc0002999e0, 0x3, 0x3}, 0x0, ...})
        /go/pkg/mod/github.com/gin-contrib/cors@v1.7.6/config.go:45 +0x29c
    github.com/gin-contrib/cors.New({0x0, {0x0, 0x0, 0x0}, 0x0, 0x0, {0xc0002999e0, 0x3, 0x3}, 0x0, ...})
        /go/pkg/mod/github.com/gin-contrib/cors@v1.7.6/cors.go:204 +0x39
    kueueviz/middleware.SetupCORS()
        /go/middleware/cors.go:115 +0x99
    kueueviz/config.SetupGinEngine()
        /go/config/server.go:65 +0x8f
    main.main()
        /go/main.go:40 +0x73
    ```
#### 打包kueueViz backend
- make 命令: `make kueueviz-image-build`, 但是会同时打包backend和和frontend
- 自己的打包命令: `make kueueviz-backend-image-build`

#### 验证官方的0.13.4
```bash
helm upgrade --install kueue mine/kueue-0.13.4.tgz --namespace kueue-system --create-namespace \
    --set enableKueueViz=true \
    --set controllerManager.manager.image.repository="release-ci.daocloud.io/demo/kueue/kueue" \
    --set kueueViz.backend.image.repository="release-ci.daocloud.io/demo/kueue/kueueviz-backend" \
    --set kueueViz.frontend.image.repository="release-ci.daocloud.io/demo/kueue/kueueviz-frontend"
```
- 但是前端访问显示wss断开, 可能是ing对应的secret没有?
    ```cgo
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout backend.kueueviz.local.key \
      -out backend.kueueviz.local.crt \
      -subj "/CN=backend.kueueviz.local" \
      -addext "subjectAltName=DNS:backend.kueueviz.local"
      
    kubectl create secret tls kueueviz-backend-tls   --namespace=kueue-system   --cert=backend.kueueviz.local.crt   --key=backend.kueueviz.local.key
    
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout frontend.kueueviz.local.key \
      -out frontend.kueueviz.local.crt \
      -subj "/CN=frontend.kueueviz.local" \
      -addext "subjectAltName=DNS:frontend.kueueviz.local"
      
    kubectl create secret tls kueueviz-frontend-tls \
      --namespace=kueue-system \
      --cert=frontend.kueueviz.local.crt \
      --key=frontend.kueueviz.local.key
    ```
- 按照官方的操作,确实是可以访问的:  
  - site/content/en/docs/tasks/manage/enable_kubeviz.md  
  - https://github.com/kubernetes-sigs/kueue/blob/38404ca262630e0a759210a5333eb05d59a8081b/site/content/en/docs/tasks/manage/enable_kubeviz.md#L73  
  - 就是在我本地WSL2里执行`kie ctx a223`, `kie ns kueue-system`, 然后`kubectl port-forward svc/kueue-kueueviz-backend  -n kueue-system 8081:8080`再把frontend的deploy的env修改为:
    ```yaml
    - env:
        - name: REACT_APP_WEBSOCKET_URL
          value: ws://localhost:8081
    ```
- 为什么把REACT_APP_WEBSOCKET_URL改为ws://kueue-kueueviz-backend:8080不行呢, 连不上`kueue-kueueviz-backend:8080`这个svc?
## 入口
cmd/kueue/main.go:107

## charts
### crd
charts/kueue/templates/crd  
11个crd资源  

### kueueviz是干什么的?
就是个dashboard

## 本地检测与ut,e2e
https://kueue.sigs.k8s.io/docs/contribution_guidelines/testing/

## cmd/kueuectl
kubectl插件?

## manager入口
在main函数里  
mgr, err := ctrl.NewManager(kubeConfig, options)

## 打包并推送镜像
- IMAGE_REGISTRY=registry.example.com/my-user make image-local-push deploy
- IMAGE_REGISTRY=release-ci.daocloud.io/demo/kueue:mabing0902 PLATFORMS=linux/amd64 make image-local-push deploy
- 修改后的,用于自己打包的: make image-local-push-daocloud

## kueue-operator
https://github.com/openshift/kueue-operator, baize没有用到这个, 现在也只有2个star

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

## github workflow
### 发版
流水线过程中会提一个pr: https://github.com/kubernetes-sigs/krew-index/pull/4919/ 去修改plugins/kueue.yaml里的版本号码

### krew-release.yml
- [krew-release.yml](.github/workflows/krew-release.yml)这个是发布kubectl-kueue插件的,依赖[.krew.yaml](.krew.yaml)
- 流水线: https://github.com/kubernetes-sigs/kueue/actions/runs/18343139035/job/52243155690

### openvex.yaml
- [openvex.yaml](.github/workflows/openvex.yaml)这个是漏洞检测/安全相关的
- 流水线: https://github.com/kubernetes-sigs/kueue/actions/runs/18363028342/job/52310276240

### sbom.yaml
用到了[setup-bom](https://github.com/kubernetes-sigs/release-actions/tree/main/setup-bom)来引入了一个action.yml, 然后上传了一个kueue-v0.13.6.spdx.json这样的物料清单到release里
- [sbom.yaml](.github/workflows/sbom.yaml)调用了bom这个命令来生成项目的物料清单, 主要作用就是提高项目的透明度
- 流水线: https://github.com/kubernetes-sigs/kueue/actions/runs/18363034142/job/52310291827

### release页面的几个yaml
[NEW_RELEASE.md](.github/ISSUE_TEMPLATE/NEW_RELEASE.md)里提到有个步骤`make artifacts IMAGE_REGISTRY=registry.k8s.io/kueue GIT_TAG=$VERSION`, 好像都是手动发布的, 有点意外