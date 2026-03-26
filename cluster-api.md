# cluster-api
## OpenShift Installer 中实现的 Cluster API 相关设计
### 一、核心发现
根据 Red Hat 官方文档明确指出：
> **Machine API 将基于上游 Cluster API 项目的主要资源与自定义 OpenShift Container Platform 资源相结合。**

这表明 OpenShift 的 Machine API 是直接从 Cluster API 派生而来的。
### 二、OpenShift Machine API 架构
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    OpenShift Machine API (基于 Cluster API)                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    核心资源 (源自 Cluster API)                           │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  Machine                                                         │   │   │
│   │  │  - 描述节点主机的基本单元                                         │   │   │
│   │  │  - 具有 providerSpec 规格                                        │   │   │
│   │  │  - API: machine.openshift.io/v1beta1                            │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  MachineSet                                                      │   │   │
│   │  │  - 机器组，类似于 ReplicaSet                                      │   │   │
│   │  │  - 管理 Machine 的副本数                                         │   │   │
│   │  │  - API: machine.openshift.io/v1beta1                            │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  MachineDeployment (隐含支持)                                    │   │   │
│   │  │  - 通过 MachineSet 实现滚动更新                                   │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    OpenShift 扩展资源                                   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  MachineAutoscaler                                               │   │   │
│   │  │  - 自动扩展云中的机器                                             │   │   │
│   │  │  - 设置最小/最大扩展界限                                          │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  ClusterAutoscaler                                               │   │   │
│   │  │  - 基于上游集群自动扩展项目                                       │   │   │
│   │  │  - 与 Machine API 集成                                           │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │  MachineHealthCheck                                              │   │   │
│   │  │  - 检测机器不健康状态                                             │   │   │
│   │  │  - 删除并生成新机器                                               │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```
### 三、Cluster API 标签的继承证据
OpenShift Machine API 资源中保留了 Cluster API 的标签前缀，这是最直接的继承证据：
```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    # 注意：这些标签使用 cluster-api 前缀
    machine.openshift.io/cluster-api-cluster: <infrastructure_id>
    machine.openshift.io/cluster-api-machine-role: <role>
    machine.openshift.io/cluster-api-machine-type: <role>
    machine.openshift.io/cluster-api-machineset: <infrastructure_id>-<role>
spec:
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: <infrastructure_id>
      machine.openshift.io/cluster-api-machineset: <infrastructure_id>-<role>
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: <infrastructure_id>
        machine.openshift.io/cluster-api-machine-role: <role>
        machine.openshift.io/cluster-api-machine-type: <role>
        machine.openshift.io/cluster-api-machineset: <infrastructure_id>-<role>
```
### 四、OpenShift 与 Cluster API 的对比
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    OpenShift Machine API vs Cluster API                          │
├───────────────────────────┬─────────────────────────────────────────────────────┤
│       Cluster API         │              OpenShift Machine API                  │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  cluster.x-k8s.io/v1beta1 │  machine.openshift.io/v1beta1                       │
│  (标准 API Group)          │  (OpenShift 定制 API Group)                         │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  Cluster CR               │  无直接对应 (使用 Infrastructure CR)                 │
│  (集群定义)                │                                                     │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  Machine CR               │  Machine CR                                         │
│  (节点抽象)                │  (节点抽象，相同概念)                                │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  MachineSet CR            │  MachineSet CR                                      │
│  (副本集管理)              │  (副本集管理，相同概念)                              │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  MachineDeployment CR     │  通过 MachineSet 实现                                │
│  (滚动更新)                │  (隐含支持)                                         │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  MachineHealthCheck CR    │  MachineHealthCheck CR                              │
│  (健康检查)                │  (健康检查，相同概念)                                │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  Infrastructure Provider  │  Platform ProviderSpec                              │
│  (如 AWSMachine)          │  (如 AWSMachineProviderConfig)                      │
├───────────────────────────┼─────────────────────────────────────────────────────┤
│  Bootstrap Provider       │  userDataSecret                                     │
│  (如 KubeadmConfig)       │  (Ignition 配置)                                    │
└───────────────────────────┴─────────────────────────────────────────────────────┘
```
### 五、OpenShift 特有的扩展
#### 5.1 平台特定的 ProviderSpec
OpenShift 为每个平台定义了特定的 ProviderSpec：

| 平台 | ProviderSpec Kind | API Group |
|------|------------------|-----------|
| AWS | AWSMachineProviderConfig | awsproviderconfig.openshift.io/v1beta1 |
| Azure | AzureMachineProviderSpec | azureproviderconfig.openshift.io/v1beta1 |
| GCP | GCPMachineProviderSpec | gcpproviderconfig.openshift.io/v1beta1 |
| vSphere | VSphereMachineProviderSpec | vsphereprovider.openshift.io/v1beta1 |
| Bare Metal | BareMetalMachineProviderSpec | metal3.io/v1alpha1 |
| OpenStack | OpenStackMachineProviderSpec | openstackproviderconfig.openshift.io/v1beta1 |
#### 5.2 Machine API Operator 架构
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Machine API Operator 架构                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    openshift-machine-api 命名空间                        │   │
│   │                                                                         │   │
│   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │   │
│   │  │ machine-api-    │  │ cluster-        │  │ machine-api-    │        │   │
│   │  │ operator        │  │ autoscaler-     │  │ controllers     │        │   │
│   │  │                 │  │ operator        │  │                 │        │   │
│   │  │ 部署和管理      │  │ 部署和管理      │  │ 核心控制器:     │        │   │
│   │  │ machine-api     │  │ 集群自动扩展器  │  │ - controller-   │        │   │
│   │  │ controllers     │  │                 │  │   manager       │        │   │
│   │  │                 │  │                 │  │ - machine-      │        │   │
│   │  │                 │  │                 │  │   controller    │        │   │
│   │  │                 │  │                 │  │ - nodelink-     │        │   │
│   │  │                 │  │                 │  │   controller    │        │   │
│   │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │   │
│   │                                                                         │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │  │                    CRD 资源                                      │   │   │
│   │  │  Machine │ MachineSet │ MachineAutoscaler │ ClusterAutoscaler  │   │   │
│   │  │  MachineHealthCheck │ BareMetalHost (裸机)                       │   │   │
│   │  └─────────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```
### 六、OpenShift Installer 与 Machine API 的关系
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    OpenShift Installer 与 Machine API 的关系                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    安装阶段                                              │   │
│   │                                                                         │   │
│   │  install-config.yaml                                                    │   │
│   │       │                                                                 │   │
│   │       ▼                                                                 │   │
│   │  OpenShift Installer                                                    │   │
│   │       │                                                                 │   │
│   │       ├──► 生成 Terraform 配置 (云平台)                                 │   │
│   │       │    或 Bare Metal IPI 配置                                       │   │
│   │       │                                                                 │   │
│   │       ├──► 创建初始 MachineSet (Worker 节点)                            │   │
│   │       │    - 使用 Cluster API 兼容的标签                                │   │
│   │       │    - 使用平台特定的 ProviderSpec                                │   │
│   │       │                                                                 │   │
│   │       └──► 部署 Machine API Operator                                    │   │
│   │            - 安装后管理所有节点置备                                      │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    安装后阶段                                            │   │
│   │                                                                         │   │
│   │  Machine API Operator                                                   │   │
│   │       │                                                                 │   │
│   │       ├──► 管理 Machine 生命周期                                        │   │
│   │       │    - 创建/删除/更新节点                                         │   │
│   │       │                                                                 │   │
│   │       ├──► 与 Cluster Autoscaler 集成                                   │   │
│   │       │    - 自动扩缩容                                                 │   │
│   │       │                                                                 │   │
│   │       └──► MachineHealthCheck                                           │   │
│   │            - 节点故障自动修复                                            │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```
### 七、关键差异点
| 特性 | Cluster API | OpenShift Machine API |
|------|-------------|----------------------|
| **API Group** | `cluster.x-k8s.io` | `machine.openshift.io` |
| **设计理念** | 模块化、可扩展 | 一体化、企业级 |
| **Bootstrap** | KubeadmConfig 等 Provider | Ignition + userDataSecret |
| **基础设施抽象** | Infrastructure Provider 接口 | ProviderSpec 内嵌 |
| **控制平面管理** | KubeadmControlPlane | 由 Installer 直接管理 |
| **版本管理** | 无内置 | CVO 统一管理 |
### 八、总结
OpenShift Installer 中实现的 Cluster API 相关设计主要体现在：
1. **Machine API 是 Cluster API 的派生实现**
   - OpenShift 官方文档明确说明 Machine API 基于上游 Cluster API 项目
   - 保留了 `cluster-api` 标签前缀作为继承证据
2. **核心资源完全对应**
   - `Machine` - 节点抽象
   - `MachineSet` - 副本集管理
   - `MachineHealthCheck` - 健康检查
3. **OpenShift 特有扩展**
   - 定制 API Group (`machine.openshift.io`)
   - 平台特定的 ProviderSpec
   - 与 Cluster Autoscaler 深度集成
   - 与 CVO 版本管理集成
4. **设计差异**
   - OpenShift 采用一体化设计，将 Bootstrap 和 Infrastructure 集成到 ProviderSpec 中
   - Cluster API 采用模块化设计，通过 Provider 接口解耦
        
