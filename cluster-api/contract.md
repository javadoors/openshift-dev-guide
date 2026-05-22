# CAPI Provider 完整契约规范
> 基于 `sigs.k8s.io/cluster-api` v1.9.x (API v1beta1)

## 一、Provider 分类体系
CAPI 定义了 **三类 Provider**，每类必须独立实现并注册：

| Provider 类型 | 职责 | API Group 前缀 |
|---|---|---|
| **Infrastructure Provider** | 管理基础设施资源（机器、网络、负载均衡等） | `infrastructure.cluster.x-k8s.io` |
| **Bootstrap Provider** | 生成节点引导配置（证书、kubeadm config 等） | `bootstrap.cluster.x-k8s.io` |
| **Control Plane Provider** | 管理控制平面生命周期 | `controlplane.cluster.x-k8s.io` |

## 二、Infrastructure Provider 契约

### 2.1 必须定义的 CRD

#### 2.1.1 InfraCluster（如 `BKECluster`）
**Spec 契约字段：**
```go
type InfraClusterSpec struct {
    // ClusterRef 引用 CAPI 核心 Cluster 对象（可选，通过 label 约定代替）
    
    // ControlPlaneEndpoint 表示 API Server 访问端点
    // 当基础设施就绪后必须设置此字段
    // +optional
    ControlPlaneEndpoint APIEndpoint `json:"controlPlaneEndpoint,omitempty"`
}
```

**Status 契约字段（必须严格遵从）：**
```go
type InfraClusterStatus struct {
    // Ready 表示基础设施集群是否就绪
    // CAPI 核心控制器通过此字段判断 Cluster.spec.infrastructureReady
    // +required
    Ready bool `json:"ready"`

    // FailureDomains 报告可用的故障域
    // CAPI 核心控制器使用此信息进行 Machine 调度决策
    // +optional
    FailureDomains clusterv1.FailureDomains `json:"failureDomains,omitempty"`

    // FailureMessage 表示集群级别的失败原因（人类可读）
    // 设置后 CAPI 不会持续重试
    // +optional
    FailureMessage *string `json:"failureMessage,omitempty"`

    // FailureReason 表示集群级别的失败原因（机器可读）
    // 使用 ClusterStatusError 类型
    // +optional
    FailureReason *clusterv1.ClusterStatusError `json:"failureReason,omitempty"`

    // Conditions 遵从 CAPI 条件体系
    // +optional
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
```

**必须实现的接口方法：**
```go
// InfraCluster 必须实现以下接口
type InfraCluster interface {
    // GetConditions 返回条件列表
    GetConditions() clusterv1.Conditions
    // SetConditions 设置条件列表
    SetConditions(conditions clusterv1.Conditions)
}
```

**必须设置的标签：**
```yaml
metadata:
  labels:
    cluster.x-k8s.io/provider: "infrastructure-<provider-name>"
    cluster.x-k8s.io/v1beta1: "v1beta1"
```
**OwnerReference 约束：**
- InfraCluster 必须设置 `metadata.ownerReferences` 指向对应的 CAPI `Cluster` 对象
- Controller 字段必须设为 `true`

#### 2.1.2 InfraMachine（如 `BKEMachine`）
**Spec 契约字段：**
```go
type InfraMachineSpec struct {
    // ProviderID 是基础设施中机器的唯一标识符
    // 格式: <cloud-provider>://<provider-id>
    // Machine 控制器使用此字段关联 Node 和 Machine
    // +optional
    ProviderID *string `json:"providerID,omitempty"`
}
```

**Status 契约字段（必须严格遵从）：**
```go
type InfraMachineStatus struct {
    // Ready 表示基础设施机器是否就绪
    // CAPI 核心控制器通过此字段判断 Machine 是否可引导
    // +required
    Ready bool `json:"ready"`

    // FailureMessage 表示机器级别的失败原因（人类可读）
    // +optional
    FailureMessage *string `json:"failureMessage,omitempty"`

    // FailureReason 表示机器级别的失败原因（机器可读）
    // 使用 MachineStatusError 类型
    // +optional
    FailureReason *clusterv1.MachineStatusError `json:"failureReason,omitempty"`

    // Addresses 报告机器的地址列表
    // CAPI 使用此信息填充 Machine.Status.Addresses
    // +optional
    Addresses []clusterv1.MachineAddress `json:"addresses,omitempty"`

    // Conditions 遵从 CAPI 条件体系
    // +optional
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
```

**必须实现的接口方法：**
```go
type InfraMachine interface {
    GetConditions() clusterv1.Conditions
    SetConditions(conditions clusterv1.Conditions)
}
```

#### 2.1.3 InfraMachineTemplate（如 `BKEMachineTemplate`）
```go
type InfraMachineTemplate struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec InfraMachineTemplateSpec `json:"spec,omitempty"`
}

type InfraMachineTemplateSpec struct {
    Template InfraMachineTemplateResource `json:"template"`
}

type InfraMachineTemplateResource struct {
    // Spec 是 InfraMachine 的 Spec
    Spec InfraMachineSpec `json:"spec"`
}
```

#### 2.1.4 InfraClusterTemplate（ClusterClass 支持）

若需支持 ClusterClass，还必须定义：
```go
type InfraClusterTemplate struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec InfraClusterTemplateSpec `json:"spec,omitempty"`
}

type InfraClusterTemplateSpec struct {
    Template InfraClusterTemplateResource `json:"template"`
}

type InfraClusterTemplateResource struct {
    Spec InfraClusterSpec `json:"spec"`
}
```

### 2.2 Infrastructure Provider 控制器行为契约

#### 2.2.1 InfraCluster 控制器

| 事件 | 行为契约 |
|---|---|
| **创建 (Create)** | 1. 为 Cluster 创建基础设施资源<br>2. 设置 `status.ready = true` 表示就绪<br>3. 设置 `status.controlPlaneEndpoint` 使 CAPI 核心可传播到 Cluster<br>4. 设置 `status.failureDomains` 供 Machine 调度<br>5. 设置 OwnerReference 指向 Cluster |
| **更新 (Update)** | 响应 spec 变更，更新基础设施资源 |
| **删除 (Delete)** | 1. 清理所有基础设施资源<br>2. 使用 Finalizer 防止在清理完成前被删除<br>3. 清理完成后移除 Finalizer |
| **失败** | 设置 `status.failureReason` + `status.failureMessage`，CAPI 核心将停止重试 |

#### 2.2.2 InfraMachine 控制器

| 事件 | 行为契约 |
|---|---|
| **创建 (Create)** | 1. 创建基础设施机器<br>2. 设置 `spec.providerID` 标识机器<br>3. 设置 `status.ready = true`<br>4. 设置 `status.addresses`<br>5. 设置 OwnerReference 指向 Machine |
| **更新 (Update)** | 响应 spec 变更（如 ProviderID 更新） |
| **删除 (Delete)** | 1. 销毁基础设施机器<br>2. 使用 Finalizer 防止提前删除<br>3. 清理完成后移除 Finalizer |
| **失败** | 设置 `status.failureReason` + `status.failureMessage` |

### 2.3 Finalizer 约定

```go
const InfraClusterFinalizer = "<infracluster>.infrastructure.cluster.x-k8s.io"
const InfraMachineFinalizer = "<inframachine>.infrastructure.cluster.x-k8s.io"
```

### 2.4 标签与注解约定

**InfraCluster 必须携带的标签：**
```yaml
labels:
  cluster.x-k8s.io/cluster-name: "<cluster-name>"
```

**InfraMachine 必须携带的标签：**
```yaml
labels:
  cluster.x-k8s.io/cluster-name: "<cluster-name>"
  cluster.x-k8s.io/machine-name: "<machine-name>"
```

## 三、Bootstrap Provider 契约

### 3.1 必须定义的 CRD

#### 3.1.1 BootstrapConfig（如 `KubeadmConfig`）

**Spec 契约字段：**

```go
type BootstrapConfigSpec struct {
    // ClusterConfiguration 传递给 kubeadm init 的配置（仅控制平面首节点使用）
    // +optional
    ClusterConfiguration *kubeadm.ClusterConfiguration `json:"clusterConfiguration,omitempty"`

    // InitConfiguration 传递给 kubeadm init 的 InitConfiguration
    // +optional
    InitConfiguration *kubeadm.InitConfiguration `json:"initConfiguration,omitempty"`

    // JoinConfiguration 传递给 kubeadm join 的配置
    // +optional
    JoinConfiguration *kubeadm.JoinConfiguration `json:"joinConfiguration,omitempty"`

    // Files 要写入节点的额外文件
    // +optional
    Files []File `json:"files,omitempty"`

    // PreKubeadmCommands 在 kubeadm 执行前运行的命令
    // +optional
    PreKubeadmCommands []string `json:"preKubeadmCommands,omitempty"`

    // PostKubeadmCommands 在 kubeadm 执行后运行的命令
    // +optional
    PostKubeadmCommands []string `json:"postKubeadmCommands,omitempty"`

    // Users 要创建的用户
    // +optional
    Users []User `json:"users,omitempty"`

    // NTP 配置
    // +optional
    NTP *NTP `json:"ntp,omitempty"`

    // Format 指定引导数据输出格式 (cloud-init / ignition)
    // +optional
    Format BootstrapFormat `json:"format,omitempty"`

    // Verbosity kubeadm 日志级别
    // +optional
    Verbosity *int32 `json:"verbosity,omitempty"`
}
```

**Status 契约字段：**

```go
type BootstrapConfigStatus struct {
    // Ready 表示引导数据是否已生成
    // +required
    Ready bool `json:"ready"`

    // DataSecretName 引用包含引导数据的 Secret 名称
    // CAPI 核心控制器将此值传播到 Machine.Spec.Bootstrap.DataSecretName
    // Secret 必须在与 Machine 相同的命名空间中
    // +optional
    DataSecretName *string `json:"dataSecretName,omitempty"`

    // FailureMessage 人类可读的失败原因
    // +optional
    FailureMessage *string `json:"failureMessage,omitempty"`

    // FailureReason 机器可读的失败原因
    // +optional
    FailureReason *string `json:"failureReason,omitempty"`

    // Conditions
    // +optional
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
```

#### 3.1.2 BootstrapConfigTemplate（ClusterClass 支持）

```go
type BootstrapConfigTemplate struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec BootstrapConfigTemplateSpec `json:"spec,omitempty"`
}

type BootstrapConfigTemplateSpec struct {
    Template BootstrapConfigTemplateResource `json:"template"`
}

type BootstrapConfigTemplateResource struct {
    Spec BootstrapConfigSpec `json:"spec"`
}
```

### 3.2 Bootstrap Provider 控制器行为契约

| 事件 | 行为契约 |
|---|---|
| **创建 (Create)** | 1. 生成引导数据（如 cloud-init）<br>2. 创建 Secret 存储引导数据<br>3. 设置 `status.dataSecretName` 指向 Secret<br>4. 设置 `status.ready = true`<br>5. 设置 OwnerReference 指向 Machine |
| **更新 (Update)** | BootstrapConfig 通常不可变，更新时重建 |
| **删除 (Delete)** | 清理关联的 Secret 资源 |

**引导数据 Secret 格式：**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <dataSecretName>
  namespace: <machine-namespace>
type: Opaque
data:
  value: <base64-encoded-bootstrap-data>
```

## 四、Control Plane Provider 契约

### 4.1 必须定义的 CRD

#### 4.1.1 ControlPlane（如 `KubeadmControlPlane`）

**Spec 契约字段：**
```go
type ControlPlaneSpec struct {
    // MachineTemplate 引用 InfraMachineTemplate
    // +required
    MachineTemplate ControlPlaneMachineTemplate `json:"machineTemplate"`

    // Replicas 控制平面副本数
    // +optional
    Replicas *int32 `json:"replicas,omitempty"`

    // Version Kubernetes 版本
    // +required
    Version string `json:"version"`

    // RolloutStrategy 滚动更新策略
    // +optional
    RolloutStrategy *RolloutStrategy `json:"rolloutStrategy,omitempty"`
}
```

**Status 契约字段：**
```go
type ControlPlaneStatus struct {
    // Ready 表示控制平面是否就绪
    // +required
    Ready bool `json:"ready"`

    // Replicas 当前副本数
    // +optional
    Replicas int32 `json:"replicas,omitempty"`

    // Version 当前运行的 Kubernetes 版本
    // +optional
    Version *string `json:"version,omitempty"`

    // Selector 用于关联 Machine 的标签选择器
    // +optional
    Selector string `json:"selector,omitempty"`

    // FailureMessage 人类可读的失败原因
    // +optional
    FailureMessage *string `json:"failureMessage,omitempty"`

    // FailureReason 机器可读的失败原因
    // +optional
    FailureReason *string `json:"failureReason,omitempty"`

    // Initialized 表示控制平面是否已初始化
    // +optional
    Initialized bool `json:"initialized,omitempty"`

    // ExternalManagedControlPlane 是否为外部托管控制平面
    // +optional
    ExternalManagedControlPlane bool `json:"externalManagedControlPlane,omitempty"`

    // Conditions
    // +optional
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
```

#### 4.1.2 ControlPlaneTemplate（ClusterClass 支持）

```go
type ControlPlaneTemplate struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec ControlPlaneTemplateSpec `json:"spec,omitempty"`
}

type ControlPlaneTemplateSpec struct {
    Template ControlPlaneTemplateResource `json:"template"`
}

type ControlPlaneTemplateResource struct {
    Spec ControlPlaneSpec `json:"spec"`
}
```

### 4.2 Control Plane Provider 控制器行为契约

| 事件 | 行为契约 |
|---|---|
| **初始化** | 1. 为第一个控制平面节点生成证书和配置<br>2. 创建 Secret 存放 kubeconfig（`<cluster-name>-kubeconfig`）<br>3. 创建控制平面 Machine<br>4. 设置 `status.ready = true` |
| **扩容** | 创建额外的控制平面 Machine 并加入集群 |
| **缩容** | 按安全顺序删除控制平面 Machine |
| **升级** | 1. 按序滚动更新控制平面 Machine<br>2. 先升级第一个控制平面节点<br>3. 等待节点 Ready 后再升级下一个 |
| **删除** | 清理所有控制平面相关资源 |

**Kubeconfig Secret 格式：**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <cluster-name>-kubeconfig
  namespace: <cluster-namespace>
  ownerReferences:
  - apiVersion: cluster.x-k8s.io/v1beta1
    kind: Cluster
    name: <cluster-name>
    controller: true
type: Opaque
data:
  value: <base64-encoded-kubeconfig>
```
## 五、CAPI 核心资源引用契约

### 5.1 Cluster 对 Provider 的引用方式
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  # 引用 Infrastructure Provider
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: BKECluster
    name: my-bke-cluster
    namespace: default

  # 引用 Control Plane Provider
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-control-plane
    namespace: default
```

### 5.2 Machine 对 Provider 的引用方式
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: my-machine
spec:
  clusterName: my-cluster

  # 引用 Infrastructure Provider
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: BKEMachine
    name: my-bke-machine
    namespace: default

  # 引用 Bootstrap Provider
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: my-bootstrap-config
      namespace: default
```
## 六、ClusterClass 契约（可选但推荐）

### 6.1 ClusterClass 定义
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: <class-name>
spec:
  # 引用 InfraClusterTemplate
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: BKEClusterTemplate
      name: <template-name>

  # 引用 ControlPlaneTemplate
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: <template-name>
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: BKEMachineTemplate
        name: <template-name>

  # Workers 定义
  workers:
    machineDeployments:
    - class: <worker-class-name>
      template:
        bootstrap:
          ref:
            apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
            kind: KubeadmConfigTemplate
            name: <template-name>
        infrastructure:
          ref:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: BKEMachineTemplate
            name: <template-name>

  # Variables 定义（自 CAPI v1.4+）
  variables:
  - name: <variable-name>
    required: true
    schema:
      openAPIV3Schema:
        type: string

  # Patches 定义（自 CAPI v1.4+）
  patches:
  - name: <patch-name>
    definitions:
    - selector:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: BKEClusterTemplate
        matchResources:
          infrastructureCluster: true
      jsonPatches:
      - op: add
        path: /spec/template/spec/<field>
        valueFrom:
          variable: <variable-name>
```
### 6.2 ClusterClass 下创建 Cluster
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  topology:
    class: <class-name>
    version: v1.28.0
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
      - class: <worker-class-name>
        name: md-0
        replicas: 3
    variables:
    - name: <variable-name>
      value: <variable-value>
```
## 七、条件（Conditions）契约

### 7.1 必须支持的条件类型

**InfraCluster 条件：**

| Condition Type | 描述 |
|---|---|
| `InfrastructureReady` | 基础设施集群是否就绪 |
| `ControlPlaneAvailable` | 控制平面是否可用（由 CAPI 核心设置） |

**InfraMachine 条件：**

| Condition Type | 描述 |
|---|---|
| `InfrastructureReady` | 基础设施机器是否就绪 |

### 7.2 条件设置规则

```go
// 使用 CAPI 条件工具库
import "sigs.k8s.io/cluster-api/util/conditions"

// 设置条件
conditions.MarkTrue(infraCluster, clusterv1.InfrastructureReadyCondition)
conditions.MarkFalse(infraCluster, clusterv1.InfrastructureReadyCondition,
    clusterv1.InfrastructureReadyReason, clusterv1.ConditionSeverityWarning,
    "infrastructure is not ready")

// 设置摘要条件
conditions.SetSummary(infraMachine,
    conditions.WithConditions(
        clusterv1.InfrastructureReadyCondition,
    ),
)
```
## 八、Reconcile 行为契约

### 8.1 Phase 驱动的状态机
Provider 控制器的 Reconcile 必须遵循 CAPI 的 Phase 驱动模型：
```
Cluster Phase 流转:
  Pending → Provisioning → Provisioned → Deleting

Machine Phase 流转:
  Pending → Provisioning → Provisioned → Running → Deleting → Deleted → Failed
```

### 8.2 Reconcile 关键检查顺序
**InfraCluster Reconcile：**
```
1. 获取 CAPI Cluster 对象 (通过 OwnerReference)
2. 若 Cluster 不存在 → 返回
3. 若 Cluster.DeletionTimestamp != nil → 执行清理
4. 若 DeletionTimestamp 为 nil 且无 Finalizer → 添加 Finalizer
5. 调配基础设施资源
6. 设置 status.ready / status.failureReason / status.failureMessage
7. 设置 status.controlPlaneEndpoint (若就绪)
8. 设置 status.failureDomains (若就绪)
9. 更新 Conditions
10. Patch Status
```

**InfraMachine Reconcile：**
```
1. 获取 CAPI Machine 对象 (通过 OwnerReference)
2. 若 Machine 不存在 → 返回
3. 若 Machine.DeletionTimestamp != nil → 执行清理
4. 若 DeletionTimestamp 为 nil 且无 Finalizer → 添加 Finalizer
5. 调配基础设施机器
6. 设置 spec.providerID
7. 设置 status.ready / status.addresses
8. 设置 status.failureReason / status.failureMessage (若失败)
9. 更新 Conditions
10. Patch Status
```
## 九、clusterctl 集成契约

### 9.1 metadata.yaml
Provider 必须提供 `metadata.yaml` 文件：
```yaml
apiVersion: clusterctl.cluster.x-k8s.io/v1alpha3
kind: Metadata
releaseSeries:
- major: 1
  minor: 9
  contract: v1beta1
```

### 1.9.2 clusterctl-settings.json
```json
{
  "name": "cluster-api-provider-bke",
  "config": {
    "components": ["./config/default"],
    "nextVersion": "v0.1.0"
  }
}
```

### 9.3 集群模板 (cluster-template.yaml)
Provider 必须提供集群模板供 `clusterctl generate cluster` 使用：
```yaml
# cluster-template.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: ${CLUSTER_NAME}
spec:
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: BKECluster
    name: ${CLUSTER_NAME}
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: ${CLUSTER_NAME}
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: BKECluster
metadata:
  name: ${CLUSTER_NAME}
spec:
  controlPlaneEndpoint:
    host: ${CONTROL_PLANE_ENDPOINT_IP}
    port: 6443
---
# ... 更多资源
```
## 十、Provider 注册与 RBAC 契约

### 10.1 CRD 注册
Provider 必须注册 CRD 并添加 CAPI 标签：
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: bkeclusters.infrastructure.cluster.x-k8s.io
  labels:
    cluster.x-k8s.io/provider: "infrastructure-bke"
    cluster.x-k8s.io/v1beta1: "v1beta1"
    cluster.x-k8s.io/v1alpha4: "v1beta1"
spec:
  group: infrastructure.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        # ...
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Ready
      type: string
      jsonPath: .status.ready
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    kind: BKECluster
    listKind: BKEClusterList
    singular: bkecluster
    plural: bkeclusters
    shortNames:
    - bc
```

### 10.2 RBAC 权限
Provider 控制器需要以下最小权限：
```go
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=clusters,verbs=get;list;watch
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=machines,verbs=get;list;watch
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkeclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkeclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkeclusters/finalizers,verbs=update
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=events,verbs=get;list;watch;create;update;patch
```
## 十一、关键 API 类型引用
Provider 必须依赖以下 CAPI 核心类型：
```go
import (
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

// 核心引用类型
clusterv1.Cluster                    // CAPI 核心集群对象
clusterv1.Machine                    // CAPI 核心机器对象
clusterv1.MachineDeployment          // 机器部署
clusterv1.MachineSet                 // 机器集
clusterv1.MachineHealthCheck         // 机器健康检查
clusterv1.ClusterClass               // 集群类
clusterv1.FailureDomains             // 故障域 map[string]FailureDomainSpec
clusterv1.MachineAddress             // 机器地址
clusterv1.Conditions                 // 条件列表
clusterv1.ClusterStatusError         // 集群错误类型
clusterv1.MachineStatusError         // 机器错误类型
clusterv1.APIEndpoint                // API 端点 {Host, Port}
```
## 十二、版本兼容性矩阵

| CAPI 版本 | API 版本 | Go 版本 | controller-runtime |
|---|---|---|---|
| v1.9.x | v1beta1 (stable), v1beta2 (alpha) | 1.22+ | v0.19.x |
| v1.8.x | v1beta1 | 1.21+ | v0.17.x |
| v1.7.x | v1beta1 | 1.20+ | v0.15.x |

**契约版本标签约定：**
```yaml
labels:
  cluster.x-k8s.io/v1beta1: "v1beta1"   # 必须设置
```
## 总结
开发 CAPI Provider 的核心契约要点：
1. **CRD 契约**：InfraCluster/InfraMachine/BootstrapConfig/ControlPlane 的 Spec 和 Status 必须包含 CAPI 规定的关键字段（`ready`、`providerID`、`failureReason`、`failureMessage`、`addresses`、`failureDomains`、`controlPlaneEndpoint`）
2. **接口契约**：所有 Provider CRD 必须实现 `GetConditions()` / `SetConditions()` 方法
3. **控制器行为契约**：Reconcile 必须遵循 Finalizer → 调配 → 状态更新 → Patch 的顺序
4. **引用契约**：OwnerReference 必须正确设置，标签必须按约定添加
5. **ClusterClass 契约**：若支持 ClusterClass，必须提供 Template CRD 和 Variables/Patches 定义
6. **clusterctl 契约**：必须提供 metadata.yaml、clusterctl-settings.json 和集群模板
        
