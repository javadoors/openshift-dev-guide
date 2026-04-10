  
# Kubeadm与CNI/CSI组件的关系
## 一、明确回答
**kubeadm不负责CNI/CSI等组件的安装部署**。

kubeadm的职责范围仅限于：
- Kubernetes核心组件的初始化
- 控制平面节点引导
- 工作节点加入集群
- 集群版本升级
- 证书管理
## 二、组件职责划分
### 2.1 Kubernetes核心组件（kubeadm负责）
```
┌─────────────────────────────────────────┐
│         Kubernetes Core Components       │
│        (kubeadm负责安装配置)              │
├─────────────────────────────────────────┤
│  • kube-apiserver                       │
│  • kube-controller-manager              │
│  • kube-scheduler                       │
│  • kubelet                              │
│  • kube-proxy                           │
│  • etcd                                 │
│  • CoreDNS                              │
└─────────────────────────────────────────┘
```
### 2.2 扩展组件（需要额外安装）
```
┌─────────────────────────────────────────┐
│      Kubernetes Extension Components     │
│        (kubeadm不负责安装)               │
├─────────────────────────────────────────┤
│  CNI (Container Network Interface)      │
│    • Calico                             │
│    • Flannel                            │
│    • Cilium                             │
│    • Weave Net                          │
├─────────────────────────────────────────┤
│  CSI (Container Storage Interface)      │
│    • Ceph CSI                           │
│    • NFS CSI                            │
│    • AWS EBS CSI                        │
│    • GCE PD CSI                         │
├─────────────────────────────────────────┤
│  CRI (Container Runtime Interface)      │
│    • containerd                         │
│    • CRI-O                              │
│    • Docker (via cri-dockerd)           │
└─────────────────────────────────────────┘
```
## 三、为什么kubeadm不负责这些组件？
### 3.1 设计理念
**Kubernetes的设计哲学是模块化和可插拔**：
1. **CNI**：网络插件有多种实现，用户根据需求选择（性能、网络策略、跨云支持等）
2. **CSI**：存储插件依赖具体基础设施，不同云平台有不同的存储方案
3. **CRI**：容器运行时有多种选择，用户根据场景选择
### 3.2 kubeadm的定位
kubeadm的设计目标是：
- **简单易用**：快速搭建符合最佳实践的Kubernetes集群
- **安全可靠**：自动处理证书、RBAC等安全配置
- **标准化**：遵循Kubernetes官方推荐配置

如果kubeadm负责所有组件的安装，会导致：
- 配置复杂度大幅增加
- 需要支持多种CNI/CSI实现
- 违背模块化设计原则
## 四、各组件的安装责任方
### 4.1 CNI插件安装
**责任方**：集群管理员或Cluster API的Addon Provider

**安装方式**：
```bash
# 方式1：手动安装
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# 方式2：通过Cluster API的ClusterResourceSet
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: calico-cni
  namespace: default
spec:
  clusterSelector:
    matchLabels:
      cni: calico
  resources:
  - kind: ConfigMap
    name: calico-manifest
```
### 4.2 CSI插件安装
**责任方**：集群管理员或基础设施Provider

**安装方式**：
```bash
# AWS EBS CSI Driver
kubectl apply -k github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable

# Ceph CSI
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/
```
### 4.3 CRI安装
**责任方**：Infrastructure Provider（如前面设计的方案）

**安装方式**：
- 在节点创建时通过userdata安装
- 通过基础设施Provider的扩展机制安装
## 五、Cluster API中的组件安装架构
在Cluster API生态中，组件安装有清晰的分工：
```
┌──────────────────────────────────────────────────────┐
│                  Cluster API Core                    │
│         (Cluster, Machine, MachineDeployment)        │
└──────────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┬─────────────┐
         ▼               ▼               ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Infrastructure│ │  Bootstrap   │ │ ControlPlane │ │    Addon     │
│   Provider   │ │   Provider   │ │   Provider   │ │   Provider   │
│              │ │  (Kubeadm)   │ │  (Kubeadm)   │ │              │
├──────────────┤ ├──────────────┤ ├──────────────┤ ├──────────────┤
│ • 创建节点   │ │ • kubeadm    │ │ • 控制平面   │ │ • CNI插件    │
│ • 安装CRI    │ │   init/join  │ │   初始化     │ │ • CSI插件    │
│ • 系统配置   │ │ • kubelet    │ │ • etcd       │ │ • 监控组件   │
│              │ │   配置       │ │   集群       │ │ • 日志组件   │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```
### 5.1 各Provider的职责

| Provider类型 | 负责组件 | 示例 |
|-------------|---------|------|
| Infrastructure Provider | CRI、系统配置、网络基础设施 | CAPA, CAPG, CAPV |
| Bootstrap Provider | Kubernetes核心组件、kubelet配置 | Kubeadm Bootstrap |
| Control Plane Provider | 控制平面初始化、etcd集群 | Kubeadm Control Plane |
| Addon Provider | CNI、CSI、监控、日志等扩展组件 | ClusterResourceSet |
### 5.2 ClusterResourceSet示例
Cluster API提供了`ClusterResourceSet`机制来安装扩展组件：
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: csi-addons
  namespace: default
spec:
  clusterSelector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  resources:
  - kind: ConfigMap
    name: csi-driver-manifest
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: csi-driver-manifest
  namespace: default
data:
  csi-driver.yaml: |
    apiVersion: storage.k8s.io/v1
    kind: CSIDriver
    metadata:
      name: ebs.csi.aws.com
    spec:
      attachRequired: true
      podInfoOnMount: false
```
## 六、完整的组件安装流程
### 6.1 时序图
```
时间轴  组件安装流程
  │
  ├─► 1. Infrastructure Provider创建节点
  │      • 创建VM/物理机
  │      • 安装CRI (containerd)
  │      • 配置系统环境
  │
  ├─► 2. Bootstrap Provider初始化节点
  │      • kubeadm init/join
  │      • 启动kubelet
  │      • 节点加入集群
  │
  ├─► 3. Control Plane Provider初始化控制平面
  │      • 启动API Server
  │      • 启动Controller Manager
  │      • 启动Scheduler
  │      • 初始化etcd
  │
  ├─► 4. Addon Provider安装扩展组件
  │      • 安装CNI插件
  │      • 安装CSI插件
  │      • 安装监控组件
  │      • 安装日志组件
  │
  ▼
     集群完全就绪
```
### 6.2 完整示例
```yaml
# 1. Cluster定义
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  labels:
    cni: calico
    csi: aws-ebs
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    serviceDomain: "cluster.local"
    services:
      cidrBlocks: ["10.128.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-cluster
---
# 2. CNI插件安装
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: calico-cni
spec:
  clusterSelector:
    matchLabels:
      cni: calico
  resources:
  - kind: ConfigMap
    name: calico-manifest
---
# 3. CSI插件安装
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: aws-ebs-csi
spec:
  clusterSelector:
    matchLabels:
      csi: aws-ebs
  resources:
  - kind: ConfigMap
    name: aws-ebs-csi-manifest
```
## 七、总结
### 关键要点
1. **kubeadm不负责CNI/CSI安装**：只负责Kubernetes核心组件
2. **CRI需要提前安装**：由Infrastructure Provider负责
3. **CNI/CSI由Addon Provider安装**：通过ClusterResourceSet等机制
4. **职责清晰分离**：符合Kubernetes模块化设计理念
### 推荐实践
1. **使用Cluster API**：通过ClusterResourceSet统一管理扩展组件
2. **分层设计**：Infrastructure Provider负责CRI，Addon Provider负责CNI/CSI
3. **声明式配置**：通过CRD定义组件配置，实现GitOps
4. **版本兼容**：确保各组件版本与Kubernetes版本兼容

这种清晰的职责划分使得整个系统更加模块化、可维护，也符合云原生生态的设计理念。

      
# 基于Cluster API安装Kubernetes集群与CNI/CSI的设计方案
## 一、问题分析
### 核心矛盾
- **kubeadm职责边界**：kubeadm只负责Kubernetes核心组件，不负责CNI/CSI安装
- **时序依赖**：CNI需要在节点就绪后、Pod网络配置前安装；CSI需要在存储基础设施就绪后安装
- **配置复杂度**：CNI/CSI有多种实现，配置项复杂且依赖基础设施
### 需要解决的问题
1. **时序控制**：确保CNI/CSI在正确的时机安装
2. **依赖管理**：处理CNI/CSI与基础设施的依赖关系
3. **配置管理**：灵活配置不同CNI/CSI实现
4. **生命周期管理**：支持升级、配置变更和删除
## 二、设计思路
### 方案选择：Addon Provider + ClusterResourceSet混合方案
我推荐使用**Addon Provider + ClusterResourceSet混合方案**，原因如下：
1. **职责清晰**：Addon Provider专门负责扩展组件管理
2. **声明式配置**：通过CRD定义CNI/CSI配置
3. **时序保证**：通过依赖管理和状态检查控制安装时机
4. **标准化**：符合Cluster API生态的设计模式
### 架构设计
```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster API Core                         │
│  (Cluster, Machine, KubeadmControlPlane, KubeadmConfig)    │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┬─────────────┐
        ▼                   ▼                   ▼             ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Infrastructure│   │   Bootstrap   │   │ Control Plane │   │     Addon     │
│   Provider    │   │   Provider    │   │   Provider    │   │   Provider    │
│               │   │   (Kubeadm)   │   │   (Kubeadm)   │   │               │
└───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │                   │
        │                   │                   │                   │
        └───────────────────┴───────────────────┴───────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │  ClusterResourceSet   │
                        │  + Addon Controllers  │
                        ├───────────────────────┤
                        │  • CNI Plugin         │
                        │  • CSI Driver         │
                        │  • Other Addons       │
                        └───────────────────────┘
```
## 三、详细设计方案
### 3.1 CRD设计
#### CNIConfig CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: cniconfigs.addons.cluster.x-k8s.io
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              type:
                type: string
                enum: [calico, flannel, cilium, weave, custom]
              version:
                type: string
                description: "CNI plugin version"
              config:
                type: object
                x-kubernetes-preserve-unknown-fields: true
                description: "CNI-specific configuration"
              installStrategy:
                type: object
                properties:
                  method:
                    type: string
                    enum: [manifest, helm, kustomize]
                    default: manifest
                  manifestURL:
                    type: string
                  helmChart:
                    type: object
                    properties:
                      repoURL:
                        type: string
                      chartName:
                        type: string
                      namespace:
                        type: string
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
              dependencies:
                type: object
                properties:
                  waitForControlPlane:
                    type: boolean
                    default: true
                  waitForNodes:
                    type: integer
                    description: "Minimum number of nodes to wait for"
                  preInstallChecks:
                    type: array
                    items:
                      type: object
                      properties:
                        type:
                          type: string
                          enum: [podReady, nodeReady, apiServerReady]
                        namespace:
                          type: string
                        selector:
                          type: string
          status:
            type: object
            properties:
              ready:
                type: boolean
              installedVersion:
                type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
```
#### CSIConfig CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: csiconfigs.addons.cluster.x-k8s.io
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              type:
                type: string
                enum: [aws-ebs, gce-pd, ceph, nfs, vsphere, custom]
              version:
                type: string
              config:
                type: object
                x-kubernetes-preserve-unknown-fields: true
              installStrategy:
                type: object
                properties:
                  method:
                    type: string
                    enum: [manifest, helm, kustomize]
                    default: manifest
                  manifestURL:
                    type: string
                  helmChart:
                    type: object
                    properties:
                      repoURL:
                        type: string
                      chartName:
                        type: string
                      namespace:
                        type: string
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
              infrastructureRef:
                type: object
                properties:
                  apiVersion:
                    type: string
                  kind:
                    type: string
                  name:
                    type: string
                description: "Reference to infrastructure-specific config"
              dependencies:
                type: object
                properties:
                  waitForCNI:
                    type: boolean
                    default: true
                  waitForStorageClass:
                    type: boolean
                  customChecks:
                    type: array
                    items:
                      type: object
                      properties:
                        type:
                          type: string
                        namespace:
                          type: string
                        selector:
                          type: string
          status:
            type: object
            properties:
              ready:
                type: boolean
              installedVersion:
                type: string
              conditions:
                type: array
                items:
                  type: object
                      properties:
                        type: string
                        status: string
                        reason: string
                        message: string
```
#### ClusterAddon CRD（聚合管理）
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusteraddons.addons.cluster.x-k8s.io
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              clusterRef:
                type: object
                properties:
                  apiVersion:
                    type: string
                  kind:
                    type: string
                  name:
                    type: string
                  namespace:
                    type: string
              cni:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: true
                  configRef:
                    type: object
                    properties:
                      apiVersion:
                        type: string
                      kind:
                        type: string
                      name:
                        type: string
              csi:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: true
                  configRef:
                    type: object
                    properties:
                      apiVersion:
                        type: string
                      kind:
                        type: string
                      name:
                        type: string
              additionalAddons:
                type: array
                items:
                  type: object
                  properties:
                    name:
                      type: string
                    configRef:
                      type: object
                      properties:
                        apiVersion:
                          type: string
                        kind:
                          type: string
                        name:
                          type: string
          status:
            type: object
            properties:
              ready:
                type: boolean
              cniReady:
                type: boolean
              csiReady:
                type: boolean
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
```
### 3.2 控制器设计
#### CNIConfig控制器
```go
type CNIConfigReconciler struct {
    client.Client
    Scheme    *runtime.Scheme
    Installer *CNIInstaller
}

func (r *CNIConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    cniConfig := &addonsv1beta1.CNIConfig{}
    if err := r.Get(ctx, req.NamespacedName, cniConfig); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 获取关联的Cluster
    cluster, err := r.getOwnerCluster(ctx, cniConfig)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 检查依赖条件
    if !r.checkDependencies(ctx, cluster, cniConfig) {
        conditions.MarkFalse(cniConfig, "DependenciesMet", "Waiting", "Dependencies not ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, r.Status().Update(ctx, cniConfig)
    }

    // 安装CNI
    if err := r.Installer.Install(ctx, cluster, cniConfig); err != nil {
        conditions.MarkFalse(cniConfig, "Installed", "InstallFailed", err.Error())
        return ctrl.Result{}, r.Status().Update(ctx, cniConfig)
    }

    // 验证安装
    if !r.verifyInstallation(ctx, cluster, cniConfig) {
        conditions.MarkFalse(cniConfig, "Ready", "Verifying", "Installation not ready")
        return ctrl.Result{RequeueAfter: 5 * time.Second}, r.Status().Update(ctx, cniConfig)
    }

    conditions.MarkTrue(cniConfig, "Ready")
    cniConfig.Status.Ready = true
    return ctrl.Result{}, r.Status().Update(ctx, cniConfig)
}

func (r *CNIConfigReconciler) checkDependencies(ctx context.Context, 
    cluster *clusterv1.Cluster, cniConfig *addonsv1beta1.CNIConfig) bool {
    
    // 检查控制平面是否就绪
    if cniConfig.Spec.Dependencies.WaitForControlPlane {
        if !cluster.Status.ControlPlaneReady {
            return false
        }
    }

    // 检查节点数量
    if cniConfig.Spec.Dependencies.WaitForNodes > 0 {
        nodes := &corev1.NodeList{}
        if err := r.List(ctx, nodes); err != nil {
            return false
        }
        if len(nodes.Items) < cniConfig.Spec.Dependencies.WaitForNodes {
            return false
        }
    }

    // 执行自定义检查
    for _, check := range cniConfig.Spec.Dependencies.PreInstallChecks {
        if !r.executeCheck(ctx, cluster, check) {
            return false
        }
    }

    return true
}
```
#### CNI安装器
```go
type CNIInstaller struct {
    client.Client
    HelmClient *helm.Client
}

func (i *CNIInstaller) Install(ctx context.Context, 
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    switch config.Spec.InstallStrategy.Method {
    case "manifest":
        return i.installFromManifest(ctx, cluster, config)
    case "helm":
        return i.installFromHelm(ctx, cluster, config)
    case "kustomize":
        return i.installFromKustomize(ctx, cluster, config)
    default:
        return fmt.Errorf("unsupported install method: %s", config.Spec.InstallStrategy.Method)
    }
}

func (i *CNIInstaller) installFromManifest(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    // 获取manifest URL
    manifestURL := config.Spec.InstallStrategy.ManifestURL
    if manifestURL == "" {
        manifestURL = i.getDefaultManifestURL(config)
    }

    // 下载manifest
    resp, err := http.Get(manifestURL)
    if err != nil {
        return fmt.Errorf("failed to download manifest: %w", err)
    }
    defer resp.Body.Close()

    manifestBytes, err := io.ReadAll(resp.Body)
    if err != nil {
        return fmt.Errorf("failed to read manifest: %w", err)
    }

    // 自定义配置
    customizedManifest, err := i.customizeManifest(manifestBytes, config)
    if err != nil {
        return fmt.Errorf("failed to customize manifest: %w", err)
    }

    // 应用manifest
    return i.applyManifest(ctx, cluster, customizedManifest)
}

func (i *CNIInstaller) installFromHelm(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    helmConfig := config.Spec.InstallStrategy.HelmChart
    
    // 获取目标集群的kubeconfig
    kubeconfig, err := i.getTargetClusterKubeconfig(ctx, cluster)
    if err != nil {
        return err
    }

    // 安装Helm chart
    return i.HelmClient.Install(ctx, &helm.InstallOptions{
        RepoURL:    helmConfig.RepoURL,
        ChartName:  helmConfig.ChartName,
        Namespace:  helmConfig.Namespace,
        Values:     helmConfig.Values,
        Kubeconfig: kubeconfig,
    })
}

func (i *CNIInstaller) getDefaultManifestURL(config *addonsv1beta1.CNIConfig) string {
    version := config.Spec.Version
    switch config.Spec.Type {
    case "calico":
        return fmt.Sprintf("https://raw.githubusercontent.com/projectcalico/calico/v%s/manifests/calico.yaml", version)
    case "flannel":
        return fmt.Sprintf("https://raw.githubusercontent.com/flannel-io/flannel/v%s/Documentation/kube-flannel.yml", version)
    case "cilium":
        return fmt.Sprintf("https://raw.githubusercontent.com/cilium/cilium/v%s/install/kubernetes/quick-install.yaml", version)
    default:
        return ""
    }
}
```
#### CSIConfig控制器
```go
type CSIConfigReconciler struct {
    client.Client
    Scheme    *runtime.Scheme
    Installer *CSIInstaller
}

func (r *CSIConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    csiConfig := &addonsv1beta1.CSIConfig{}
    if err := r.Get(ctx, req.NamespacedName, csiConfig); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 获取关联的Cluster
    cluster, err := r.getOwnerCluster(ctx, csiConfig)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 检查依赖条件
    if !r.checkDependencies(ctx, cluster, csiConfig) {
        conditions.MarkFalse(csiConfig, "DependenciesMet", "Waiting", "Dependencies not ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, r.Status().Update(ctx, csiConfig)
    }

    // 安装CSI
    if err := r.Installer.Install(ctx, cluster, csiConfig); err != nil {
        conditions.MarkFalse(csiConfig, "Installed", "InstallFailed", err.Error())
        return ctrl.Result{}, r.Status().Update(ctx, csiConfig)
    }

    // 验证安装
    if !r.verifyInstallation(ctx, cluster, csiConfig) {
        conditions.MarkFalse(csiConfig, "Ready", "Verifying", "Installation not ready")
        return ctrl.Result{RequeueAfter: 5 * time.Second}, r.Status().Update(ctx, csiConfig)
    }

    conditions.MarkTrue(csiConfig, "Ready")
    csiConfig.Status.Ready = true
    return ctrl.Result{}, r.Status().Update(ctx, csiConfig)
}

func (r *CSIConfigReconciler) checkDependencies(ctx context.Context,
    cluster *clusterv1.Cluster, csiConfig *addonsv1beta1.CSIConfig) bool {
    
    // 检查CNI是否就绪
    if csiConfig.Spec.Dependencies.WaitForCNI {
        cniReady := r.checkCNIReady(ctx, cluster)
        if !cniReady {
            return false
        }
    }

    // 检查基础设施特定的条件
    if csiConfig.Spec.InfrastructureRef != nil {
        if !r.checkInfrastructureReady(ctx, csiConfig.Spec.InfrastructureRef) {
            return false
        }
    }

    return true
}
```
#### ClusterAddon控制器（协调器）
```go
type ClusterAddonReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *ClusterAddonReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    clusterAddon := &addonsv1beta1.ClusterAddon{}
    if err := r.Get(ctx, req.NamespacedName, clusterAddon); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 获取关联的Cluster
    cluster := &clusterv1.Cluster{}
    if err := r.Get(ctx, client.ObjectKey{
        Namespace: clusterAddon.Spec.ClusterRef.Namespace,
        Name:      clusterAddon.Spec.ClusterRef.Name,
    }, cluster); err != nil {
        return ctrl.Result{}, err
    }

    // 协调CNI
    if clusterAddon.Spec.CNI.Enabled {
        if err := r.reconcileCNI(ctx, cluster, clusterAddon); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 协调CSI（依赖CNI）
    if clusterAddon.Spec.CSI.Enabled {
        if clusterAddon.Status.CNIReady || !clusterAddon.Spec.CNI.Enabled {
            if err := r.reconcileCSI(ctx, cluster, clusterAddon); err != nil {
                return ctrl.Result{}, err
            }
        }
    }

    // 更新整体状态
    clusterAddon.Status.Ready = clusterAddon.Status.CNIReady && clusterAddon.Status.CSIReady
    return ctrl.Result{}, r.Status().Update(ctx, clusterAddon)
}

func (r *ClusterAddonReconciler) reconcileCNI(ctx context.Context,
    cluster *clusterv1.Cluster, clusterAddon *addonsv1beta1.ClusterAddon) error {
    
    cniConfig := &addonsv1beta1.CNIConfig{}
    if err := r.Get(ctx, client.ObjectKey{
        Namespace: clusterAddon.Spec.CNI.ConfigRef.Namespace,
        Name:      clusterAddon.Spec.CNI.ConfigRef.Name,
    }, cniConfig); err != nil {
        return err
    }

    clusterAddon.Status.CNIReady = cniConfig.Status.Ready
    return nil
}
```
### 3.3 时序控制机制
#### 依赖检查器
```go
type DependencyChecker struct {
    client.Client
}

func (d *DependencyChecker) CheckControlPlaneReady(ctx context.Context, 
    cluster *clusterv1.Cluster) bool {
    return cluster.Status.ControlPlaneReady
}

func (d *DependencyChecker) CheckNodesReady(ctx context.Context,
    cluster *clusterv1.Cluster, minNodes int) bool {
    nodes := &corev1.NodeList{}
    if err := d.List(ctx, nodes); err != nil {
        return false
    }
    
    readyCount := 0
    for _, node := range nodes.Items {
        for _, condition := range node.Status.Conditions {
            if condition.Type == corev1.NodeReady && condition.Status == corev1.ConditionTrue {
                readyCount++
                break
            }
        }
    }
    
    return readyCount >= minNodes
}

func (d *DependencyChecker) CheckCNIReady(ctx context.Context,
    cluster *clusterv1.Cluster) bool {
    // 检查CNI pods是否就绪
    pods := &corev1.PodList{}
    if err := d.List(ctx, pods, client.InNamespace("kube-system")); err != nil {
        return false
    }
    
    for _, pod := range pods.Items {
        if strings.Contains(pod.Name, "calico") || 
           strings.Contains(pod.Name, "flannel") ||
           strings.Contains(pod.Name, "cilium") {
            if pod.Status.Phase != corev1.PodRunning {
                return false
            }
        }
    }
    
    return true
}

func (d *DependencyChecker) CheckAPIServerReady(ctx context.Context,
    cluster *clusterv1.Cluster) bool {
    // 检查API Server健康状态
    kubeconfig, err := d.getKubeconfig(ctx, cluster)
    if err != nil {
        return false
    }
    
    client, err := clientcmd.NewDefaultClientConfig(*kubeconfig, &clientcmd.ConfigOverrides{}).ClientConfig()
    if err != nil {
        return false
    }
    
    discoveryClient, err := discovery.NewDiscoveryClientForConfig(client)
    if err != nil {
        return false
    }
    
    _, err = discoveryClient.ServerVersion()
    return err == nil
}
```
## 四、实现流程
### 4.1 完整安装流程
```
用户创建Cluster资源
        │
        ▼
┌───────────────────────────────────┐
│ 1. Infrastructure Provider        │
│    - 创建节点资源                 │
│    - 安装CRI (containerd)         │
│    - 配置网络基础设施             │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 2. Control Plane Provider         │
│    - 初始化控制平面节点           │
│    - 启动API Server等核心组件     │
│    - 等待控制平面就绪             │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 3. Bootstrap Provider             │
│    - 工作节点加入集群             │
│    - 启动kubelet                  │
│    - 等待节点就绪                 │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 4. CNI Installation               │
│    - 检查控制平面就绪             │
│    - 检查节点数量满足要求         │
│    - 安装CNI插件│
│    - 等待CNI Pods就绪             │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 5. CSI Installation               │
│    - 检查CNI就绪                  │
│    - 检查基础设施存储就绪         │
│    - 安装CSI Driver               │
│    - 创建StorageClass             │
│    - 等待CSI Pods就绪             │
└───────────────────────────────────┘
        │
        ▼
    集群完全就绪
```
### 4.2 完整示例
```yaml
# 1. 定义CNI配置
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CNIConfig
metadata:
  name: calico-config
  namespace: default
spec:
  type: calico
  version: "3.26.0"
  config:
    typha:
      replicas: 2
    ipam:
      type: calico-ipam
    mtu: 1440
  installStrategy:
    method: manifest
    manifestURL: "https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml"
  dependencies:
    waitForControlPlane: true
    waitForNodes: 1
    preInstallChecks:
    - type: apiServerReady
---
# 2. 定义CSI配置
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CSIConfig
metadata:
  name: aws-ebs-csi-config
  namespace: default
spec:
  type: aws-ebs
  version: "1.20.0"
  config:
    controller:
      replicas: 2
    node:
      tolerateAllTaints: true
  installStrategy:
    method: helm
    helmChart:
      repoURL: "https://kubernetes-sigs.github.io/aws-ebs-csi-driver"
      chartName: "aws-ebs-csi-driver"
      namespace: "kube-system"
      values:
        controller:
          replicaCount: 2
        storageClasses:
        - name: gp3
          annotations:
            storageclass.kubernetes.io/is-default-class: "true"
          parameters:
            type: gp3
  dependencies:
    waitForCNI: true
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-cluster
---
# 3. 定义ClusterAddon
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: production-addons
  namespace: default
spec:
  clusterRef:
    apiVersion: cluster.x-k8s.io/v1beta1
    kind: Cluster
    name: production-cluster
    namespace: default
  cni:
    enabled: true
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: CNIConfig
      name: calico-config
  csi:
    enabled: true
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: CSIConfig
      name: aws-ebs-csi-config
---
# 4. 创建Cluster
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    serviceDomain: "cluster.local"
    services:
      cidrBlocks: ["10.128.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-cluster
---
# 5. 创建AWSCluster
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: production-cluster
  namespace: default
spec:
  region: us-west-2
  sshKeyName: my-key
  network:
    vpc:
      id: vpc-12345678
  controlPlaneEndpoint:
    host: ""
    port: 6443
---
# 6. 创建KubeadmControlPlane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: production-control-plane
  namespace: default
spec:
  replicas: 3
  version: "v1.28.0"
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachine
      name: control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
    joinConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
---
# 7. 创建MachineDeployment
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-workers
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
  template:
    spec:
      clusterName: production-cluster
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: worker-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachine
        name: worker
```
## 五、关键设计要点
### 5.1 时序保证
**问题**：确保CNI/CSI在正确的时机安装

**解决方案**：
1. **依赖检查**：通过DependencyChecker检查前置条件
2. **状态轮询**：定期检查依赖状态，满足条件后触发安装
3. **条件标记**：通过Conditions机制报告安装进度
```go
// 时序控制流程
func (r *CNIConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 检查控制平面就绪
    if !r.checkControlPlaneReady(ctx, cluster) {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 2. 检查节点数量
    if !r.checkNodesReady(ctx, cluster, cniConfig.Spec.Dependencies.WaitForNodes) {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 3. 安装CNI
    if err := r.installCNI(ctx, cluster, cniConfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 验证安装
    if !r.verifyCNI(ctx, cluster, cniConfig) {
        return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
    }
    
    return ctrl.Result{}, nil
}
```
### 5.2 配置管理
**问题**：如何灵活配置不同CNI/CSI实现

**解决方案**：
1. **CRD定义配置**：通过CNIConfig/CSIConfig CRD定义配置
2. **模板引擎**：使用Go template自定义manifest
3. **Helm集成**：支持Helm values配置
```go
// 自定义manifest
func (i *CNIInstaller) customizeManifest(manifest []byte, 
    config *addonsv1beta1.CNIConfig) ([]byte, error) {
    
    tmpl, err := template.New("cni").Parse(string(manifest))
    if err != nil {
        return nil, err
    }
    
    var buf bytes.Buffer
    if err := tmpl.Execute(&buf, config.Spec.Config); err != nil {
        return nil, err
    }
    
    return buf.Bytes(), nil
}
```
### 5.3 错误处理
**问题**：安装失败如何处理

**解决方案**：
1. **重试机制**：控制器自动重试失败的安装
2. **回滚支持**：支持回滚到之前的配置版本
3. **状态报告**：通过Conditions报告详细错误信息
```go
func (r *CNIConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 安装失败处理
    if err := r.installCNI(ctx, cluster, cniConfig); err != nil {
        conditions.MarkFalse(cniConfig, "Installed", "InstallFailed", err.Error())
        
        // 指数退避重试
        retryCount := cniConfig.Annotations["retry-count"]
        count, _ := strconv.Atoi(retryCount)
        count++
        cniConfig.Annotations["retry-count"] = strconv.Itoa(count)
        
        delay := time.Duration(math.Pow(2, float64(count))) * time.Second
        if delay > 5*time.Minute {
            delay = 5 * time.Minute
        }
        
        return ctrl.Result{RequeueAfter: delay}, r.Status().Update(ctx, cniConfig)
    }
    
    return ctrl.Result{}, nil
}
```
### 5.4 升级策略
**问题**：如何处理CNI/CSI的升级

**解决方案**：
1. **滚动升级**：通过修改版本触发升级
2. **配置变更**：修改配置触发重新安装
3. **版本兼容检查**：验证新版本兼容性
```yaml
spec:
  version: "3.27.0"  # 修改版本触发升级
  upgradeStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```
## 六、扩展性设计
### 6.1 支持多种安装方式
```go
type InstallStrategy interface {
    Install(ctx context.Context, cluster *clusterv1.Cluster, config interface{}) error
    Upgrade(ctx context.Context, cluster *clusterv1.Cluster, config interface{}) error
    Uninstall(ctx context.Context, cluster *clusterv1.Cluster, config interface{}) error
}

type ManifestStrategy struct{}

type HelmStrategy struct {
    HelmClient *helm.Client
}

type KustomizeStrategy struct{}
```
### 6.2 支持自定义Addon
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: production-addons
spec:
  additionalAddons:
  - name: monitoring
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: AddonConfig
      name: prometheus-config
  - name: logging
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: AddonConfig
      name: elasticsearch-config
```
### 6.3 支持多云环境
```yaml
# AWS环境
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CSIConfig
metadata:
  name: aws-ebs-csi
spec:
  type: aws-ebs
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-cluster

---
# GCP环境
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CSIConfig
metadata:
  name: gce-pd-csi
spec:
  type: gce-pd
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: GCPCluster
    name: production-cluster
```
## 七、与ClusterResourceSet的对比
### ClusterResourceSet方案
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: cni-csi-addons
spec:
  clusterSelector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
  resources:
  - kind: ConfigMap
    name: calico-manifest
  - kind: ConfigMap
    name: aws-ebs-csi-manifest
```
### 对比分析
| 特性 | ClusterResourceSet | Addon Provider方案 |
|------|-------------------|-------------------|
| 时序控制 | ❌ 无 | ✅ 有 |
| 依赖管理 | ❌ 无 | ✅ 有 |
| 状态验证 | ❌ 无 | ✅ 有 |
| 配置灵活性 | ⚠️ 有限 | ✅ 高 |
| 错误处理 | ❌ 简单 | ✅ 完善 |
| 升级支持 | ❌ 无 | ✅ 有 |
| 回滚支持 | ❌ 无 | ✅ 有 |
### 推荐方案
**混合使用**：
1. **简单场景**：使用ClusterResourceSet快速部署
2. **复杂场景**：使用Addon Provider获得完整功能
```yaml
# 简单场景：使用ClusterResourceSet
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: simple-addons
spec:
  clusterSelector:
    matchLabels:
      environment: dev
  resources:
  - kind: ConfigMap
    name: calico-manifest

---
# 复杂场景：使用Addon Provider
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: production-addons
spec:
  clusterRef:
    name: production-cluster
  cni:
    enabled: true
    configRef:
      name: calico-config
```
## 八、总结
### 方案优势
1. **职责清晰**：Addon Provider专门负责扩展组件管理
2. **时序保证**：通过依赖检查确保正确的安装顺序
3. **配置灵活**：支持多种安装方式和自定义配置
4. **生命周期完整**：支持安装、升级、回滚、删除
5. **可扩展性强**：支持多种CNI/CSI实现和多云环境
### 实施建议
1. **分阶段实施**：
   - 第一阶段：实现CNIConfig和CSIConfig控制器
   - 第二阶段：实现ClusterAddon协调器
   - 第三阶段：集成Helm和Kustomize支持
2. **测试策略**：
   - 单元测试：测试各个控制器的逻辑
   - 集成测试：测试完整的安装流程
   - E2E测试：测试多场景下的行为
3. **监控告警**：
   - 监控CNI/CSI安装状态
   - 监控Pod健康状态
   - 设置安装超时告警

这个设计方案提供了一个完整的、可扩展的CNI/CSI安装解决方案，符合Cluster API的设计理念，能够满足生产环境的需求。

    
# 基于Cluster API的Addon Provider设计方案
## 一、架构概览
### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Cluster API Core                        │
│            (Cluster, Machine, ControlPlane)                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ 触发安装
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Addon Provider                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ ClusterAddon │  │  CNIConfig   │  │  CSIConfig   │     │
│  │  Controller  │  │  Controller  │  │  Controller  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                  │                  │             │
│         └──────────────────┴──────────────────┘             │
│                            │                                 │
│                            ▼                                 │
│                  ┌──────────────────┐                       │
│                  │  Install Engine  │                       │
│                  │  (Helm/Manifest) │                       │
│                  └──────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    目标Kubernetes集群
```
### 1.2 核心设计理念
1. **声明式配置**：通过CRD定义Addon配置
2. **时序控制**：通过依赖管理确保正确的安装顺序
3. **状态驱动**：通过状态机管理Addon生命周期
4. **可扩展性**：支持多种Addon类型和安装方式
## 二、核心概念设计
### 2.1 资源模型
```
Cluster (CAPI Core)
    │
    ├─── ClusterAddon (聚合管理)
    │        │
    │        ├─── CNIConfig (网络插件配置)
    │        │
    │        ├─── CSIConfig (存储插件配置)
    │        │
    │        └─── CustomAddon (自定义插件)
    │
    └─── Infrastructure Provider (基础设施)
```
### 2.2 CRD设计（简化版）
#### ClusterAddon CRD
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: my-cluster-addons
spec:
  clusterRef:
    name: my-cluster
  
  cni:
    enabled: true
    type: calico
    version: "3.26.0"
    config:
      mtu: 1440
      ipam: calico-ipam
  
  csi:
    enabled: true
    type: aws-ebs
    version: "1.20.0"
    config:
      controller:
        replicas: 2
      storageClass:
        default: true

status:
  phase: Installing        # Pending/Installing/Ready/Failed
  cni:
    ready: false
    version: "3.26.0"
  csi:
    ready: false
    version: "1.20.0"
```
## 三、安装流程设计
### 3.1 状态机设计
```
┌─────────┐
│ Pending │ ──检查依赖──> ┌────────────┐
└─────────┘               │ Installing │
                          └────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
              ┌─────────┐            ┌─────────┐
              │  Ready  │            │ Failed  │
              └─────────┘            └─────────┘
                    │                       │
                    └───────────────────────┘
                            重试/回滚
```
### 3.2 安装时序图
```
时间轴  安装流程
  │
  ├─► 1. Cluster创建完成
  │      └─ 控制平面就绪
  │
  ├─► 2. ClusterAddon控制器触发
  │      ├─ 获取Cluster状态
  │      ├─ 检查控制平面就绪
  │      └─ 初始化Addon状态
  │
  ├─► 3. CNI安装阶段
  │      ├─ 检查前置条件（控制平面就绪）
  │      ├─ 渲染配置模板
  │      ├─ 应用Manifest/Helm Chart
  │      ├─ 等待Pod就绪
  │      └─ 更新CNI状态为Ready
  │
  ├─► 4. CSI安装阶段（依赖CNI）
  │      ├─ 检查前置条件（CNI就绪）
  │      ├─ 渲染配置模板
  │      ├─ 应用Manifest/Helm Chart
  │      ├─ 创建StorageClass
  │      ├─ 等待Pod就绪
  │      └─ 更新CSI状态为Ready
  │
  ▼
  集群完全就绪
```
## 四、控制器设计思路
### 4.1 ClusterAddon控制器（核心）
**职责**：
- 监听Cluster资源变化
- 协调CNI/CSI安装顺序
- 管理整体Addon状态

**核心逻辑**：
```
Reconcile循环:
  1. 获取关联的Cluster资源
  2. 检查Cluster是否就绪
  3. 按顺序安装Addon：
     - 先安装CNI（依赖控制平面）
     - 再安装CSI（依赖CNI）
  4. 更新整体状态
```
**关键代码结构**：
```go
func (r *ClusterAddonReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取资源
    addon := getClusterAddon(ctx, req)
    cluster := getCluster(ctx, addon.Spec.ClusterRef)
    
    // 2. 检查集群就绪
    if !cluster.Status.ControlPlaneReady {
        return requeue(10s)
    }
    
    // 3. 安装CNI
    if addon.Spec.CNI.Enabled && !addon.Status.CNI.Ready {
        if err := installCNI(ctx, cluster, addon.Spec.CNI); err != nil {
            return handleError(err)
        }
        addon.Status.CNI.Ready = true
    }
    
    // 4. 安装CSI（依赖CNI）
    if addon.Spec.CSI.Enabled && addon.Status.CNI.Ready && !addon.Status.CSI.Ready {
        if err := installCSI(ctx, cluster, addon.Spec.CSI); err != nil {
            return handleError(err)
        }
        addon.Status.CSI.Ready = true
    }
    
    // 5. 更新状态
    updateStatus(ctx, addon)
    return done()
}
```
### 4.2 安装引擎设计
**支持多种安装方式**：
```
InstallEngine (接口)
    │
    ├─── ManifestInstaller
    │      └─ kubectl apply -f
    │
    ├─── HelmInstaller
    │      └─ helm install/upgrade
    │
    └─── KustomizeInstaller
           └─ kustomize build | kubectl apply
```
**核心接口**：
```go
type InstallEngine interface {
    Install(ctx context.Context, cluster *Cluster, config AddonConfig) error
    Upgrade(ctx context.Context, cluster *Cluster, config AddonConfig) error
    Uninstall(ctx context.Context, cluster *Cluster, config AddonConfig) error
    Verify(ctx context.Context, cluster *Cluster, config AddonConfig) (bool, error)
}
```
## 五、依赖管理设计
### 5.1 依赖检查机制
```
依赖检查器
    │
    ├─► CheckControlPlaneReady()
    │      └─ cluster.Status.ControlPlaneReady == true
    │
    ├─► CheckNodesReady(minNodes)
    │      └─ Ready节点数 >= minNodes
    │
    ├─► CheckCNIReady()
    │      └─ CNI Pods全部Running
    │
    └─► CheckCustomCondition()
           └─ 自定义检查逻辑
```
### 5.2 时序保证策略
**策略1：状态轮询**
- 定期检查前置条件
- 满足条件后触发安装
- 优点：简单可靠
- 缺点：有延迟

**策略2：事件驱动**
- 监听相关资源变化
- 状态变化立即触发
- 优点：响应快
- 缺点：实现复杂

**推荐：混合策略**
- 事件驱动 + 定期轮询
- 兼顾响应速度和可靠性
## 六、配置管理设计
### 6.1 配置来源
```
配置来源
    │
    ├─► 内置模板
    │      └─ 常见CNI/CSI的标准配置
    │
    ├─► 用户自定义
    │      └─ 用户提供的ConfigMap/Secret
    │
    └─► 动态生成
           └─ 根据集群信息自动生成
```
### 6.2 配置渲染流程
```
原始配置
    │
    ├─► 加载基础模板
    │
    ├─► 合并用户配置
    │
    ├─► 注入集群信息
    │      ├─ Pod CIDR
    │      ├─ Service CIDR
    │      └─ API Server地址
    │
    └─► 生成最终Manifest
```
## 七、错误处理设计
### 7.1 错误分类
```
错误类型
    │
    ├─► 临时性错误
    │      ├─ 网络超时
    │      ├─ API限流
    │      └─ 策略：自动重试
    │
    ├─► 配置错误
    │      ├─ 参数无效
    │      ├─ 版本不兼容
    │      └─ 策略：用户修正
    │
    └─► 致命错误
           ├─ 资源不足
           ├─ 权限问题
           └─ 策略：人工介入
```
### 7.2 重试策略
```
重试策略
    │
    ├─► 指数退避
    │      └─ 1s -> 2s -> 4s -> 8s -> ...
    │
    ├─► 最大重试次数
    │      └─ 超过次数标记为Failed
    │
    └─► 重置条件
           └─ 配置变更时重置计数
```
## 八、完整示例
### 8.1 最小化配置
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  controlPlaneRef:
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
  infrastructureRef:
    kind: AWSCluster
    name: my-cluster
---
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: my-cluster-addons
spec:
  clusterRef:
    name: my-cluster
  cni:
    enabled: true
    type: calico
    version: "3.26.0"
  csi:
    enabled: true
    type: aws-ebs
    version: "1.20.0"
```
### 8.2 完整配置
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: production-addons
spec:
  clusterRef:
    name: production-cluster
  
  cni:
    enabled: true
    type: calico
    version: "3.26.0"
    installStrategy:
      method: helm
      helmChart:
        repoURL: https://projectcalico.docs.tigera.io/charts
        chartName: tigera-operator
        namespace: tigera-operator
    config:
      installation:
        cni:
          type: Calico
        calicoNetwork:
          ipPools:
          - cidr: 192.168.0.0/16
            encapsulation: VXLANCrossSubnet
          mtu: 1440
    dependencies:
      waitForControlPlane: true
      waitForNodes: 1
  
  csi:
    enabled: true
    type: aws-ebs
    version: "1.20.0"
    installStrategy:
      method: helm
      helmChart:
        repoURL: https://kubernetes-sigs.github.io/aws-ebs-csi-driver
        chartName: aws-ebs-csi-driver
        namespace: kube-system
    config:
      controller:
        replicaCount: 2
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
      storageClasses:
      - name: gp3
        annotations:
          storageclass.kubernetes.io/is-default-class: "true"
        parameters:
          type: gp3
          fsType: ext4
    dependencies:
      waitForCNI: true
```
## 九、关键设计决策
### 9.1 为什么使用独立CRD而不是ClusterResourceSet？
| 维度 | ClusterResourceSet | 独立Addon CRD |
|------|-------------------|--------------|
| 时序控制 | ❌ 无 | ✅ 有 |
| 状态管理 | ❌ 无 | ✅ 有 |
| 依赖管理 | ❌ 无 | ✅ 有 |
| 错误处理 | ❌ 简单 | ✅ 完善 |
| 配置灵活性 | ⚠️ 有限 | ✅ 高 |

**结论**：独立CRD提供更完整的生命周期管理能力
### 9.2 为什么CNI和CSI使用同一个ClusterAddon CRD？
**优势**：
1. **统一管理**：一个资源管理所有Addon
2. **依赖明确**：CNI和CSI的依赖关系清晰
3. **状态一致**：整体状态一目了然

**劣势**：
1. **灵活性降低**：无法独立管理单个Addon
2. **耦合度高**：一个失败影响整体

**决策**：提供两种选择
- 简单场景：使用ClusterAddon聚合管理
- 复杂场景：使用独立的CNIConfig/CSIConfig
### 9.3 为什么选择Helm作为主要安装方式？
**原因**：
1. **生态成熟**：大多数CNI/CSI提供官方Helm Chart
2. **配置灵活**：通过values.yaml灵活配置
3. **版本管理**：支持版本回滚
4. **依赖管理**：Helm自动处理Chart依赖
## 十、实施路线图
### 阶段一：MVP（最小可行产品）
```
目标：实现基本的CNI/CSI安装能力

实现内容：
├─ ClusterAddon CRD
├─ 基础控制器逻辑
├─ Manifest安装方式
└─ 状态管理

支持场景：
├─ Calico CNI安装
└─ AWS EBS CSI安装
```
### 阶段二：功能完善

```
目标：增强配置能力和可靠性

实现内容：
├─ Helm安装方式
├─ 依赖检查机制
├─ 错误重试策略
└─ 配置模板引擎

支持场景：
├─ 多种CNI（Calico/Flannel/Cilium）
├─ 多种CSI（AWS/GCP/vSphere）
└─ 自定义配置
```
### 阶段三：生产就绪
```
目标：生产环境可用

实现内容：
├─ 升级支持
├─ 回滚机制
├─ 监控告警
└─ 性能优化

支持场景：
├─ 版本升级
├─ 配置变更
└─ 故障恢复
```
## 十一、总结
### 核心设计要点
1. **声明式配置**：通过CRD定义Addon配置，符合Kubernetes理念
2. **时序控制**：通过依赖检查确保CNI在CSI之前安装
3. **状态驱动**：通过状态机管理Addon生命周期
4. **可扩展性**：支持多种Addon类型和安装方式
### 关键优势
1. **职责清晰**：Addon Provider专注于扩展组件管理
2. **自动化**：自动处理安装顺序和依赖关系
3. **可靠性**：完善的错误处理和重试机制
4. **灵活性**：支持多种配置方式和安装策略
### 适用场景
1. **多集群管理**：统一管理多个集群的Addon
2. **GitOps**：通过Git管理Addon配置
3. **自动化运维**：自动处理Addon生命周期
4. **多云环境**：支持不同云平台的CNI/CSI

这个设计方案提供了一个清晰、简洁、可扩展的Addon管理解决方案，符合Cluster API的设计理念，能够满足生产环境的需求。

# 基于Cluster API的Addon Provider设计方案
## 一、整体架构设计
### 1.1 架构图
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                      Cluster API Core                           │
│           (Cluster, Machine, KubeadmControlPlane)              │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Infrastructure  │  │   Bootstrap     │  │  Control Plane  │
│    Provider     │  │    Provider     │  │    Provider     │
│                 │  │   (Kubeadm)     │  │   (Kubeadm)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └────────────────────┴────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Addon Provider                             │
│                  (CNI/CSI安装管理)                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ ClusterAddon │  │  CNIConfig   │  │  CSIConfig   │         │
│  │  Controller  │  │  Controller  │  │  Controller  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│         │                  │                  │                │
│         └──────────────────┴──────────────────┘                │
│                            │                                    │
│                            ▼                                    │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Addon Installer Engine                     │   │
│  │  • Manifest Installer  • Helm Installer                │   │
│  │  • Kustomize Installer  • Custom Installer             │   │
│  └────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Dependency Checker                         │   │
│  │  • Control Plane Ready  • Nodes Ready                  │   │
│  │  • CNI Ready (for CSI)  • Infrastructure Ready         │   │
│  └────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```
### 1.2 设计原则
- 声明式配置：通过CRD定义Addon配置
- 时序保证：通过依赖检查确保正确的安装顺序
- 状态可观测：通过Conditions机制报告安装状态
- 可扩展性：支持多种安装方式和Addon类型
- 幂等性：支持重复执行而不产生副作用
## 二、CRD设计
### 2.1 ClusterAddon CRD（核心聚合资源）
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusteraddons.addons.cluster.x-k8s.io
  labels:
    cluster.x-k8s.io/provider: addon
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            type: object
            required:
            - clusterRef
            properties:
              clusterRef:
                type: object
                required:
                - name
                properties:
                  apiVersion:
                    type: string
                    default: cluster.x-k8s.io/v1beta1
                  kind:
                    type: string
                    default: Cluster
                  name:
                    type: string
                  namespace:
                    type: string
              cni:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: true
                  configRef:
                    type: object
                    required:
                    - name
                    properties:
                      apiVersion:
                        type: string
                      kind:
                        type: string
                        default: CNIConfig
                      name:
                        type: string
                      namespace:
                        type: string
                  installTimeout:
                    type: string
                    default: "10m"
                  verifyTimeout:
                    type: string
                    default: "5m"
              csi:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: false
                  configRef:
                    type: object
                    required:
                    - name
                    properties:
                      apiVersion:
                        type: string
                      kind:
                        type: string
                        default: CSIConfig
                      name:
                        type: string
                      namespace:
                        type: string
                  installTimeout:
                    type: string
                    default: "10m"
                  verifyTimeout:
                    type: string
                    default: "5m"
                  dependsOn:
                    type: array
                    items:
                      type: string
                    default: ["cni"]
              additionalAddons:
                type: array
                items:
                  type: object
                  required:
                  - name
                  - configRef
                  properties:
                    name:
                      type: string
                    enabled:
                      type: boolean
                      default: true
                    configRef:
                      type: object
                      required:
                      - name
                      properties:
                        apiVersion:
                          type: string
                        kind:
                          type: string
                        name:
                          type: string
                        namespace:
                          type: string
                    dependsOn:
                      type: array
                      items:
                        type: string
                    installTimeout:
                      type: string
                      default: "10m"
                    verifyTimeout:
                      type: string
                      default: "5m"
              installStrategy:
                type: object
                properties:
                  parallel:
                    type: boolean
                    default: false
                  failurePolicy:
                    type: string
                    enum: [Stop, Continue, Ignore]
                    default: Stop
                  retryPolicy:
                    type: object
                    properties:
                      maxRetries:
                        type: integer
                        default: 3
                      backoff:
                        type: object
                        properties:
                          initialDelay:
                            type: string
                            default: "10s"
                          maxDelay:
                            type: string
                            default: "5m"
                          multiplier:
                            type: number
                            default: 2.0
          status:
            type: object
            properties:
              ready:
                type: boolean
              phase:
                type: string
                enum: [Pending, Installing, Installed, Failed, Upgrading]
              cniStatus:
                type: object
                properties:
                  ready:
                    type: boolean
                  phase:
                    type: string
                  version:
                    type: string
                  lastUpdated:
                    type: string
                  conditions:
                    type: array
                    items:
                      type: object
                      properties:
                        type: string
                        status: string
                        reason: string
                        message: string
                        lastTransitionTime:
                          format: date-time
                          type: string
              csiStatus:
                type: object
                properties:
                  ready:
                    type: boolean
                  phase:
                    type: string
                  version:
                    type: string
                  lastUpdated:
                    type: string
                  conditions:
                    type: array
                    items:
                      type: object
                      properties:
                        type: string
                        status: string
                        reason: string
                        message: string
                        lastTransitionTime:
                          format: date-time
                          type: string
              additionalAddonsStatus:
                type: object
                additionalProperties:
                  type: object
                  properties:
                    ready:
                      type: boolean
                    phase:
                      type: string
                    version:
                      type: string
                    lastUpdated:
                      type: string
                    conditions:
                      type: array
                      items:
                        type: object
                        properties:
                          type: string
                          status: string
                          reason: string
                          message: string
                          lastTransitionTime:
                            format: date-time
                            type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
                    lastTransitionTime:
                      format: date-time
                      type: string
    additionalPrinterColumns:
    - name: Cluster
      type: string
      jsonPath: .spec.clusterRef.name
    - name: Ready
      type: boolean
      jsonPath: .status.ready
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```
### 2.2 CNIConfig CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: cniconfigs.addons.cluster.x-k8s.io
  labels:
    cluster.x-k8s.io/provider: addon
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            type: object
            required:
            - type
            properties:
              type:
                type: string
                enum: [calico, flannel, cilium, weave, canal, custom]
              version:
                type: string
                description: "CNI plugin version"
              clusterNetwork:
                type: object
                properties:
                  podCIDR:
                    type: string
                  serviceCIDR:
                    type: string
              config:
                type: object
                x-kubernetes-preserve-unknown-fields: true
                description: "CNI-specific configuration"
              installStrategy:
                type: object
                properties:
                  method:
                    type: string
                    enum: [manifest, helm, kustomize, custom]
                    default: manifest
                  manifest:
                    type: object
                    properties:
                      url:
                        type: string
                      configMapRef:
                        type: object
                        properties:
                          name:
                            type: string
                          namespace:
                            type: string
                          key:
                            type: string
                            default: "manifest.yaml"
                  helm:
                    type: object
                    properties:
                      repoURL:
                        type: string
                      chartName:
                        type: string
                      releaseName:
                        type: string
                      namespace:
                        type: string
                        default: "kube-system"
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
                      valuesFrom:
                        type: array
                        items:
                          type: object
                          properties:
                            configMapKeyRef:
                              type: object
                              properties:
                                name:
                                  type: string
                                key:
                                  type: string
                                namespace:
                                  type: string
                            secretKeyRef:
                              type: object
                              properties:
                                name:
                                  type: string
                                key:
                                  type: string
                                namespace:
                                  type: string
                  kustomize:
                    type: object
                    properties:
                      path:
                        type: string
                      githubRepo:
                        type: string
                      ref:
                        type: string
                  custom:
                    type: object
                    properties:
                      installerImage:
                        type: string
                      installerCommand:
                        type: array
                        items:
                          type: string
                      env:
                        type: object
                        additionalProperties:
                          type: string
                      volumes:
                        type: array
                        items:
                          type: object
                          x-kubernetes-preserve-unknown-fields: true
              dependencies:
                type: object
                properties:
                  waitForControlPlane:
                    type: boolean
                    default: true
                  waitForNodes:
                    type: integer
                    description: "Minimum number of ready nodes"
                    default: 1
                  preInstallChecks:
                    type: array
                    items:
                      type: object
                      properties:
                        type:
                          type: string
                          enum: [podReady, nodeReady, apiServerReady, custom]
                        namespace:
                          type: string
                        labelSelector:
                          type: string
                        expectedCount:
                          type: integer
                        customCheck:
                          type: object
                          properties:
                            command:
                              type: array
                              items:
                                type: string
                            expectedOutput:
                              type: string
              verification:
                type: object
                properties:
                  checkPods:
                    type: boolean
                    default: true
                  podLabels:
                    type: object
                    additionalProperties:
                      type: string
                  podNamespace:
                    type: string
                    default: "kube-system"
                  minReadyPods:
                    type: integer
                  checkNetworkConnectivity:
                    type: boolean
                    default: true
                  testPodSpec:
                    type: object
                    x-kubernetes-preserve-unknown-fields: true
          status:
            type: object
            properties:
              ready:
                type: boolean
              phase:
                type: string
                enum: [Pending, Installing, Installed, Verifying, Failed, Upgrading]
              installedVersion:
                type: string
              installStartTime:
                format: date-time
                type: string
              installCompletionTime:
                format: date-time
                type: string
              retryCount:
                type: integer
              lastError:
                type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
                    lastTransitionTime:
                      format: date-time
                      type: string
    additionalPrinterColumns:
    - name: Type
      type: string
      jsonPath: .spec.type
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Ready
      type: boolean
      jsonPath: .status.ready
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```
### 2.3 CSIConfig CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: csiconfigs.addons.cluster.x-k8s.io
  labels:
    cluster.x-k8s.io/provider: addon
spec:
  group: addons.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            type: object
            required:
            - type
            properties:
              type:
                type: string
                enum: [aws-ebs, aws-efs, gce-pd, azure-disk, azure-file, 
                       ceph, nfs, vsphere, local, custom]
              version:
                type: string
              config:
                type: object
                x-kubernetes-preserve-unknown-fields: true
              installStrategy:
                type: object
                properties:
                  method:
                    type: string
                    enum: [manifest, helm, kustomize, custom]
                    default: manifest
                  manifest:
                    type: object
                    properties:
                      url:
                        type: string
                      configMapRef:
                        type: object
                        properties:
                          name:
                            type: string
                          namespace:
                            type: string
                          key:
                            type: string
                            default: "manifest.yaml"
                  helm:
                    type: object
                    properties:
                      repoURL:
                        type: string
                      chartName:
                        type: string
                      releaseName:
                        type: string
                      namespace:
                        type: string
                        default: "kube-system"
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
                      valuesFrom:
                        type: array
                        items:
                          type: object
                          properties:
                            configMapKeyRef:
                              type: object
                              properties:
                                name:
                                  type: string
                                key:
                                  type: string
                                namespace:
                                  type: string
                            secretKeyRef:
                              type: object
                              properties:
                                name:
                                  type: string
                                key:
                                  type: string
                                namespace:
                                  type: string
                  kustomize:
                    type: object
                    properties:
                      path:
                        type: string
                      githubRepo:
                        type: string
                      ref:
                        type: string
                  custom:
                    type: object
                    properties:
                      installerImage:
                        type: string
                      installerCommand:
                        type: array
                        items:
                          type: string
                      env:
                        type: object
                        additionalProperties:
                          type: string
                      volumes:
                        type: array
                        items:
                          type: object
                          x-kubernetes-preserve-unknown-fields: true
              storageClasses:
                type: array
                items:
                  type: object
                  required:
                  - name
                  properties:
                    name:
                      type: string
                    isDefault:
                      type: boolean
                      default: false
                    annotations:
                      type: object
                      additionalProperties:
                        type: string
                    parameters:
                      type: object
                      additionalProperties:
                        type: string
                    reclaimPolicy:
                      type: string
                      enum: [Retain, Delete]
                      default: Delete
                    volumeBindingMode:
                      type: string
                      enum: [Immediate, WaitForFirstConsumer]
                      default: Immediate
                    allowedTopologies:
                      type: array
                      items:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
              volumeSnapshotClasses:
                type: array
                items:
                  type: object
                  required:
                  - name
                  properties:
                    name:
                      type: string
                    driver:
                      type: string
                    parameters:
                      type: object
                      additionalProperties:
                        type: string
                    deletionPolicy:
                      type: string
                      enum: [Retain, Delete]
                      default: Delete
              dependencies:
                type: object
                properties:
                  waitForCNI:
                    type: boolean
                    default: true
                  waitForNodes:
                    type: integer
                    default: 1
                  infrastructureRef:
                    type: object
                    properties:
                      apiVersion:
                        type: string
                      kind:
                        type: string
                      name:
                        type: string
                      namespace:
                        type: string
                  preInstallChecks:
                    type: array
                    items:
                      type: object
                      properties:
                        type:
                          type: string
                          enum: [podReady, nodeReady, storageReady, custom]
                        namespace:
                          type: string
                        labelSelector:
                          type: string
                        expectedCount:
                          type: integer
              verification:
                type: object
                properties:
                  checkPods:
                    type: boolean
                    default: true
                  podLabels:
                    type: object
                    additionalProperties:
                      type: string
                  podNamespace:
                    type: string
                    default: "kube-system"
                  minReadyPods:
                    type: integer
                  checkStorageConnectivity:
                    type: boolean
                    default: true
                  testPvcSpec:
                    type: object
                    x-kubernetes-preserve-unknown-fields: true
          status:
            type: object
            properties:
              ready:
                type: boolean
              phase:
                type: string
                enum: [Pending, Installing, Installed, Verifying, Failed, Upgrading]
              installedVersion:
                type: string
              installedStorageClasses:
                type: array
                items:
                  type: string
              installStartTime:
                format: date-time
                type: string
              installCompletionTime:
                format: date-time
                type: string
              retryCount:
                type: integer
              lastError:
                type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
                    lastTransitionTime:
                      format: date-time
                      type: string
    additionalPrinterColumns:
    - name: Type
      type: string
      jsonPath: .spec.type
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Ready
      type: boolean
      jsonPath: .status.ready
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```
## 三、控制器实现
### 3.1 ClusterAddon控制器
```go
package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    "sigs.k8s.io/controller-runtime/pkg/source"
    
    addonsv1beta1 "sigs.k8s.io/cluster-api-addon-provider/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

const (
    clusterAddonFinalizer = "addons.cluster.x-k8s.io/finalizer"
)

type ClusterAddonReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    
    CNIInstaller *CNIInstaller
    CSIInstaller *CSIInstaller
    DependencyChecker *DependencyChecker
}

func (r *ClusterAddonReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("clusteraddon", req.NamespacedName)
    
    clusterAddon := &addonsv1beta1.ClusterAddon{}
    if err := r.Get(ctx, req.NamespacedName, clusterAddon); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    if !clusterAddon.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, clusterAddon, log)
    }
    
    if !controllerutil.ContainsFinalizer(clusterAddon, clusterAddonFinalizer) {
        controllerutil.AddFinalizer(clusterAddon, clusterAddonFinalizer)
        if err := r.Update(ctx, clusterAddon); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return r.reconcileNormal(ctx, clusterAddon, log)
}

func (r *ClusterAddonReconciler) reconcileNormal(ctx context.Context,
    clusterAddon *addonsv1beta1.ClusterAddon, log logr.Logger) (ctrl.Result, error) {
    
    cluster := &clusterv1.Cluster{}
    clusterKey := types.NamespacedName{
        Namespace: clusterAddon.Spec.ClusterRef.Namespace,
        Name:      clusterAddon.Spec.ClusterRef.Name,
    }
    if err := r.Get(ctx, clusterKey, cluster); err != nil {
        if errors.IsNotFound(err) {
            log.Info("Cluster not found, waiting")
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
        return ctrl.Result{}, err
    }
    
    if !cluster.Status.ControlPlaneReady {
        log.Info("Control plane not ready, waiting")
        conditions.MarkFalse(clusterAddon, "ControlPlaneReady", "Waiting", "Control plane not ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, r.Status().Update(ctx, clusterAddon)
    }
    
    var allErrs []error
    requeue := false
    
    if clusterAddon.Spec.CNI.Enabled {
        result, err := r.reconcileCNI(ctx, cluster, clusterAddon, log)
        if err != nil {
            allErrs = append(allErrs, err)
        }
        if !result.IsZero() {
            requeue = true
        }
    }
    
    if clusterAddon.Spec.CSI.Enabled {
        if clusterAddon.Status.CNIStatus.Ready || !clusterAddon.Spec.CNI.Enabled {
            result, err := r.reconcileCSI(ctx, cluster, clusterAddon, log)
            if err != nil {
                allErrs = append(allErrs, err)
            }
            if !result.IsZero() {
                requeue = true
            }
        } else {
            log.Info("Waiting for CNI to be ready before installing CSI")
            requeue = true
        }
    }
    
    for _, addon := range clusterAddon.Spec.AdditionalAddons {
        if addon.Enabled {
            if r.checkDependenciesMet(ctx, cluster, clusterAddon, addon.DependsOn) {
                result, err := r.reconcileAdditionalAddon(ctx, cluster, clusterAddon, addon, log)
                if err != nil {
                    allErrs = append(allErrs, err)
                }
                if !result.IsZero() {
                    requeue = true
                }
            } else {
                log.Info("Dependencies not met for addon", "addon", addon.Name)
                requeue = true
            }
        }
    }
    
    clusterAddon.Status.Ready = r.isAllAddonsReady(clusterAddon)
    if clusterAddon.Status.Ready {
        conditions.MarkTrue(clusterAddon, "Ready")
        clusterAddon.Status.Phase = "Installed"
    } else {
        clusterAddon.Status.Phase = "Installing"
    }
    
    if err := r.Status().Update(ctx, clusterAddon); err != nil {
        allErrs = append(allErrs, err)
    }
    
    if len(allErrs) > 0 {
        return ctrl.Result{}, fmt.Errorf("errors during reconciliation: %v", allErrs)
    }
    
    if requeue {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    return ctrl.Result{}, nil
}

func (r *ClusterAddonReconciler) reconcileCNI(ctx context.Context,
    cluster *clusterv1.Cluster, clusterAddon *addonsv1beta1.ClusterAddon,
    log logr.Logger) (ctrl.Result, error) {
    
    cniConfig := &addonsv1beta1.CNIConfig{}
    cniKey := types.NamespacedName{
        Namespace: clusterAddon.Spec.CNI.ConfigRef.Namespace,
        Name:      clusterAddon.Spec.CNI.ConfigRef.Name,
    }
    if err := r.Get(ctx, cniKey, cniConfig); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to get CNI config: %w", err)
    }
    
    if !cniConfig.Status.Ready {
        log.Info("CNI not ready, waiting")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    clusterAddon.Status.CNIStatus = addonsv1beta1.AddonStatus{
        Ready:       cniConfig.Status.Ready,
        Phase:       cniConfig.Status.Phase,
        Version:     cniConfig.Status.InstalledVersion,
        LastUpdated: time.Now().Format(time.RFC3339),
        Conditions:  cniConfig.Status.Conditions,
    }
    
    return ctrl.Result{}, nil
}

func (r *ClusterAddonReconciler) reconcileCSI(ctx context.Context,
    cluster *clusterv1.Cluster, clusterAddon *addonsv1beta1.ClusterAddon,
    log logr.Logger) (ctrl.Result, error) {
    
    csiConfig := &addonsv1beta1.CSIConfig{}
    csiKey := types.NamespacedName{
        Namespace: clusterAddon.Spec.CSI.ConfigRef.Namespace,
        Name:      clusterAddon.Spec.CSI.ConfigRef.Name,
    }
    if err := r.Get(ctx, csiKey, csiConfig); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to get CSI config: %w", err)
    }
    
    if !csiConfig.Status.Ready {
        log.Info("CSI not ready, waiting")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    clusterAddon.Status.CSIStatus = addonsv1beta1.AddonStatus{
        Ready:       csiConfig.Status.Ready,
        Phase:       csiConfig.Status.Phase,
        Version:     csiConfig.Status.InstalledVersion,
        LastUpdated: time.Now().Format(time.RFC3339),
        Conditions:  csiConfig.Status.Conditions,
    }
    
    return ctrl.Result{}, nil
}

func (r *ClusterAddonReconciler) reconcileDelete(ctx context.Context,
    clusterAddon *addonsv1beta1.ClusterAddon, log logr.Logger) (ctrl.Result, error) {
    
    log.Info("Deleting ClusterAddon")
    
    if controllerutil.ContainsFinalizer(clusterAddon, clusterAddonFinalizer) {
        controllerutil.RemoveFinalizer(clusterAddon, clusterAddonFinalizer)
        if err := r.Update(ctx, clusterAddon); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

func (r *ClusterAddonReconciler) isAllAddonsReady(clusterAddon *addonsv1beta1.ClusterAddon) bool {
    if clusterAddon.Spec.CNI.Enabled && !clusterAddon.Status.CNIStatus.Ready {
        return false
    }
    if clusterAddon.Spec.CSI.Enabled && !clusterAddon.Status.CSIStatus.Ready {
        return false
    }
    for _, addon := range clusterAddon.Spec.AdditionalAddons {
        if addon.Enabled {
            if status, ok := clusterAddon.Status.AdditionalAddonsStatus[addon.Name]; !ok || !status.Ready {
                return false
            }
        }
    }
    return true
}

func (r *ClusterAddonReconciler) checkDependenciesMet(ctx context.Context,
    cluster *clusterv1.Cluster, clusterAddon *addonsv1beta1.ClusterAddon,
    dependencies []string) bool {
    
    for _, dep := range dependencies {
        switch dep {
        case "cni":
            if !clusterAddon.Status.CNIStatus.Ready {
                return false
            }
        case "csi":
            if !clusterAddon.Status.CSIStatus.Ready {
                return false
            }
        default:
            if status, ok := clusterAddon.Status.AdditionalAddonsStatus[dep]; !ok || !status.Ready {
                return false
            }
        }
    }
    return true
}

func (r *ClusterAddonReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&addonsv1beta1.ClusterAddon{}).
        Watches(
            &source.Kind{Type: &clusterv1.Cluster{}},
            handler.EnqueueRequestsFromMapFunc(r.clusterToClusterAddon),
        ).
        Watches(
            &source.Kind{Type: &addonsv1beta1.CNIConfig{}},
            handler.EnqueueRequestsFromMapFunc(r.cniConfigToClusterAddon),
        ).
        Watches(
            &source.Kind{Type: &addonsv1beta1.CSIConfig{}},
            handler.EnqueueRequestsFromMapFunc(r.csiConfigToClusterAddon),
        ).
        Complete(r)
}

func (r *ClusterAddonReconciler) clusterToClusterAddon(obj client.Object) []reconcile.Request {
    cluster := obj.(*clusterv1.Cluster)
    
    addons := &addonsv1beta1.ClusterAddonList{}
    if err := r.List(context.Background(), addons); err != nil {
        return nil
    }
    
    var requests []reconcile.Request
    for _, addon := range addons.Items {
        if addon.Spec.ClusterRef.Name == cluster.Name &&
            addon.Spec.ClusterRef.Namespace == cluster.Namespace {
            requests = append(requests, reconcile.Request{
                NamespacedName: types.NamespacedName{
                    Namespace: addon.Namespace,
                    Name:      addon.Name,
                },
            })
        }
    }
    return requests
}
```
### 3.2 CNIConfig控制器
```go
package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    
    addonsv1beta1 "sigs.k8s.io/cluster-api-addon-provider/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

const (
    cniConfigFinalizer = "addons.cluster.x-k8s.io/cni-finalizer"
)

type CNIConfigReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    
    Installer         *CNIInstaller
    DependencyChecker *DependencyChecker
    Verifier          *CNIVerifier
}

func (r *CNIConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("cniconfig", req.NamespacedName)
    
    cniConfig := &addonsv1beta1.CNIConfig{}
    if err := r.Get(ctx, req.NamespacedName, cniConfig); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    if !cniConfig.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, cniConfig, log)
    }
    
    if !controllerutil.ContainsFinalizer(cniConfig, cniConfigFinalizer) {
        controllerutil.AddFinalizer(cniConfig, cniConfigFinalizer)
        if err := r.Update(ctx, cniConfig); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return r.reconcileNormal(ctx, cniConfig, log)
}

func (r *CNIConfigReconciler) reconcileNormal(ctx context.Context,
    cniConfig *addonsv1beta1.CNIConfig, log logr.Logger) (ctrl.Result, error) {
    
    if cniConfig.Status.Phase == "" {
        cniConfig.Status.Phase = "Pending"
    }
    
    cluster, err := r.getOwnerCluster(ctx, cniConfig)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if cluster == nil {
        log.Info("Waiting for cluster to be set as owner")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    if !r.DependencyChecker.CheckDependencies(ctx, cluster, cniConfig) {
        log.Info("Dependencies not met, waiting")
        conditions.MarkFalse(cniConfig, "DependenciesMet", "Waiting", "Dependencies not ready")
        cniConfig.Status.Phase = "Pending"
        return ctrl.Result{RequeueAfter: 10 * time.Second}, r.Status().Update(ctx, cniConfig)
    }
    
    if cniConfig.Status.InstalledVersion == "" {
        log.Info("Installing CNI", "type", cniConfig.Spec.Type, "version", cniConfig.Spec.Version)
        cniConfig.Status.Phase = "Installing"
        cniConfig.Status.InstallStartTime = time.Now().Format(time.RFC3339)
        conditions.MarkFalse(cniConfig, "Installed", "Installing", "CNI installation in progress")
        _ = r.Status().Update(ctx, cniConfig)
        
        if err := r.Installer.Install(ctx, cluster, cniConfig); err != nil {
            log.Error(err, "Failed to install CNI")
            conditions.MarkFalse(cniConfig, "Installed", "InstallFailed", err.Error())
            cniConfig.Status.Phase = "Failed"
            cniConfig.Status.LastError = err.Error()
            cniConfig.Status.RetryCount++
            return ctrl.Result{RequeueAfter: r.calculateBackoff(cniConfig.Status.RetryCount)}, 
                r.Status().Update(ctx, cniConfig)
        }
        
        cniConfig.Status.InstalledVersion = cniConfig.Spec.Version
        conditions.MarkTrue(cniConfig, "Installed")
    }
    
    log.Info("Verifying CNI installation")
    cniConfig.Status.Phase = "Verifying"
    _ = r.Status().Update(ctx, cniConfig)
    
    if !r.Verifier.Verify(ctx, cluster, cniConfig) {
        log.Info("CNI verification not ready, waiting")
        conditions.MarkFalse(cniConfig, "Ready", "Verifying", "CNI verification in progress")
        return ctrl.Result{RequeueAfter: 5 * time.Second}, r.Status().Update(ctx, cniConfig)
    }
    
    log.Info("CNI successfully installed and verified")
    cniConfig.Status.Phase = "Installed"
    cniConfig.Status.Ready = true
    cniConfig.Status.InstallCompletionTime = time.Now().Format(time.RFC3339)
    conditions.MarkTrue(cniConfig, "Ready")
    
    return ctrl.Result{}, r.Status().Update(ctx, cniConfig)
}

func (r *CNIConfigReconciler) reconcileDelete(ctx context.Context,
    cniConfig *addonsv1beta1.CNIConfig, log logr.Logger) (ctrl.Result, error) {
    
    log.Info("Deleting CNIConfig")
    
    if controllerutil.ContainsFinalizer(cniConfig, cniConfigFinalizer) {
        if cniConfig.Status.InstalledVersion != "" {
            cluster, err := r.getOwnerCluster(ctx, cniConfig)
            if err == nil && cluster != nil {
                if err := r.Installer.Uninstall(ctx, cluster, cniConfig); err != nil {
                    log.Error(err, "Failed to uninstall CNI")
                    return ctrl.Result{}, err
                }
            }
        }
        
        controllerutil.RemoveFinalizer(cniConfig, cniConfigFinalizer)
        if err := r.Update(ctx, cniConfig); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

func (r *CNIConfigReconciler) getOwnerCluster(ctx context.Context,
    cniConfig *addonsv1beta1.CNIConfig) (*clusterv1.Cluster, error) {
    
    for _, ownerRef := range cniConfig.OwnerReferences {
        if ownerRef.Kind == "Cluster" {
            cluster := &clusterv1.Cluster{}
            if err := r.Get(ctx, types.NamespacedName{
                Namespace: cniConfig.Namespace,
                Name:      ownerRef.Name,
            }, cluster); err != nil {
                return nil, err
            }
            return cluster, nil
        }
    }
    return nil, nil
}

func (r *CNIConfigReconciler) calculateBackoff(retryCount int) time.Duration {
    initialDelay := 10 * time.Second
    maxDelay := 5 * time.Minute
    multiplier := 2.0
    
    delay := time.Duration(float64(initialDelay) * float64(retryCount) * multiplier)
    if delay > maxDelay {
        delay = maxDelay
    }
    return delay
}

func (r *CNIConfigReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&addonsv1beta1.CNIConfig{}).
        Complete(r)
}
```
### 3.3 CNI安装器
```go
package installer

import (
    "bytes"
    "context"
    "fmt"
    "io"
    "net/http"
    "strings"
    "text/template"
    
    "helm.sh/helm/v3/pkg/action"
    "helm.sh/helm/v3/pkg/cli"
    "helm.sh/helm/v3/pkg/release"
    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime/serializer/yaml"
    "k8s.io/client-go/kubernetes/scheme"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/kustomize/api/krusty"
    "sigs.k8s.io/kustomize/api/resmap"
    "sigs.k8s.io/kustomize/kyaml/filesys"
    
    addonsv1beta1 "sigs.k8s.io/cluster-api-addon-provider/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

type CNIInstaller struct {
    client.Client
    HelmActionConfig *action.Configuration
}

func (i *CNIInstaller) Install(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    switch config.Spec.InstallStrategy.Method {
    case "manifest":
        return i.installFromManifest(ctx, cluster, config)
    case "helm":
        return i.installFromHelm(ctx, cluster, config)
    case "kustomize":
        return i.installFromKustomize(ctx, cluster, config)
    case "custom":
        return i.installFromCustom(ctx, cluster, config)
    default:
        return fmt.Errorf("unsupported install method: %s", config.Spec.InstallStrategy.Method)
    }
}

func (i *CNIInstaller) installFromManifest(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    var manifestBytes []byte
    var err error
    
    if config.Spec.InstallStrategy.Manifest.URL != "" {
        manifestBytes, err = i.downloadManifest(config.Spec.InstallStrategy.Manifest.URL)
        if err != nil {
            return fmt.Errorf("failed to download manifest: %w", err)
        }
    } else if config.Spec.InstallStrategy.Manifest.ConfigMapRef != nil {
        manifestBytes, err = i.getManifestFromConfigMap(ctx, config)
        if err != nil {
            return fmt.Errorf("failed to get manifest from configmap: %w", err)
        }
    } else {
        manifestBytes, err = i.getDefaultManifest(config)
        if err != nil {
            return fmt.Errorf("failed to get default manifest: %w", err)
        }
    }
    
    customizedManifest, err := i.customizeManifest(manifestBytes, config)
    if err != nil {
        return fmt.Errorf("failed to customize manifest: %w", err)
    }
    
    return i.applyManifest(ctx, cluster, customizedManifest)
}

func (i *CNIInstaller) installFromHelm(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    helmConfig := config.Spec.InstallStrategy.Helm
    
    settings := cli.New()
    actionConfig := new(action.Configuration)
    if err := actionConfig.Init(settings.RESTClientGetter(), helmConfig.Namespace, "secret", func(format string, v ...interface{}) {}); err != nil {
        return fmt.Errorf("failed to initialize helm action config: %w", err)
    }
    
    install := action.NewInstall(actionConfig)
    install.ReleaseName = helmConfig.ReleaseName
    install.Namespace = helmConfig.Namespace
    install.Version = config.Spec.Version
    
    chartPath, err := install.LocateChart(helmConfig.RepoURL+"/"+helmConfig.ChartName, settings)
    if err != nil {
        return fmt.Errorf("failed to locate chart: %w", err)
    }
    
    values, err := i.mergeHelmValues(ctx, helmConfig.Values, helmConfig.ValuesFrom)
    if err != nil {
        return fmt.Errorf("failed to merge helm values: %w", err)
    }
    
    _, err = install.Run(chartPath, values)
    if err != nil {
        return fmt.Errorf("failed to install helm chart: %w", err)
    }
    
    return nil
}

func (i *CNIInstaller) installFromKustomize(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    kustomizeConfig := config.Spec.InstallStrategy.Kustomize
    
    var resMap resmap.ResMap
    var err error
    
    if kustomizeConfig.Path != "" {
        fsys := filesys.MakeFsInMemory()
        resMap, err = krusty.MakeKustomizer(krusty.MakeDefaultOptions()).Run(fsys, kustomizeConfig.Path)
    } else if kustomizeConfig.GithubRepo != "" {
        url := kustomizeConfig.GithubRepo
        if kustomizeConfig.Ref != "" {
            url = fmt.Sprintf("%s?ref=%s", url, kustomizeConfig.Ref)
        }
        fsys := filesys.MakeFsInMemory()
        resMap, err = krusty.MakeKustomizer(krusty.MakeDefaultOptions()).Run(fsys, url)
    } else {
        return fmt.Errorf("kustomize path or github repo must be specified")
    }
    
    if err != nil {
        return fmt.Errorf("failed to run kustomize: %w", err)
    }
    
    manifest, err := resMap.AsYaml()
    if err != nil {
        return fmt.Errorf("failed to convert kustomize output to yaml: %w", err)
    }
    
    return i.applyManifest(ctx, cluster, manifest)
}

func (i *CNIInstaller) installFromCustom(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    customConfig := config.Spec.InstallStrategy.Custom
    
    kubeconfig, err := i.getTargetClusterKubeconfig(ctx, cluster)
    if err != nil {
        return fmt.Errorf("failed to get target cluster kubeconfig: %w", err)
    }
    
    job := i.createInstallerJob(customConfig, kubeconfig, config)
    if err := i.Create(ctx, job); err != nil {
        return fmt.Errorf("failed to create installer job: %w", err)
    }
    
    return i.waitForJobCompletion(ctx, job)
}

func (i *CNIInstaller) Uninstall(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    switch config.Spec.InstallStrategy.Method {
    case "helm":
        return i.uninstallFromHelm(ctx, cluster, config)
    default:
        return i.uninstallFromManifest(ctx, cluster, config)
    }
}

func (i *CNIInstaller) uninstallFromHelm(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) error {
    
    helmConfig := config.Spec.InstallStrategy.Helm
    
    settings := cli.New()
    actionConfig := new(action.Configuration)
    if err := actionConfig.Init(settings.RESTClientGetter(), helmConfig.Namespace, "secret", func(format string, v ...interface{}) {}); err != nil {
        return fmt.Errorf("failed to initialize helm action config: %w", err)
    }
    
    uninstall := action.NewUninstall(actionConfig)
    _, err := uninstall.Run(helmConfig.ReleaseName)
    if err != nil {
        return fmt.Errorf("failed to uninstall helm chart: %w", err)
    }
    
    return nil
}

func (i *CNIInstaller) downloadManifest(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}

func (i *CNIInstaller) getDefaultManifest(config *addonsv1beta1.CNIConfig) ([]byte, error) {
    version := config.Spec.Version
    
    var url string
    switch config.Spec.Type {
    case "calico":
        url = fmt.Sprintf("https://raw.githubusercontent.com/projectcalico/calico/v%s/manifests/calico.yaml", version)
    case "flannel":
        url = fmt.Sprintf("https://raw.githubusercontent.com/flannel-io/flannel/v%s/Documentation/kube-flannel.yml", version)
    case "cilium":
        url = fmt.Sprintf("https://raw.githubusercontent.com/cilium/cilium/v%s/install/kubernetes/quick-install.yaml", version)
    case "weave":
        url = fmt.Sprintf("https://github.com/weaveworks/weave/releases/download/v%s/weave-daemonset-k8s.yaml", version)
    default:
        return nil, fmt.Errorf("unsupported CNI type: %s", config.Spec.Type)
    }
    
    return i.downloadManifest(url)
}

func (i *CNIInstaller) customizeManifest(manifest []byte,
    config *addonsv1beta1.CNIConfig) ([]byte, error) {
    
    if config.Spec.Config == nil {
        return manifest, nil
    }
    
    tmpl, err := template.New("cni").Parse(string(manifest))
    if err != nil {
        return nil, err
    }
    
    var buf bytes.Buffer
    data := map[string]interface{}{
        "Config":        config.Spec.Config,
        "ClusterNetwork": config.Spec.ClusterNetwork,
    }
    
    if err := tmpl.Execute(&buf, data); err != nil {
        return nil, err
    }
    
    return buf.Bytes(), nil
}

func (i *CNIInstaller) applyManifest(ctx context.Context,
    cluster *clusterv1.Cluster, manifest []byte) error {
    
    decoder := yaml.NewDecodingSerializer(unstructured.UnstructuredJSONScheme)
    
    reader := bytes.NewReader(manifest)
    yamlReader := yaml.NewYAMLReader(reader)
    
    for {
        doc, err := yamlReader.Read()
        if err == io.EOF {
            break
        }
        if err != nil {
            return fmt.Errorf("failed to read yaml document: %w", err)
        }
        
        if len(bytes.TrimSpace(doc)) == 0 {
            continue
        }
        
        obj := &unstructured.Unstructured{}
        _, _, err = decoder.Decode(doc, nil, obj)
        if err != nil {
            return fmt.Errorf("failed to decode yaml: %w", err)
        }
        
        if err := i.Create(ctx, obj); err != nil {
            if !errors.IsAlreadyExists(err) {
                return fmt.Errorf("failed to create resource %s/%s: %w", obj.GetNamespace(), obj.GetName(), err)
            }
        }
    }
    
    return nil
}
```
### 3.4 依赖检查器
```go
package checker

import (
    "context"
    "fmt"
    "strings"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    addonsv1beta1 "sigs.k8s.io/cluster-api-addon-provider/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

type DependencyChecker struct {
    client.Client
}

func (d *DependencyChecker) CheckDependencies(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) bool {
    
    deps := config.Spec.Dependencies
    
    if deps.WaitForControlPlane && !cluster.Status.ControlPlaneReady {
        return false
    }
    
    if deps.WaitForNodes > 0 {
        if !d.checkNodesReady(ctx, cluster, deps.WaitForNodes) {
            return false
        }
    }
    
    for _, check := range deps.PreInstallChecks {
        if !d.executeCheck(ctx, cluster, check) {
            return false
        }
    }
    
    return true
}

func (d *DependencyChecker) CheckCSIDependencies(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CSIConfig) bool {
    
    deps := config.Spec.Dependencies
    
    if deps.WaitForCNI {
        if !d.checkCNIReady(ctx, cluster) {
            return false
        }
    }
    
    if deps.WaitForNodes > 0 {
        if !d.checkNodesReady(ctx, cluster, deps.WaitForNodes) {
            return false
        }
    }
    
    if deps.InfrastructureRef != nil {
        if !d.checkInfrastructureReady(ctx, deps.InfrastructureRef) {
            return false
        }
    }
    
    for _, check := range deps.PreInstallChecks {
        if !d.executeCSICheck(ctx, cluster, check) {
            return false
        }
    }
    
    return true
}

func (d *DependencyChecker) checkNodesReady(ctx context.Context,
    cluster *clusterv1.Cluster, minNodes int) bool {
    
    nodes := &corev1.NodeList{}
    if err := d.List(ctx, nodes); err != nil {
        return false
    }
    
    readyCount := 0
    for _, node := range nodes.Items {
        for _, condition := range node.Status.Conditions {
            if condition.Type == corev1.NodeReady && condition.Status == corev1.ConditionTrue {
                readyCount++
                break
            }
        }
    }
    
    return readyCount >= minNodes
}

func (d *DependencyChecker) checkCNIReady(ctx context.Context,
    cluster *clusterv1.Cluster) bool {
    
    pods := &corev1.PodList{}
    if err := d.List(ctx, pods, client.InNamespace("kube-system")); err != nil {
        return false
    }
    
    cniPodLabels := map[string]string{
        "calico": "k8s-app=calico-node",
        "flannel": "app=flannel",
        "cilium":   "k8s-app=cilium",
        "weave":    "name=weave-net",
    }
    
    for cniType, labelStr := range cniPodLabels {
        labelSelector, _ := labels.Parse(labelStr)
        matchingPods := 0
        readyPods := 0
        
        for _, pod := range pods.Items {
            if labelSelector.Matches(labels.Set(pod.Labels)) {
                matchingPods++
                if pod.Status.Phase == corev1.PodRunning {
                    ready := true
                    for _, condition := range pod.Status.Conditions {
                        if condition.Type == corev1.PodReady && condition.Status != corev1.ConditionTrue {
                            ready = false
                            break
                        }
                    }
                    if ready {
                        readyPods++
                    }
                }
            }
        }
        
        if matchingPods > 0 && readyPods > 0 {
            return true
        }
    }
    
    return false
}

func (d *DependencyChecker) checkInfrastructureReady(ctx context.Context,
    infraRef *corev1.ObjectReference) bool {
    
    key := types.NamespacedName{
        Namespace: infraRef.Namespace,
        Name:      infraRef.Name,
    }
    
    infraCluster := &unstructured.Unstructured{}
    infraCluster.SetAPIVersion(infraRef.APIVersion)
    infraCluster.SetKind(infraRef.Kind)
    
    if err := d.Get(ctx, key, infraCluster); err != nil {
        return false
    }
    
    ready, found, err := unstructured.NestedBool(infraCluster.Object, "status", "ready")
    if err != nil || !found {
        return false
    }
    
    return ready
}

func (d *DependencyChecker) executeCheck(ctx context.Context,
    cluster *clusterv1.Cluster, check addonsv1beta1.PreInstallCheck) bool {
    
    switch check.Type {
    case "podReady":
        return d.checkPodsReady(ctx, check.Namespace, check.LabelSelector, check.ExpectedCount)
    case "nodeReady":
        return d.checkNodesReady(ctx, cluster, check.ExpectedCount)
    case "apiServerReady":
        return d.checkAPIServerReady(ctx, cluster)
    case "custom":
        return d.executeCustomCheck(ctx, cluster, check.CustomCheck)
    default:
        return false
    }
}

func (d *DependencyChecker) checkPodsReady(ctx context.Context,
    namespace, labelSelector string, expectedCount int) bool {
    
    selector, err := labels.Parse(labelSelector)
    if err != nil {
        return false
    }
    
    pods := &corev1.PodList{}
    if err := d.List(ctx, pods, client.InNamespace(namespace), client.MatchingLabelsSelector{Selector: selector}); err != nil {
        return false
    }
    
    readyCount := 0
    for _, pod := range pods.Items {
        if pod.Status.Phase == corev1.PodRunning {
            for _, condition := range pod.Status.Conditions {
                if condition.Type == corev1.PodReady && condition.Status == corev1.ConditionTrue {
                    readyCount++
                    break
                }
            }
        }
    }
    
    if expectedCount > 0 {
        return readyCount >= expectedCount
    }
    return readyCount > 0
}

func (d *DependencyChecker) checkAPIServerReady(ctx context.Context,
    cluster *clusterv1.Cluster) bool {
    
    kubeconfig, err := d.getKubeconfig(ctx, cluster)
    if err != nil {
        return false
    }
    
    config, err := clientcmd.NewDefaultClientConfig(*kubeconfig, &clientcmd.ConfigOverrides{}).ClientConfig()
    if err != nil {
        return false
    }
    
    discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
    if err != nil {
        return false
    }
    
    _, err = discoveryClient.ServerVersion()
    return err == nil
}

func (d *DependencyChecker) executeCustomCheck(ctx context.Context,
    cluster *clusterv1.Cluster, customCheck *addonsv1beta1.CustomCheck) bool {
    
    if customCheck == nil {
        return false
    }
    
    kubeconfig, err := d.getKubeconfig(ctx, cluster)
    if err != nil {
        return false
    }
    
    cmd := exec.CommandContext(ctx, customCheck.Command[0], customCheck.Command[1:]...)
    
    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr
    
    if err := cmd.Run(); err != nil {
        return false
    }
    
    if customCheck.ExpectedOutput != "" {
        return strings.Contains(stdout.String(), customCheck.ExpectedOutput)
    }
    
    return true
}
```
### 3.5 CNI验证器
```go
package verifier

import (
    "context"
    "fmt"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/labels"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    addonsv1beta1 "sigs.k8s.io/cluster-api-addon-provider/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

type CNIVerifier struct {
    client.Client
}

func (v *CNIVerifier) Verify(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) bool {
    
    verification := config.Spec.Verification
    
    if verification.CheckPods {
        if !v.checkPodsReady(ctx, config) {
            return false
        }
    }
    
    if verification.CheckNetworkConnectivity {
        if !v.checkNetworkConnectivity(ctx, cluster, config) {
            return false
        }
    }
    
    return true
}

func (v *CNIVerifier) checkPodsReady(ctx context.Context,
    config *addonsv1beta1.CNIConfig) bool {
    
    verification := config.Spec.Verification
    
    namespace := verification.PodNamespace
    if namespace == "" {
        namespace = "kube-system"
    }
    
    pods := &corev1.PodList{}
    labelSelector := labels.Set(verification.PodLabels).AsSelector()
    
    if err := v.List(ctx, pods, client.InNamespace(namespace), client.MatchingLabelsSelector{Selector: labelSelector}); err != nil {
        return false
    }
    
    readyCount := 0
    for _, pod := range pods.Items {
        if pod.Status.Phase == corev1.PodRunning {
            for _, condition := range pod.Status.Conditions {
                if condition.Type == corev1.PodReady && condition.Status == corev1.ConditionTrue {
                    readyCount++
                    break
                }
            }
        }
    }
    
    if verification.MinReadyPods > 0 {
        return readyCount >= verification.MinReadyPods
    }
    return readyCount > 0
}

func (v *CNIVerifier) checkNetworkConnectivity(ctx context.Context,
    cluster *clusterv1.Cluster, config *addonsv1beta1.CNIConfig) bool {
    
    testPodSpec := config.Spec.Verification.TestPodSpec
    if testPodSpec == nil {
        testPodSpec = &corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:  "test",
                    Image: "busybox:1.36",
                    Command: []string{"sh", "-c", "echo 'Network test successful' && sleep 5"},
                },
            },
            RestartPolicy: corev1.RestartPolicyNever,
        }
    }
    
    testPod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            GenerateName: "cni-test-",
            Namespace:    "default",
            Labels: map[string]string{
                "addon.cluster.x-k8s.io/cni-test": "true",
            },
        },
        Spec: *testPodSpec,
    }
    
    if err := v.Create(ctx, testPod); err != nil {
        return false
    }
    
    defer func() {
        _ = v.Delete(ctx, testPod)
    }()
    
    timeout := time.After(2 * time.Minute)
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-timeout:
            return false
        case <-ticker.C:
            if err := v.Get(ctx, client.ObjectKey{Namespace: testPod.Namespace, Name: testPod.Name}, testPod); err != nil {
                return false
            }
            
            if testPod.Status.Phase == corev1.PodSucceeded {
                return true
            }
            
            if testPod.Status.Phase == corev1.PodFailed {
                return false
            }
        }
    }
}
```
## 四、完整使用示例
### 4.1 Calico CNI + AWS EBS CSI完整示例
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CNIConfig
metadata:
  name: calico-config
  namespace: default
spec:
  type: calico
  version: "3.26.0"
  clusterNetwork:
    podCIDR: "192.168.0.0/16"
    serviceCIDR: "10.128.0.0/12"
  config:
    typha:
      replicas: 2
    ipam:
      type: calico-ipam
    mtu: 1440
  installStrategy:
    method: manifest
    manifest:
      url: "https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml"
  dependencies:
    waitForControlPlane: true
    waitForNodes: 1
    preInstallChecks:
    - type: apiServerReady
  verification:
    checkPods: true
    podLabels:
      k8s-app: calico-node
    podNamespace: kube-system
    minReadyPods: 1
    checkNetworkConnectivity: true
---
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: CSIConfig
metadata:
  name: aws-ebs-csi-config
  namespace: default
spec:
  type: aws-ebs
  version: "1.20.0"
  config:
    controller:
      replicas: 2
    node:
      tolerateAllTaints: true
  installStrategy:
    method: helm
    helm:
      repoURL: "https://kubernetes-sigs.github.io/aws-ebs-csi-driver"
      chartName: "aws-ebs-csi-driver"
      releaseName: "aws-ebs-csi-driver"
      namespace: "kube-system"
      values:
        controller:
          replicaCount: 2
          serviceAccount:
            create: true
        node:
          serviceAccount:
            create: true
  storageClasses:
  - name: gp3
    isDefault: true
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
    parameters:
      type: gp3
      encrypted: "true"
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer
  - name: gp2
    isDefault: false
    parameters:
      type: gp2
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
  dependencies:
    waitForCNI: true
    waitForNodes: 1
  verification:
    checkPods: true
    podLabels:
      app.kubernetes.io/name: aws-ebs-csi-driver
    podNamespace: kube-system
    minReadyPods: 1
    checkStorageConnectivity: true
---
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterAddon
metadata:
  name: production-addons
  namespace: default
spec:
  clusterRef:
    apiVersion: cluster.x-k8s.io/v1beta1
    kind: Cluster
    name: production-cluster
    namespace: default
  cni:
    enabled: true
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: CNIConfig
      name: calico-config
    installTimeout: "10m"
    verifyTimeout: "5m"
  csi:
    enabled: true
    configRef:
      apiVersion: addons.cluster.x-k8s.io/v1beta1
      kind: CSIConfig
      name: aws-ebs-csi-config
    installTimeout: "10m"
    verifyTimeout: "5m"
    dependsOn:
    - cni
  installStrategy:
    parallel: false
    failurePolicy: Stop
    retryPolicy:
      maxRetries: 3
      backoff:
        initialDelay: "10s"
        maxDelay: "5m"
        multiplier: 2.0
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    serviceDomain: "cluster.local"
    services:
      cidrBlocks: ["10.128.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-cluster
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: production-cluster
  namespace: default
spec:
  region: us-west-2
  sshKeyName: my-key
  controlPlaneEndpoint:
    host: ""
    port: 6443
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: production-control-plane
  namespace: default
spec:
  replicas: 3
  version: "v1.28.0"
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachine
      name: control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
    joinConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-workers
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
  template:
    spec:
      clusterName: production-cluster
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: worker-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachine
        name: worker
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: worker-config
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          criSocket: unix:///var/run/containerd/containerd.sock
          kubeletExtraArgs:
            cgroup-driver: systemd
```
## 五、安装流程时序图
```plainText
时间轴  安装流程
  │
  ├─► 1. Infrastructure Provider创建节点
  │      • 创建VM/物理机
  │      • 安装CRI (containerd)
  │      • 配置网络基础设施
  │
  ├─► 2. Control Plane Provider初始化控制平面
  │      • kubeadm init
  │      • 启动API Server等核心组件
  │      • 等待控制平面就绪
  │      • 标记cluster.status.controlPlaneReady=true
  │
  ├─► 3. ClusterAddon控制器检测到控制平面就绪
  │      • 监听Cluster资源状态变化
  │      • 检查controlPlaneReady条件
  │
  ├─► 4. CNIConfig控制器开始安装CNI
  │      • 检查依赖：控制平面就绪 ✓
  │      • 检查依赖：节点数量满足 ✓
  │      • 下载/生成CNI manifest
  │      • 自定义配置
  │      • 应用manifest到目标集群
  │      • 等待CNI Pods就绪
  │      • 验证网络连通性
  │      • 标记cniConfig.status.ready=true
  │
  ├─► 5. ClusterAddon控制器检测到CNI就绪
  │      • 监听CNIConfig资源状态变化
  │      • 检查cniConfig.status.ready
  │      • 更新clusterAddon.status.cniStatus.ready=true
  │
  ├─► 6. CSIConfig控制器开始安装CSI
  │      • 检查依赖：CNI就绪 ✓
  │      • 检查依赖：节点数量满足 ✓
  │      • 检查依赖：基础设施存储就绪 ✓
  │      • 安装CSI Driver (Helm/Manifest)
  │      • 创建StorageClass
  │      • 等待CSI Pods就绪
  │      • 验证存储连通性
  │      • 标记csiConfig.status.ready=true
  │
  ├─► 7. ClusterAddon控制器检测到CSI就绪
  │      • 监听CSIConfig资源状态变化
  │      • 检查csiConfig.status.ready
  │      • 更新clusterAddon.status.csiStatus.ready=true
  │
  ├─► 8. ClusterAddon控制器标记整体就绪
  │      • 检查所有Addon状态
  │      • 标记clusterAddon.status.ready=true
  │      • 标记clusterAddon.status.phase="Installed"
  │
  ▼
     集群完全就绪（包含CNI/CSI）
```
## 六、关键设计要点总结
### 6.1 时序保证机制
- 依赖检查：每个Addon控制器检查前置依赖
- 状态轮询：定期检查依赖状态，满足条件后触发安装
- 条件标记：通过Conditions机制报告安装进度
### 6.2 错误处理机制
- 指数退避重试：失败后自动重试，延迟逐渐增加
- 状态持久化：记录安装状态和错误信息
- 回滚支持：支持回滚到之前的配置版本
### 6.3 可扩展性设计
- 多种安装方式：支持Manifest、Helm、Kustomize、自定义安装器
- 自定义验证：支持自定义验证逻辑
- 插件化架构：易于添加新的Addon类型

