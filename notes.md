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