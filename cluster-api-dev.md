# 📌 为什么没有 Worker Provider
- **工作节点的生命周期管理**  
  在 Cluster API 中，工作节点（Worker Nodes）是通过 **MachineDeployment → MachineSet → Machine** 来管理的。  
- **Provider 的职责划分**  
  - **Infrastructure Provider**：负责底层资源（VM、网络、存储），既用于控制平面节点，也用于工作节点。  
  - **Bootstrap Provider**：负责节点启动配置（cloud-init、Ignition、kubeadm），同样适用于所有节点。  
  - **Control Plane Provider**：专门管理控制平面节点的升级和替换。  

👉 因此，工作节点并不需要一个单独的 Provider 类型，它们直接复用 **Infrastructure Provider + Bootstrap Provider** 的组合来完成创建和引导。  
## 🎯 总结
- Cluster API 的 Provider 类型是 **按功能划分**，而不是按节点角色划分。  
- 控制平面节点需要额外的管理逻辑，所以有 **Control Plane Provider**。  
- 工作节点只需要基础设施和引导配置，因此直接由 **Infrastructure Provider + Bootstrap Provider** 支撑，不需要单独的 Worker Provider。  
## Cluster API Provider 职责速查表
把三类 Provider 的作用、适用对象（控制平面/工作节点）、典型实现列出来，方便快速对照：  
### 📊 Cluster API Provider 职责速查表
| Provider 类型 | 主要作用 | 适用对象 | 典型实现 |
|---------------|----------|----------|----------|
| **Infrastructure Provider** | 提供底层基础设施资源（VM、网络、存储），定义节点的宿主环境 | 控制平面节点 + 工作节点 | AWSCluster / AWSMachine, AzureCluster / AzureMachine, GCPCluster / GCPMachine, vSphereCluster / VSphereMachine, DockerCluster / DockerMachine |
| **Bootstrap Provider** | 提供节点引导配置（安装 kubelet、加入集群、初始化控制平面），通常生成 cloud-init 或 Ignition 脚本 | 控制平面节点 + 工作节点 | KubeadmConfig / KubeadmConfigTemplate, TalosConfig, EKS Bootstrap Provider |
| **Control Plane Provider** | 管理控制平面节点的生命周期（扩缩容、滚动升级、etcd/核心组件管理） | 仅控制平面节点 | KubeadmControlPlane, TalosControlPlane, ExternalControlPlane Provider |
### 📌 关键点说明
- **没有 Worker Provider**：工作节点不需要单独的 Provider 类型，它们直接复用 **Infrastructure Provider + Bootstrap Provider** 的组合来完成创建和引导。  
- **控制平面特殊性**：因为控制平面涉及 etcd、CoreDNS、API Server 等关键组件，需要额外的 Provider（Control Plane Provider）来保证一致性和安全升级。  
- **多 Provider 协作**：一个完整的集群通常至少需要一个 Infrastructure Provider + 一个 Bootstrap Provider；如果是自管控平面，还需要一个 Control Plane Provider。  
### 🎯 总结
- **Infrastructure Provider**：负责“机器和环境”。  
- **Bootstrap Provider**：负责“节点启动配置”。  
- **Control Plane Provider**：负责“控制平面管理”。  
- **Worker 节点**：由 MachineDeployment 驱动，依赖 Infra + Bootstrap，不需要单独 Provider。  

# Cluster API开发指南:UPI模式openFuyao集群安装升级功能
基于Cluster API的架构和开发模式,我为您创建了一份详细的开发指南:
## 一、Cluster API架构概述
### 1.1 核心概念
Cluster API采用声明式API管理Kubernetes集群生命周期,核心资源包括:
```go
// Cluster - 集群顶层资源
type Cluster struct {
    Spec ClusterSpec
    Status ClusterStatus
}

// Machine - 单个节点抽象
type Machine struct {
    Spec MachineSpec
    Status MachineStatus
}

// MachineSet - Machine副本集
type MachineSet struct {
    Spec MachineSetSpec
    Status MachineSetStatus
}

// MachineDeployment - Machine部署控制器
type MachineDeployment struct {
    Spec MachineDeploymentSpec
    Status MachineDeploymentStatus
}
```
### 1.2 Provider类型
Cluster API定义了三种Provider类型:
1. **Infrastructure Provider**: 提供基础设施资源(VM、网络、存储等)
2. **Bootstrap Provider**: 提供节点引导配置(kubeadm、cloud-init等)
3. **Control Plane Provider**: 提供控制面管理(KubeadmControlPlane等)
### 1.3 UPI vs IPI模式
- **IPI (Installer-Provisioned Infrastructure)**: Provider自动创建和管理基础设施
- **UPI (User-Provisioned Infrastructure)**: 用户提供基础设施,Provider仅管理集群配置
## 二、Provider开发基础
### 2.1 项目结构
```
cluster-api-provider-openfuyao/
├── api/
│   └── v1beta1/
│       ├── openfuyaocluster_types.go      # 集群CRD
│       ├── openfuyaomachine_types.go      # 机器CRD
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── controllers/
│   ├── openfuyaocluster_controller.go     # 集群控制器
│   └── openfuyaomachine_controller.go     # 机器控制器
├── config/
│   ├── crd/
│   │   └── bases/
│   ├── rbac/
│   └── manager/
├── webhooks/
│   └── v1beta1/
│       ├── openfuyaocluster_webhook.go
│       └── openfuyaomachine_webhook.go
├── cmd/
│   └── main.go
├── Dockerfile
└── go.mod
```
### 2.2 CRD定义
#### OpenFuyaoCluster CRD
```go
// d:\code\github\cluster-api-provider-openfuyao\api\v1beta1\openfuyaocluster_types.go

package v1beta1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    clusterv1 "sigs.k8s.io/cluster-api/api/core/v1beta2"
)

const (
    OpenFuyaoClusterFinalizer = "openfuyaocluster.infrastructure.cluster.x-k8s.io"
)

// OpenFuyaoCluster 定义UPI模式的集群基础设施
type OpenFuyaoCluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   OpenFuyaoClusterSpec   `json:"spec,omitempty"`
    Status OpenFuyaoClusterStatus `json:"status,omitempty"`
}

type OpenFuyaoClusterSpec struct {
    // ControlPlaneEndpoint 用户提供的控制面端点
    // +optional
    ControlPlaneEndpoint clusterv1.APIEndpoint `json:"controlPlaneEndpoint,omitempty"`
    
    // Nodes 用户提供的节点列表
    // +kubebuilder:validation:MinItems=1
    Nodes []NodeSpec `json:"nodes"`
    
    // Network 网络配置
    Network NetworkSpec `json:"network,omitempty"`
    
    // Version Kubernetes版本
    Version string `json:"version"`
    
    // UpgradeConfig 升级配置
    UpgradeConfig UpgradeConfig `json:"upgradeConfig,omitempty"`
}

type NodeSpec struct {
    // Name 节点名称
    Name string `json:"name"`
    
    // Address 节点IP地址
    Address string `json:"address"`
    
    // Roles 节点角色
    Roles []NodeRole `json:"roles"`
    
    // SSHConfig SSH连接配置
    SSHConfig SSHConfig `json:"sshConfig"`
    
    // Labels 节点标签
    Labels map[string]string `json:"labels,omitempty"`
    
    // Taints 节点污点
    Taints []corev1.Taint `json:"taints,omitempty"`
}

type NodeRole string

const (
    NodeRoleControlPlane NodeRole = "control-plane"
    NodeRoleWorker       NodeRole = "worker"
)

type SSHConfig struct {
    // Username SSH用户名
    Username string `json:"username"`
    
    // Port SSH端口
    Port int32 `json:"port,omitempty"`
    
    // PrivateKeySecretRef SSH私钥Secret引用
    PrivateKeySecretRef *corev1.SecretReference `json:"privateKeySecretRef"`
}

type NetworkSpec struct {
    // PodCIDR Pod网络CIDR
    PodCIDR string `json:"podCIDR,omitempty"`
    
    // ServiceCIDR Service网络CIDR
    ServiceCIDR string `json:"serviceCIDR,omitempty"`
    
    // CNI CNI插件配置
    CNI CNIConfig `json:"cni,omitempty"`
}

type CNIConfig struct {
    // Type CNI类型
    Type string `json:"type"`
    
    // Config CNI配置
    Config string `json:"config,omitempty"`
}

type UpgradeConfig struct {
    // Strategy 升级策略
    Strategy UpgradeStrategy `json:"strategy,omitempty"`
    
    // DrainOptions 节点排水选项
    DrainOptions DrainOptions `json:"drainOptions,omitempty"`
}

type UpgradeStrategy struct {
    // Type 升级类型
    Type UpgradeType `json:"type"`
    
    // MaxUnavailable 最大不可用节点数
    MaxUnavailable *intstr.IntOrString `json:"maxUnavailable,omitempty"`
    
    // MaxSurge 最大激增节点数
    MaxSurge *intstr.IntOrString `json:"maxSurge,omitempty"`
}

type UpgradeType string

const (
    UpgradeTypeRollingUpdate UpgradeType = "RollingUpdate"
    UpgradeTypeOnDelete      UpgradeType = "OnDelete"
)

type DrainOptions struct {
    // Timeout 排水超时时间
    Timeout *metav1.Duration `json:"timeout,omitempty"`
    
    // GracePeriodSeconds 优雅终止时间
    GracePeriodSeconds *int32 `json:"gracePeriodSeconds,omitempty"`
    
    // DeleteEmptyDirData 是否删除emptyDir数据
    DeleteEmptyDirData bool `json:"deleteEmptyDirData,omitempty"`
    
    // DisableEviction 是否禁用驱逐
    DisableEviction bool `json:"disableEviction,omitempty"`
}

type OpenFuyaoClusterStatus struct {
    // Initialization 初始化状态
    Initialization OpenFuyaoClusterInitializationStatus `json:"initialization,omitempty"`
    
    // Conditions 条件状态
    Conditions []clusterv1.Condition `json:"conditions,omitempty"`
    
    // FailureDomains 故障域
    FailureDomains []clusterv1.FailureDomain `json:"failureDomains,omitempty"`
    
    // Ready 集群是否就绪
    Ready bool `json:"ready"`
    
    // Version 当前集群版本
    Version string `json:"version,omitempty"`
    
    // UpgradeStatus 升级状态
    UpgradeStatus *UpgradeStatus `json:"upgradeStatus,omitempty"`
}

type OpenFuyaoClusterInitializationStatus struct {
    // Provisioned 基础设施是否已就绪
    Provisioned *bool `json:"provisioned,omitempty"`
}

type UpgradeStatus struct {
    // Phase 升级阶段
    Phase UpgradePhase `json:"phase"`
    
    // CurrentVersion 当前版本
    CurrentVersion string `json:"currentVersion"`
    
    // TargetVersion 目标版本
    TargetVersion string `json:"targetVersion"`
    
    // UpgradingNodes 正在升级的节点
    UpgradingNodes []string `json:"upgradingNodes,omitempty"`
    
    // UpgradedNodes 已升级的节点
    UpgradedNodes []string `json:"upgradedNodes,omitempty"`
    
    // FailedNodes 升级失败的节点
    FailedNodes []string `json:"failedNodes,omitempty"`
    
    // StartTime 升级开始时间
    StartTime *metav1.Time `json:"startTime,omitempty"`
    
    // CompletionTime 升级完成时间
    CompletionTime *metav1.Time `json:"completionTime,omitempty"`
}

type UpgradePhase string

const (
    UpgradePhasePending    UpgradePhase = "Pending"
    UpgradePhasePreparing  UpgradePhase = "Preparing"
    UpgradePhaseUpgrading  UpgradePhase = "Upgrading"
    UpgradePhaseVerifying  UpgradePhase = "Verifying"
    UpgradePhaseCompleted  UpgradePhase = "Completed"
    UpgradePhaseFailed     UpgradePhase = "Failed"
    UpgradePhaseRollback   UpgradePhase = "Rollback"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

type OpenFuyaoClusterList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []OpenFuyaoCluster `json:"items"`
}
```
#### OpenFuyaoMachine CRD
```go
// d:\code\github\cluster-api-provider-openfuyao\api\v1beta1\openfuyaomachine_types.go

package v1beta1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    clusterv1 "sigs.k8s.io/cluster-api/api/core/v1beta2"
)

const (
    OpenFuyaoMachineFinalizer = "openfuyaomachine.infrastructure.cluster.x-k8s.io"
)

type OpenFuyaoMachine struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   OpenFuyaoMachineSpec   `json:"spec,omitempty"`
    Status OpenFuyaoMachineStatus `json:"status,omitempty"`
}

type OpenFuyaoMachineSpec struct {
    // ProviderID 节点Provider ID
    ProviderID *string `json:"providerID,omitempty"`
    
    // NodeRef 节点引用
    NodeRef *corev1.ObjectReference `json:"nodeRef,omitempty"`
    
    // Address 节点地址
    Address string `json:"address"`
    
    // SSHConfig SSH配置
    SSHConfig SSHConfig `json:"sshConfig"`
}

type OpenFuyaoMachineStatus struct {
    // Initialization 初始化状态
    Initialization OpenFuyaoMachineInitializationStatus `json:"initialization,omitempty"`
    
    // Conditions 条件状态
    Conditions []clusterv1.Condition `json:"conditions,omitempty"`
    
    // Ready 机器是否就绪
    Ready bool `json:"ready"`
    
    // NodeInfo 节点信息
    NodeInfo *corev1.NodeSystemInfo `json:"nodeInfo,omitempty"`
    
    // Version 节点版本
    Version string `json:"version,omitempty"`
    
    // UpgradePhase 升级阶段
    UpgradePhase MachineUpgradePhase `json:"upgradePhase,omitempty"`
}

type OpenFuyaoMachineInitializationStatus struct {
    // Provisioned 基础设施是否已就绪
    Provisioned *bool `json:"provisioned,omitempty"`
    
    // BootstrapDataReady 引导数据是否就绪
    BootstrapDataReady *bool `json:"bootstrapDataReady,omitempty"`
}

type MachineUpgradePhase string

const (
    MachineUpgradePhaseNone       MachineUpgradePhase = ""
    MachineUpgradePhasePreparing  MachineUpgradePhase = "Preparing"
    MachineUpgradePhaseDraining   MachineUpgradePhase = "Draining"
    MachineUpgradePhaseUpgrading  MachineUpgradePhase = "Upgrading"
    MachineUpgradePhaseVerifying  MachineUpgradePhase = "Verifying"
    MachineUpgradePhaseCompleted  MachineUpgradePhase = "Completed"
    MachineUpgradePhaseFailed     MachineUpgradePhase = "Failed"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

type OpenFuyaoMachineList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []OpenFuyaoMachine `json:"items"`
}
```
### 2.3 Controller实现
#### OpenFuyaoCluster Controller

```go
// d:\code\github\cluster-api-provider-openfuyao\controllers\openfuyaocluster_controller.go

package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/pkg/errors"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/klog/v2"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    
    infrav1 "sigs.k8s.io/cluster-api-provider-openfuyao/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/core/v1beta2"
    "sigs.k8s.io/cluster-api/util"
    "sigs.k8s.io/cluster-api/util/conditions"
    "sigs.k8s.io/cluster-api/util/patch"
    "sigs.k8s.io/cluster-api/util/predicates"
)

type OpenFuyaoClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    
    WatchFilterValue string
}

func (r *OpenFuyaoClusterReconciler) SetupWithManager(ctx context.Context, mgr ctrl.Manager) error {
    log := ctrl.LoggerFrom(ctx)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&infrav1.OpenFuyaoCluster{}).
        WithEventFilter(predicates.ResourceHasFilterLabel(mgr.GetScheme(), log, r.WatchFilterValue)).
        Watches(
            &clusterv1.Cluster{},
            handler.EnqueueRequestsFromMapFunc(r.clusterToOpenFuyaoCluster),
        ).
        Complete(r)
}

func (r *OpenFuyaoClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (_ ctrl.Result, reterr error) {
    log := ctrl.LoggerFrom(ctx)
    
    openFuyaoCluster := &infrav1.OpenFuyaoCluster{}
    if err := r.Client.Get(ctx, req.NamespacedName, openFuyaoCluster); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    cluster, err := util.GetOwnerCluster(ctx, r.Client, openFuyaoCluster.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    if cluster == nil {
        log.Info("Waiting for Cluster Controller to set OwnerRef on OpenFuyaoCluster")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    patchHelper, err := patch.NewHelper(openFuyaoCluster, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    defer func() {
        if err := patchHelper.Patch(ctx, openFuyaoCluster); err != nil && reterr == nil {
            reterr = err
        }
    }()
    
    if !openFuyaoCluster.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, openFuyaoCluster)
    }
    
    return r.reconcileNormal(ctx, openFuyaoCluster, cluster)
}

func (r *OpenFuyaoClusterReconciler) reconcileNormal(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, cluster *clusterv1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    controllerutil.AddFinalizer(openFuyaoCluster, infrav1.OpenFuyaoClusterFinalizer)
    
    if openFuyaoCluster.Spec.ControlPlaneEndpoint.IsZero() {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, "WaitingForControlPlaneEndpoint", clusterv1.ConditionSeverityInfo, "")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    if len(openFuyaoCluster.Spec.Nodes) == 0 {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, "WaitingForNodes", clusterv1.ConditionSeverityInfo, "")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    if err := r.validateNodes(ctx, openFuyaoCluster); err != nil {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, "NodesValidationFailed", clusterv1.ConditionSeverityError, err.Error())
        return ctrl.Result{}, err
    }
    
    openFuyaoCluster.Status.Initialization.Provisioned = util.PtrTo(true)
    openFuyaoCluster.Status.Ready = true
    openFuyaoCluster.Status.Version = openFuyaoCluster.Spec.Version
    
    conditions.MarkTrue(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition)
    
    log.Info("OpenFuyaoCluster is ready")
    return ctrl.Result{}, nil
}

func (r *OpenFuyaoClusterReconciler) reconcileDelete(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    log.Info("Deleting OpenFuyaoCluster")
    
    controllerutil.RemoveFinalizer(openFuyaoCluster, infrav1.OpenFuyaoClusterFinalizer)
    
    return ctrl.Result{}, nil
}

func (r *OpenFuyaoClusterReconciler) validateNodes(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    controlPlaneNodes := 0
    for _, node := range openFuyaoCluster.Spec.Nodes {
        for _, role := range node.Roles {
            if role == infrav1.NodeRoleControlPlane {
                controlPlaneNodes++
            }
        }
    }
    
    if controlPlaneNodes == 0 {
        return errors.New("at least one control plane node is required")
    }
    
    if controlPlaneNodes%2 == 0 {
        return errors.New("control plane nodes should be odd number for HA")
    }
    
    return nil
}

func (r *OpenFuyaoClusterReconciler) clusterToOpenFuyaoCluster(ctx context.Context, o client.Object) []ctrl.Request {
    cluster := o.(*clusterv1.Cluster)
    
    if cluster.Spec.InfrastructureRef == nil {
        return nil
    }
    
    if cluster.Spec.InfrastructureRef.GroupVersionKind().Group != infrav1.GroupVersion.Group {
        return nil
    }
    
    return []ctrl.Request{
        {
            NamespacedName: client.ObjectKey{
                Name:      cluster.Spec.InfrastructureRef.Name,
                Namespace: cluster.Namespace,
            },
        },
    }
}
```
#### OpenFuyaoMachine Controller
```go
// d:\code\github\cluster-api-provider-openfuyao\controllers\openfuyaomachine_controller.go

package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/pkg/errors"
    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/klog/v2"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    
    infrav1 "sigs.k8s.io/cluster-api-provider-openfuyao/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/core/v1beta2"
    "sigs.k8s.io/cluster-api/util"
    "sigs.k8s.io/cluster-api/util/conditions"
    "sigs.k8s.io/cluster-api/util/patch"
    "sigs.k8s.io/cluster-api/util/predicates"
)

type OpenFuyaoMachineReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    
    WatchFilterValue string
}

func (r *OpenFuyaoMachineReconciler) SetupWithManager(ctx context.Context, mgr ctrl.Manager) error {
    log := ctrl.LoggerFrom(ctx)
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&infrav1.OpenFuyaoMachine{}).
        WithEventFilter(predicates.ResourceHasFilterLabel(mgr.GetScheme(), log, r.WatchFilterValue)).
        Watches(
            &clusterv1.Machine{},
            handler.EnqueueRequestsFromMapFunc(r.machineToOpenFuyaoMachine),
        ).
        Complete(r)
}

func (r *OpenFuyaoMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (_ ctrl.Result, reterr error) {
    log := ctrl.LoggerFrom(ctx)
    
    openFuyaoMachine := &infrav1.OpenFuyaoMachine{}
    if err := r.Client.Get(ctx, req.NamespacedName, openFuyaoMachine); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    machine, err := util.GetOwnerMachine(ctx, r.Client, openFuyaoMachine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    if machine == nil {
        log.Info("Waiting for Machine Controller to set OwnerRef on OpenFuyaoMachine")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    cluster, err := util.GetClusterFromMetadata(ctx, r.Client, openFuyaoMachine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    if cluster == nil {
        log.Info("Waiting for Cluster to be set on Machine")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    patchHelper, err := patch.NewHelper(openFuyaoMachine, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    defer func() {
        if err := patchHelper.Patch(ctx, openFuyaoMachine); err != nil && reterr == nil {
            reterr = err
        }
    }()
    
    if !openFuyaoMachine.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, openFuyaoMachine, machine, cluster)
    }
    
    return r.reconcileNormal(ctx, openFuyaoMachine, machine, cluster)
}

func (r *OpenFuyaoMachineReconciler) reconcileNormal(ctx context.Context, openFuyaoMachine *infrav1.OpenFuyaoMachine, machine *clusterv1.Machine, cluster *clusterv1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    controllerutil.AddFinalizer(openFuyaoMachine, infrav1.OpenFuyaoMachineFinalizer)
    
    if machine.Spec.Bootstrap.DataSecretName == nil {
        conditions.MarkFalse(openFuyaoMachine, clusterv1.InfrastructureReadyV1Beta2Condition, "WaitingForBootstrapData", clusterv1.ConditionSeverityInfo, "")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    providerID := fmt.Sprintf("openfuyao://%s/%s", cluster.Name, openFuyaoMachine.Name)
    openFuyaoMachine.Spec.ProviderID = &providerID
    
    openFuyaoMachine.Status.Initialization.Provisioned = util.PtrTo(true)
    openFuyaoMachine.Status.Initialization.BootstrapDataReady = util.PtrTo(true)
    openFuyaoMachine.Status.Ready = true
    
    conditions.MarkTrue(openFuyaoMachine, clusterv1.InfrastructureReadyV1Beta2Condition)
    
    log.Info("OpenFuyaoMachine is ready")
    return ctrl.Result{}, nil
}

func (r *OpenFuyaoMachineReconciler) reconcileDelete(ctx context.Context, openFuyaoMachine *infrav1.OpenFuyaoMachine, machine *clusterv1.Machine, cluster *clusterv1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    log.Info("Deleting OpenFuyaoMachine")
    
    controllerutil.RemoveFinalizer(openFuyaoMachine, infrav1.OpenFuyaoMachineFinalizer)
    
    return ctrl.Result{}, nil
}

func (r *OpenFuyaoMachineReconciler) machineToOpenFuyaoMachine(ctx context.Context, o client.Object) []ctrl.Request {
    machine := o.(*clusterv1.Machine)
    
    if machine.Spec.InfrastructureRef == nil {
        return nil
    }
    
    if machine.Spec.InfrastructureRef.GroupVersionKind().Group != infrav1.GroupVersion.Group {
        return nil
    }
    
    return []ctrl.Request{
        {
            NamespacedName: client.ObjectKey{
                Name:      machine.Spec.InfrastructureRef.Name,
                Namespace: machine.Namespace,
            },
        },
    }
}
```
## 三、UPI模式实现要点
### 3.1 用户提供的资源管理
UPI模式的核心是用户提供基础设施,Provider仅管理集群配置:
```go
// 用户提供的资源验证
func (r *OpenFuyaoClusterReconciler) validateUserProvidedResources(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    // 1. 验证控制面端点
    if openFuyaoCluster.Spec.ControlPlaneEndpoint.IsZero() {
        return errors.New("control plane endpoint must be provided by user")
    }
    
    // 2. 验证节点列表
    if len(openFuyaoCluster.Spec.Nodes) == 0 {
        return errors.New("at least one node must be provided by user")
    }
    
    // 3. 验证SSH连接
    for _, node := range openFuyaoCluster.Spec.Nodes {
        if err := r.validateSSHConnection(ctx, node); err != nil {
            return errors.Wrapf(err, "failed to validate SSH connection for node %s", node.Name)
        }
    }
    
    return nil
}

func (r *OpenFuyaoClusterReconciler) validateSSHConnection(ctx context.Context, node infrav1.NodeSpec) error {
    // 获取SSH私钥
    secret := &corev1.Secret{}
    secretKey := client.ObjectKey{
        Name:      node.SSHConfig.PrivateKeySecretRef.Name,
        Namespace: node.SSHConfig.PrivateKeySecretRef.Namespace,
    }
    
    if err := r.Client.Get(ctx, secretKey, secret); err != nil {
        return errors.Wrap(err, "failed to get SSH private key secret")
    }
    
    privateKey, ok := secret.Data["private-key"]
    if !ok {
        return errors.New("SSH private key not found in secret")
    }
    
    // 测试SSH连接
    sshClient := NewSSHClient(node.Address, node.SSHConfig.Username, privateKey, node.SSHConfig.Port)
    if err := sshClient.Connect(ctx); err != nil {
        return errors.Wrap(err, "failed to connect via SSH")
    }
    defer sshClient.Close()
    
    return nil
}
```
### 3.2 集群安装流程
```go
// d:\code\github\cluster-api-provider-openfuyao\pkg\install\installer.go

package install

import (
    "context"
    "fmt"
    "time"
    
    "github.com/pkg/errors"
    "k8s.io/klog/v2"
    
    infrav1 "sigs.k8s.io/cluster-api-provider-openfuyao/api/v1beta1"
)

type Installer struct {
    sshClientFactory SSHClientFactory
    kubeadmExecutor  KubeadmExecutor
}

func NewInstaller() *Installer {
    return &Installer{
        sshClientFactory: NewSSHClientFactory(),
        kubeadmExecutor:  NewKubeadmExecutor(),
    }
}

func (i *Installer) InstallCluster(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    log := klog.FromContext(ctx)
    
    log.Info("Starting cluster installation")
    
    // 1. 初始化控制面节点
    controlPlaneNodes := i.getControlPlaneNodes(openFuyaoCluster)
    if len(controlPlaneNodes) == 0 {
        return errors.New("no control plane nodes found")
    }
    
    // 2. 初始化第一个控制面节点
    firstControlPlaneNode := controlPlaneNodes[0]
    if err := i.initializeFirstControlPlane(ctx, openFuyaoCluster, firstControlPlaneNode); err != nil {
        return errors.Wrap(err, "failed to initialize first control plane node")
    }
    
    // 3. 加入其他控制面节点
    for j := 1; j < len(controlPlaneNodes); j++ {
        if err := i.joinControlPlane(ctx, openFuyaoCluster, controlPlaneNodes[j]); err != nil {
            return errors.Wrapf(err, "failed to join control plane node %s", controlPlaneNodes[j].Name)
        }
    }
    
    // 4. 加入Worker节点
    workerNodes := i.getWorkerNodes(openFuyaoCluster)
    for _, workerNode := range workerNodes {
        if err := i.joinWorker(ctx, openFuyaoCluster, workerNode); err != nil {
            return errors.Wrapf(err, "failed to join worker node %s", workerNode.Name)
        }
    }
    
    log.Info("Cluster installation completed successfully")
    return nil
}

func (i *Installer) initializeFirstControlPlane(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    log := klog.FromContext(ctx)
    
    log.Info(fmt.Sprintf("Initializing first control plane node: %s", node.Name))
    
    sshClient, err := i.sshClientFactory.CreateClient(ctx, node)
    if err != nil {
        return errors.Wrap(err, "failed to create SSH client")
    }
    defer sshClient.Close()
    
    // 1. 安装依赖
    if err := i.installDependencies(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "failed to install dependencies")
    }
    
    // 2. 生成kubeadm配置
    kubeadmConfig := i.generateKubeadmConfig(openFuyaoCluster, node)
    
    // 3. 执行kubeadm init
    if err := i.kubeadmExecutor.Init(ctx, sshClient, kubeadmConfig); err != nil {
        return errors.Wrap(err, "failed to execute kubeadm init")
    }
    
    // 4. 安装CNI
    if err := i.installCNI(ctx, sshClient, openFuyaoCluster.Spec.Network.CNI); err != nil {
        return errors.Wrap(err, "failed to install CNI")
    }
    
    log.Info(fmt.Sprintf("First control plane node %s initialized successfully", node.Name))
    return nil
}

func (i *Installer) joinControlPlane(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    log := klog.FromContext(ctx)
    
    log.Info(fmt.Sprintf("Joining control plane node: %s", node.Name))
    
    sshClient, err := i.sshClientFactory.CreateClient(ctx, node)
    if err != nil {
        return errors.Wrap(err, "failed to create SSH client")
    }
    defer sshClient.Close()
    
    // 1. 安装依赖
    if err := i.installDependencies(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "failed to install dependencies")
    }
    
    // 2. 获取join token
    joinToken, err := i.getJoinToken(ctx, openFuyaoCluster)
    if err != nil {
        return errors.Wrap(err, "failed to get join token")
    }
    
    // 3. 执行kubeadm join (control plane)
    if err := i.kubeadmExecutor.JoinControlPlane(ctx, sshClient, joinToken, openFuyaoCluster.Spec.ControlPlaneEndpoint); err != nil {
        return errors.Wrap(err, "failed to execute kubeadm join for control plane")
    }
    
    log.Info(fmt.Sprintf("Control plane node %s joined successfully", node.Name))
    return nil
}

func (i *Installer) joinWorker(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    log := klog.FromContext(ctx)
    
    log.Info(fmt.Sprintf("Joining worker node: %s", node.Name))
    
    sshClient, err := i.sshClientFactory.CreateClient(ctx, node)
    if err != nil {
        return errors.Wrap(err, "failed to create SSH client")
    }
    defer sshClient.Close()
    
    // 1. 安装依赖
    if err := i.installDependencies(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "failed to install dependencies")
    }
    
    // 2. 获取join token
    joinToken, err := i.getJoinToken(ctx, openFuyaoCluster)
    if err != nil {
        return errors.Wrap(err, "failed to get join token")
    }
    
    // 3. 执行kubeadm join (worker)
    if err := i.kubeadmExecutor.JoinWorker(ctx, sshClient, joinToken, openFuyaoCluster.Spec.ControlPlaneEndpoint); err != nil {
        return errors.Wrap(err, "failed to execute kubeadm join for worker")
    }
    
    log.Info(fmt.Sprintf("Worker node %s joined successfully", node.Name))
    return nil
}

func (i *Installer) installDependencies(ctx context.Context, sshClient SSHClient, version string) error {
    // 安装containerd、kubelet、kubeadm、kubectl
    commands := []string{
        "apt-get update",
        "apt-get install -y containerd",
        fmt.Sprintf("apt-get install -y kubelet=%s kubeadm=%s kubectl=%s", version, version, version),
        "systemctl enable kubelet",
    }
    
    for _, cmd := range commands {
        if err := sshClient.Execute(ctx, cmd); err != nil {
            return errors.Wrapf(err, "failed to execute command: %s", cmd)
        }
    }
    
    return nil
}

func (i *Installer) generateKubeadmConfig(openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) string {
    return fmt.Sprintf(`
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: %s
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: %s
controlPlaneEndpoint: "%s:%d"
networking:
  podSubnet: %s
  serviceSubnet: %s
`, node.Address, openFuyaoCluster.Spec.Version, 
   openFuyaoCluster.Spec.ControlPlaneEndpoint.Host, 
   openFuyaoCluster.Spec.ControlPlaneEndpoint.Port,
   openFuyaoCluster.Spec.Network.PodCIDR,
   openFuyaoCluster.Spec.Network.ServiceCIDR)
}

func (i *Installer) installCNI(ctx context.Context, sshClient SSHClient, cniConfig infrav1.CNIConfig) error {
    // 安装CNI插件
    if cniConfig.Type == "calico" {
        return sshClient.Execute(ctx, "kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml")
    }
    return nil
}

func (i *Installer) getJoinToken(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) (string, error) {
    // 从第一个控制面节点获取join token
    controlPlaneNodes := i.getControlPlaneNodes(openFuyaoCluster)
    if len(controlPlaneNodes) == 0 {
        return "", errors.New("no control plane nodes found")
    }
    
    sshClient, err := i.sshClientFactory.CreateClient(ctx, controlPlaneNodes[0])
    if err != nil {
        return "", errors.Wrap(err, "failed to create SSH client")
    }
    defer sshClient.Close()
    
    output, err := sshClient.ExecuteWithOutput(ctx, "kubeadm token create --print-join-command")
    if err != nil {
        return "", errors.Wrap(err, "failed to create join token")
    }
    
    return output, nil
}

func (i *Installer) getControlPlaneNodes(openFuyaoCluster *infrav1.OpenFuyaoCluster) []infrav1.NodeSpec {
    var controlPlaneNodes []infrav1.NodeSpec
    for _, node := range openFuyaoCluster.Spec.Nodes {
        for _, role := range node.Roles {
            if role == infrav1.NodeRoleControlPlane {
                controlPlaneNodes = append(controlPlaneNodes, node)
                break
            }
        }
    }
    return controlPlaneNodes
}

func (i *Installer) getWorkerNodes(openFuyaoCluster *infrav1.OpenFuyaoCluster) []infrav1.NodeSpec {
    var workerNodes []infrav1.NodeSpec
    for _, node := range openFuyaoCluster.Spec.Nodes {
        for _, role := range node.Roles {
            if role == infrav1.NodeRoleWorker {
                workerNodes = append(workerNodes, node)
                break
            }
        }
    }
    return workerNodes
}
```
### 3.3 集群升级流程
```go
// d:\code\github\cluster-api-provider-openfuyao\pkg\upgrade\upgrader.go

package upgrade

import (
    "context"
    "fmt"
    "time"
    
    "github.com/pkg/errors"
    "k8s.io/apimachinery/pkg/util/sets"
    "k8s.io/klog/v2"
    
    infrav1 "sigs.k8s.io/cluster-api-provider-openfuyao/api/v1beta1"
)

type Upgrader struct {
    sshClientFactory SSHClientFactory
    kubeadmExecutor  KubeadmExecutor
    drainManager     DrainManager
}

func NewUpgrader() *Upgrader {
    return &Upgrader{
        sshClientFactory: NewSSHClientFactory(),
        kubeadmExecutor:  NewKubeadmExecutor(),
        drainManager:     NewDrainManager(),
    }
}

func (u *Upgrader) UpgradeCluster(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    log := klog.FromContext(ctx)
    
    log.Info(fmt.Sprintf("Starting cluster upgrade from %s to %s", openFuyaoCluster.Status.Version, openFuyaoCluster.Spec.Version))
    
    // 1. 验证升级路径
    if err := u.validateUpgradePath(openFuyaoCluster.Status.Version, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "invalid upgrade path")
    }
    
    // 2. 初始化升级状态
    openFuyaoCluster.Status.UpgradeStatus = &infrav1.UpgradeStatus{
        Phase:          infrav1.UpgradePhasePreparing,
        CurrentVersion: openFuyaoCluster.Status.Version,
        TargetVersion:  openFuyaoCluster.Spec.Version,
        StartTime:      &metav1.Time{Time: time.Now()},
    }
    
    // 3. 升级控制面节点
    if err := u.upgradeControlPlane(ctx, openFuyaoCluster); err != nil {
        openFuyaoCluster.Status.UpgradeStatus.Phase = infrav1.UpgradePhaseFailed
        return errors.Wrap(err, "failed to upgrade control plane")
    }
    
    // 4. 升级Worker节点
    if err := u.upgradeWorkers(ctx, openFuyaoCluster); err != nil {
        openFuyaoCluster.Status.UpgradeStatus.Phase = infrav1.UpgradePhaseFailed
        return errors.Wrap(err, "failed to upgrade workers")
    }
    
    // 5. 完成升级
    openFuyaoCluster.Status.UpgradeStatus.Phase = infrav1.UpgradePhaseCompleted
    openFuyaoCluster.Status.UpgradeStatus.CompletionTime = &metav1.Time{Time: time.Now()}
    openFuyaoCluster.Status.Version = openFuyaoCluster.Spec.Version
    
    log.Info("Cluster upgrade completed successfully")
    return nil
}

func (u *Upgrader) upgradeControlPlane(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    log := klog.FromContext(ctx)
    
    controlPlaneNodes := u.getControlPlaneNodes(openFuyaoCluster)
    
    for _, node := range controlPlaneNodes {
        log.Info(fmt.Sprintf("Upgrading control plane node: %s", node.Name))
        
        // 1. 更新状态
        openFuyaoCluster.Status.UpgradeStatus.UpgradingNodes = append(openFuyaoCluster.Status.UpgradeStatus.UpgradingNodes, node.Name)
        openFuyaoCluster.Status.UpgradeStatus.Phase = infrav1.UpgradePhaseUpgrading
        
        // 2. 升级节点
        if err := u.upgradeNode(ctx, openFuyaoCluster, node, true); err != nil {
            openFuyaoCluster.Status.UpgradeStatus.FailedNodes = append(openFuyaoCluster.Status.UpgradeStatus.FailedNodes, node.Name)
            return errors.Wrapf(err, "failed to upgrade control plane node %s", node.Name)
        }
        
        // 3. 验证节点
        if err := u.verifyNode(ctx, openFuyaoCluster, node); err != nil {
            return errors.Wrapf(err, "failed to verify control plane node %s", node.Name)
        }
        
        // 4. 更新状态
        openFuyaoCluster.Status.UpgradeStatus.UpgradedNodes = append(openFuyaoCluster.Status.UpgradeStatus.UpgradedNodes, node.Name)
        
        log.Info(fmt.Sprintf("Control plane node %s upgraded successfully", node.Name))
    }
    
    return nil
}

func (u *Upgrader) upgradeWorkers(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    log := klog.FromContext(ctx)
    
    workerNodes := u.getWorkerNodes(openFuyaoCluster)
    
    // 根据升级策略确定并发升级数量
    maxUnavailable := 1
    if openFuyaoCluster.Spec.UpgradeConfig.Strategy.MaxUnavailable != nil {
        maxUnavailable = openFuyaoCluster.Spec.UpgradeConfig.Strategy.MaxUnavailable.IntValue()
    }
    
    // 分批升级Worker节点
    for i := 0; i < len(workerNodes); i += maxUnavailable {
        end := i + maxUnavailable
        if end > len(workerNodes) {
            end = len(workerNodes)
        }
        
        batch := workerNodes[i:end]
        
        // 并发升级一批节点
        errChan := make(chan error, len(batch))
        for _, node := range batch {
            go func(n infrav1.NodeSpec) {
                errChan <- u.upgradeWorkerNode(ctx, openFuyaoCluster, n)
            }(node)
        }
        
        // 等待所有节点升级完成
        for range batch {
            if err := <-errChan; err != nil {
                return err
            }
        }
    }
    
    return nil
}

func (u *Upgrader) upgradeWorkerNode(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    log := klog.FromContext(ctx)
    
    log.Info(fmt.Sprintf("Upgrading worker node: %s", node.Name))
    
    // 1. 更新状态
    openFuyaoCluster.Status.UpgradeStatus.UpgradingNodes = append(openFuyaoCluster.Status.UpgradeStatus.UpgradingNodes, node.Name)
    
    // 2. 排水节点
    if err := u.drainNode(ctx, openFuyaoCluster, node); err != nil {
        openFuyaoCluster.Status.UpgradeStatus.FailedNodes = append(openFuyaoCluster.Status.UpgradeStatus.FailedNodes, node.Name)
        return errors.Wrapf(err, "failed to drain worker node %s", node.Name)
    }
    
    // 3. 升级节点
    if err := u.upgradeNode(ctx, openFuyaoCluster, node, false); err != nil {
        openFuyaoCluster.Status.UpgradeStatus.FailedNodes = append(openFuyaoCluster.Status.UpgradeStatus.FailedNodes, node.Name)
        return errors.Wrapf(err, "failed to upgrade worker node %s", node.Name)
    }
    
    // 4. 验证节点
    if err := u.verifyNode(ctx, openFuyaoCluster, node); err != nil {
        return errors.Wrapf(err, "failed to verify worker node %s", node.Name)
    }
    
    // 5. 恢复节点
    if err := u.uncordonNode(ctx, openFuyaoCluster, node); err != nil {
        return errors.Wrapf(err, "failed to uncordon worker node %s", node.Name)
    }
    
    // 6. 更新状态
    openFuyaoCluster.Status.UpgradeStatus.UpgradedNodes = append(openFuyaoCluster.Status.UpgradeStatus.UpgradedNodes, node.Name)
    
    log.Info(fmt.Sprintf("Worker node %s upgraded successfully", node.Name))
    return nil
}

func (u *Upgrader) upgradeNode(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec, isControlPlane bool) error {
    sshClient, err := u.sshClientFactory.CreateClient(ctx, node)
    if err != nil {
        return errors.Wrap(err, "failed to create SSH client")
    }
    defer sshClient.Close()
    
    // 1. 升级kubeadm
    if err := u.upgradeKubeadm(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "failed to upgrade kubeadm")
    }
    
    // 2. 执行kubeadm upgrade
    if isControlPlane {
        if err := u.kubeadmExecutor.UpgradeControlPlane(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
            return errors.Wrap(err, "failed to upgrade control plane")
        }
    } else {
        if err := u.kubeadmExecutor.UpgradeNode(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
            return errors.Wrap(err, "failed to upgrade node")
        }
    }
    
    // 3. 升级kubelet和kubectl
    if err := u.upgradeKubeletAndKubectl(ctx, sshClient, openFuyaoCluster.Spec.Version); err != nil {
        return errors.Wrap(err, "failed to upgrade kubelet and kubectl")
    }
    
    // 4. 重启kubelet
    if err := sshClient.Execute(ctx, "systemctl restart kubelet"); err != nil {
        return errors.Wrap(err, "failed to restart kubelet")
    }
    
    return nil
}

func (u *Upgrader) drainNode(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    return u.drainManager.Drain(ctx, openFuyaoCluster, node, openFuyaoCluster.Spec.UpgradeConfig.DrainOptions)
}

func (u *Upgrader) uncordonNode(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    return u.drainManager.Uncordon(ctx, openFuyaoCluster, node)
}

func (u *Upgrader) verifyNode(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, node infrav1.NodeSpec) error {
    // 等待节点Ready
    // TODO: 实现节点验证逻辑
    time.Sleep(30 * time.Second)
    return nil
}

func (u *Upgrader) validateUpgradePath(currentVersion, targetVersion string) error {
    // 验证版本升级路径
    // TODO: 实现版本验证逻辑
    return nil
}

func (u *Upgrader) upgradeKubeadm(ctx context.Context, sshClient SSHClient, version string) error {
    cmd := fmt.Sprintf("apt-get install -y kubeadm=%s", version)
    return sshClient.Execute(ctx, cmd)
}

func (u *Upgrader) upgradeKubeletAndKubectl(ctx context.Context, sshClient SSHClient, version string) error {
    commands := []string{
        fmt.Sprintf("apt-get install -y kubelet=%s kubectl=%s", version, version),
        "systemctl daemon-reload",
    }
    
    for _, cmd := range commands {
        if err := sshClient.Execute(ctx, cmd); err != nil {
            return err
        }
    }
    
    return nil
}

func (u *Upgrader) getControlPlaneNodes(openFuyaoCluster *infrav1.OpenFuyaoCluster) []infrav1.NodeSpec {
    var controlPlaneNodes []infrav1.NodeSpec
    for _, node := range openFuyaoCluster.Spec.Nodes {
        for _, role := range node.Roles {
            if role == infrav1.NodeRoleControlPlane {
                controlPlaneNodes = append(controlPlaneNodes, node)
                break
            }
        }
    }
    return controlPlaneNodes
}

func (u *Upgrader) getWorkerNodes(openFuyaoCluster *infrav1.OpenFuyaoCluster) []infrav1.NodeSpec {
    var workerNodes []infrav1.NodeSpec
    for _, node := range openFuyaoCluster.Spec.Nodes {
        for _, role := range node.Roles {
            if role == infrav1.NodeRoleWorker {
                workerNodes = append(workerNodes, node)
                break
            }
        }
    }
    return workerNodes
}
```
## 四、开发最佳实践
### 4.1 错误处理
```go
// 使用条件状态报告错误
func (r *OpenFuyaoClusterReconciler) handleError(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster, err error) {
    log := ctrl.LoggerFrom(ctx)
    
    if errors.Is(err, ErrSSHConnectionFailed) {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, 
            "SSHConnectionFailed", clusterv1.ConditionSeverityError, err.Error())
        log.Error(err, "SSH connection failed")
    } else if errors.Is(err, ErrNodeValidationFailed) {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, 
            "NodeValidationFailed", clusterv1.ConditionSeverityError, err.Error())
        log.Error(err, "Node validation failed")
    } else {
        conditions.MarkFalse(openFuyaoCluster, clusterv1.InfrastructureReadyV1Beta2Condition, 
            "InternalError", clusterv1.ConditionSeverityError, err.Error())
        log.Error(err, "Internal error occurred")
    }
}
```
### 4.2 状态管理
```go
// 使用patch helper管理状态更新
func (r *OpenFuyaoClusterReconciler) updateStatus(ctx context.Context, openFuyaoCluster *infrav1.OpenFuyaoCluster) error {
    patchHelper, err := patch.NewHelper(openFuyaoCluster, r.Client)
    if err != nil {
        return err
    }
    
    return patchHelper.Patch(ctx, openFuyaoCluster)
}
```
### 4.3 测试
```go
// d:\code\github\cluster-api-provider-openfuyao\controllers\openfuyaocluster_controller_test.go

package controllers

import (
    "context"
    "testing"
    
    "github.com/stretchr/testify/assert"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client/fake"
    
    infrav1 "sigs.k8s.io/cluster-api-provider-openfuyao/api/v1beta1"
    clusterv1 "sigs.k8s.io/cluster-api/api/core/v1beta2"
)

func TestOpenFuyaoClusterReconciler_Reconcile(t *testing.T) {
    scheme := runtime.NewScheme()
    _ = infrav1.AddToScheme(scheme)
    _ = clusterv1.AddToScheme(scheme)
    
    tests := []struct {
        name           string
        cluster        *clusterv1.Cluster
        openFuyaoCluster *infrav1.OpenFuyaoCluster
        expectReady    bool
    }{
        {
            name: "cluster with valid nodes should be ready",
            cluster: &clusterv1.Cluster{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-cluster",
                    Namespace: "default",
                },
                Spec: clusterv1.ClusterSpec{
                    InfrastructureRef: &corev1.ObjectReference{
                        APIVersion: infrav1.GroupVersion.String(),
                        Kind:       "OpenFuyaoCluster",
                        Name:       "test-openfuyao-cluster",
                    },
                },
            },
            openFuyaoCluster: &infrav1.OpenFuyaoCluster{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-openfuyao-cluster",
                    Namespace: "default",
                    OwnerReferences: []metav1.OwnerReference{
                        {
                            APIVersion: clusterv1.GroupVersion.String(),
                            Kind:       "Cluster",
                            Name:       "test-cluster",
                        },
                    },
                },
                Spec: infrav1.OpenFuyaoClusterSpec{
                    ControlPlaneEndpoint: clusterv1.APIEndpoint{
                        Host: "192.168.1.100",
                        Port: 6443,
                    },
                    Nodes: []infrav1.NodeSpec{
                        {
                            Name:    "node1",
                            Address: "192.168.1.101",
                            Roles:   []infrav1.NodeRole{infrav1.NodeRoleControlPlane},
                        },
                    },
                },
            },
            expectReady: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            client := fake.NewClientBuilder().
                WithScheme(scheme).
                WithObjects(tt.cluster, tt.openFuyaoCluster).
                Build()
            
            reconciler := &OpenFuyaoClusterReconciler{
                Client: client,
                Scheme: scheme,
            }
            
            req := ctrl.Request{
                NamespacedName: client.ObjectKeyFromObject(tt.openFuyaoCluster),
            }
            
            _, err := reconciler.Reconcile(context.Background(), req)
            assert.NoError(t, err)
            
            updatedCluster := &infrav1.OpenFuyaoCluster{}
            _ = client.Get(context.Background(), req.NamespacedName, updatedCluster)
            
            assert.Equal(t, tt.expectReady, updatedCluster.Status.Ready)
        })
    }
}
```
## 五、部署和使用
### 5.1 部署Provider
```yaml
# d:\code\github\cluster-api-provider-openfuyao\config\manager\manager.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-api-provider-openfuyao-controller-manager
  namespace: cluster-api-provider-openfuyao-system
spec:
  replicas: 1
  selector:
    matchLabels:
      control-plane: controller-manager
  template:
    metadata:
      labels:
        control-plane: controller-manager
    spec:
      containers:
      - name: manager
        image: cluster-api-provider-openfuyao:latest
        args:
        - --leader-elect
        - --metrics-bind-address=:8443
        - --health-probe-bind-address=:8081
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 10m
            memory: 64Mi
```
### 5.2 使用示例
```yaml
# 创建OpenFuyao集群
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: openfuyao-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: openfuyao-cluster-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: OpenFuyaoCluster
    name: openfuyao-cluster
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: OpenFuyaoCluster
metadata:
  name: openfuyao-cluster
  namespace: default
spec:
  controlPlaneEndpoint:
    host: 192.168.1.100
    port: 6443
  version: v1.28.5
  nodes:
  - name: control-plane-1
    address: 192.168.1.101
    roles:
    - control-plane
    sshConfig:
      username: root
      port: 22
      privateKeySecretRef:
        name: ssh-private-key
        namespace: default
  - name: worker-1
    address: 192.168.1.102
    roles:
    - worker
    sshConfig:
      username: root
      port: 22
      privateKeySecretRef:
        name: ssh-private-key
        namespace: default
  network:
    podCIDR: 192.168.0.0/16
    serviceCIDR: 10.96.0.0/12
    cni:
      type: calico
  upgradeConfig:
    strategy:
      type: RollingUpdate
      maxUnavailable: 1
    drainOptions:
      timeout: 5m
      gracePeriodSeconds: 30
      deleteEmptyDirData: true
```
## 六、总结
本指南详细介绍了如何基于Cluster API开发UPI模式的openFuyao集群安装升级功能,包括:
1. **CRD设计**: 定义了OpenFuyaoCluster和OpenFuyaoMachine资源
2. **Controller实现**: 实现了集群和机器的控制器逻辑
3. **安装流程**: 实现了基于kubeadm的集群安装流程
4. **升级流程**: 实现了滚动升级和节点排水功能
5. **最佳实践**: 包括错误处理、状态管理和测试

通过遵循Cluster API的Provider契约和最佳实践,可以开发出稳定、可靠的UPI模式集群管理功能。
        
# 一个详细的 Cluster API 开发指南：
## Cluster API 开发指南 - UPI 模式 openFuyao 集群安装升级
### 一、开发环境准备
```bash
# 1. 安装必要工具
go install sigs.k8s.io/kubebuilder/v3/cmd/kubebuilder@latest
go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest
go install k8s.io/code-generator/cmd/register-gen@latest

# 2. 克隆项目
git clone https://github.com/openfuyao/cluster-api-provider-bke.git
cd cluster-api-provider-bke

# 3. 安装依赖
go mod download
```
### 二、项目结构说明
```
cluster-api-provider-bke/
├── api/
│   ├── capbke/v1beta1/          # BKECluster, BKEMachine 等 CRD
│   └── bkecommon/v1beta1/       # 共享类型定义
├── controllers/
│   └── capbke/
│       ├── bkecluster_controller.go    # 集群控制器
│       ├── bkemachine_controller.go    # 机器控制器
│       └── bkecluster_upgrade.go       # 升级控制器 (新增)
├── pkg/
│   ├── phaseframe/              # 阶段框架
│   ├── statusmanage/            # 状态管理
│   └── upgrader/                # 升级逻辑 (新增)
├── common/
│   ├── cluster/                 # 集群操作
│   └── node/                    # 节点操作
└── config/
    ├── crd/                     # CRD 定义
    └── samples/                 # 示例配置
```
### 三、CRD 设计与实现
#### 3.1 扩展 BKECluster 类型
```go
// api/bkecommon/v1beta1/bkecluster_types.go

package v1beta1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// BKEClusterSpec 定义集群期望状态
type BKEClusterSpec struct {
	// 控制面端点
	ControlPlaneEndpoint ControlPlaneEndpoint `json:"controlPlaneEndpoint"`

	// 集群配置
	ClusterConfig ClusterConfig `json:"clusterConfig"`

	// Kubernetes 版本
	KubernetesVersion string `json:"kubernetesVersion"`

	// 升级配置 (新增)
	UpgradeConfig *UpgradeConfig `json:"upgradeConfig,omitempty"`

	// 安装模式
	InstallMode InstallMode `json:"installMode"`

	// 节点配置
	Nodes []NodeConfig `json:"nodes,omitempty"`
}

// ControlPlaneEndpoint 控制面端点
type ControlPlaneEndpoint struct {
	Host string `json:"host"`
	Port int32  `json:"port"`
}

// ClusterConfig 集群配置
type ClusterConfig struct {
	// 集群网络配置
	Network NetworkConfig `json:"network"`

	// 容器运行时配置
	ContainerRuntime ContainerRuntimeConfig `json:"containerRuntime"`

	// Etcd 配置
	Etcd EtcdConfig `json:"etcd"`

	// Addons 配置
	Addons []AddonConfig `json:"addons,omitempty"`
}

// NetworkConfig 网络配置
type NetworkConfig struct {
	// Pod CIDR
	PodCIDR string `json:"podCIDR"`

	// Service CIDR
	ServiceCIDR string `json:"serviceCIDR"`

	// CNI 插件
	CNI CNIConfig `json:"cni"`
}

// CNIConfig CNI 配置
type CNIConfig struct {
	Type    string `json:"type"`
	Version string `json:"version"`
}

// ContainerRuntimeConfig 容器运行时配置
type ContainerRuntimeConfig struct {
	CRI     string `json:"cri"`
	Version string `json:"version"`
}

// EtcdConfig Etcd 配置
type EtcdConfig struct {
	Version    string `json:"version"`
	DataDir    string `json:"dataDir"`
	ExtraArgs  map[string]string `json:"extraArgs,omitempty"`
}

// AddonConfig Addon 配置
type AddonConfig struct {
	Name    string `json:"name"`
	Version string `json:"version"`
	Config  map[string]interface{} `json:"config,omitempty"`
}

// UpgradeConfig 升级配置 (新增)
type UpgradeConfig struct {
	// 目标版本
	DesiredVersion string `json:"desiredVersion"`

	// 升级策略
	Strategy UpgradeStrategy `json:"strategy"`

	// 升级渠道
	Channel string `json:"channel"`

	// 暂停升级
	Paused bool `json:"paused,omitempty"`
}

// UpgradeStrategy 升级策略
type UpgradeStrategy struct {
	// 升级类型: RollingUpdate, InPlace
	Type UpgradeType `json:"type"`

	// 滚动升级配置
	RollingUpdate *RollingUpdateConfig `json:"rollingUpdate,omitempty"`

	// 原地升级配置
	InPlace *InPlaceConfig `json:"inPlace,omitempty"`
}

// UpgradeType 升级类型
type UpgradeType string

const (
	UpgradeTypeRollingUpdate UpgradeType = "RollingUpdate"
	UpgradeTypeInPlace       UpgradeType = "InPlace"
)

// RollingUpdateConfig 滚动升级配置
type RollingUpdateConfig struct {
	// 最大不可用节点数
	MaxUnavailable int `json:"maxUnavailable"`

	// 最大额外节点数
	MaxSurge int `json:"maxSurge"`
}

// InPlaceConfig 原地升级配置
type InPlaceConfig struct {
	// 是否允许原地升级
	Enabled bool `json:"enabled"`

	// 升级前检查
	PreCheck bool `json:"preCheck"`
}

// InstallMode 安装模式
type InstallMode string

const (
	InstallModeUPI InstallMode = "UPI" // User-Provisioned Infrastructure
	InstallModeIPI InstallMode = "IPI" // Installer-Provisioned Infrastructure
)

// NodeConfig 节点配置
type NodeConfig struct {
	// 节点角色: control-plane, worker
	Role string `json:"role"`

	// 节点 IP
	IP string `json:"ip"`

	// 节点主机名
	Hostname string `json:"hostname,omitempty"`

	// 节点标签
	Labels map[string]string `json:"labels,omitempty"`

	// 节点污点
	Taints []Taint `json:"taints,omitempty"`
}

// Taint 污点
type Taint struct {
	Key    string `json:"key"`
	Value  string `json:"value"`
	Effect string `json:"effect"`
}

// BKEClusterStatus 定义集群当前状态
type BKEClusterStatus struct {
	// 阶段
	Phase ClusterPhase `json:"phase"`

	// 集群状态
	ClusterStatus ClusterStatus `json:"clusterStatus"`

	// 集群健康状态
	ClusterHealthState ClusterHealthState `json:"clusterHealthState"`

	// Kubernetes 版本
	KubernetesVersion string `json:"kubernetesVersion"`

	// Agent 状态
	AgentStatus AgentStatus `json:"agentStatus"`

	// 升级状态 (新增)
	UpgradeStatus *UpgradeStatus `json:"upgradeStatus,omitempty"`

	// 条件
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// 版本历史 (新增)
	VersionHistory []VersionHistory `json:"versionHistory,omitempty"`
}

// ClusterPhase 集群阶段
type ClusterPhase string

const (
	ClusterPhasePending      ClusterPhase = "Pending"
	ClusterPhaseProvisioning ClusterPhase = "Provisioning"
	ClusterPhaseProvisioned  ClusterPhase = "Provisioned"
	ClusterPhaseUpgrading    ClusterPhase = "Upgrading"
	ClusterPhaseDeleting     ClusterPhase = "Deleting"
	ClusterPhaseFailed       ClusterPhase = "Failed"
)

// ClusterStatus 集群状态
type ClusterStatus string

const (
	ClusterStatusCreating      ClusterStatus = "Creating"
	ClusterStatusRunning       ClusterStatus = "Running"
	ClusterStatusUpdating      ClusterStatus = "Updating"
	ClusterStatusUpgrading     ClusterStatus = "Upgrading"
	ClusterStatusScaling       ClusterStatus = "Scaling"
	ClusterStatusDeleting      ClusterStatus = "Deleting"
	ClusterStatusError         ClusterStatus = "Error"
	ClusterStatusUnknown       ClusterStatus = "Unknown"
)

// ClusterHealthState 集群健康状态
type ClusterHealthState string

const (
	ClusterHealthStateHealthy   ClusterHealthState = "Healthy"
	ClusterHealthStateWarning   ClusterHealthState = "Warning"
	ClusterHealthStateCritical  ClusterHealthState = "Critical"
	ClusterHealthStateUnknown   ClusterHealthState = "Unknown"
)

// AgentStatus Agent 状态
type AgentStatus struct {
	Status  string `json:"status"`
	Message string `json:"message,omitempty"`
}

// UpgradeStatus 升级状态 (新增)
type UpgradeStatus struct {
	// 当前版本
	CurrentVersion string `json:"currentVersion"`

	// 目标版本
	DesiredVersion string `json:"desiredVersion"`

	// 升级阶段
	Phase UpgradePhase `json:"phase"`

	// 升级进度 (0-100)
	Progress int `json:"progress"`

	// 升级计划
	UpgradePlan []UpgradeStep `json:"upgradePlan,omitempty"`

	// 开始时间
	StartedTime *metav1.Time `json:"startedTime,omitempty"`

	// 完成时间
	CompletionTime *metav1.Time `json:"completionTime,omitempty"`

	// 错误信息
	Error string `json:"error,omitempty"`
}

// UpgradePhase 升级阶段
type UpgradePhase string

const (
	UpgradePhasePending          UpgradePhase = "Pending"
	UpgradePhasePreCheck         UpgradePhase = "PreCheck"
	UpgradePhaseControlPlane     UpgradePhase = "ControlPlane"
	UpgradePhaseWorkers          UpgradePhase = "Workers"
	UpgradePhasePostCheck        UpgradePhase = "PostCheck"
	UpgradePhaseCompleted        UpgradePhase = "Completed"
	UpgradePhaseFailed           UpgradePhase = "Failed"
)

// UpgradeStep 升级步骤
type UpgradeStep struct {
	// 版本
	Version string `json:"version"`

	// 阶段
	Phase UpgradePhase `json:"phase"`

	// 状态
	Status StepStatus `json:"status"`

	// 开始时间
	StartedTime *metav1.Time `json:"startedTime,omitempty"`

	// 完成时间
	CompletionTime *metav1.Time `json:"completionTime,omitempty"`

	// 消息
	Message string `json:"message,omitempty"`
}

// StepStatus 步骤状态
type StepStatus string

const (
	StepStatusPending   StepStatus = "Pending"
	StepStatusRunning   StepStatus = "Running"
	StepStatusCompleted StepStatus = "Completed"
	StepStatusFailed    StepStatus = "Failed"
	StepStatusSkipped   StepStatus = "Skipped"
)

// VersionHistory 版本历史
type VersionHistory struct {
	// 版本
	Version string `json:"version"`

	// 开始时间
	StartedTime metav1.Time `json:"startedTime"`

	// 完成时间
	CompletionTime *metav1.Time `json:"completionTime,omitempty"`

	// 状态
	Status string `json:"status"`
}
```
#### 3.2 创建升级相关 CRD
```go
// api/capbke/v1beta1/clusterversion_types.go

package v1beta1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=cv
// +kubebuilder:printcolumn:name="CURRENT",type="string",JSONPath=".status.currentVersion"
// +kubebuilder:printcolumn:name="DESIRED",type="string",JSONPath=".spec.desiredVersion"
// +kubebuilder:printcolumn:name="PHASE",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="PROGRESS",type="string",JSONPath=".status.progress"

// ClusterVersion 集群版本管理
type ClusterVersion struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ClusterVersionSpec   `json:"spec,omitempty"`
	Status ClusterVersionStatus `json:"status,omitempty"`
}

// ClusterVersionSpec 集群版本期望状态
type ClusterVersionSpec struct {
	// 关联的集群
	ClusterRef ClusterReference `json:"clusterRef"`

	// 目标版本
	DesiredVersion string `json:"desiredVersion"`

	// 升级渠道
	Channel string `json:"channel,omitempty"`

	// 升级策略
	Strategy UpgradeStrategyConfig `json:"strategy,omitempty"`

	// 暂停升级
	Paused bool `json:"paused,omitempty"`
}

// ClusterReference 集群引用
type ClusterReference struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
}

// UpgradeStrategyConfig 升级策略配置
type UpgradeStrategyConfig struct {
	// 类型
	Type string `json:"type"`

	// ControlPlane 升级配置
	ControlPlane ControlPlaneUpgradeConfig `json:"controlPlane"`

	// Workers 升级配置
	Workers WorkersUpgradeConfig `json:"workers"`

	// 组件升级配置
	Components []ComponentUpgradeConfig `json:"components,omitempty"`
}

// ControlPlaneUpgradeConfig ControlPlane 升级配置
type ControlPlaneUpgradeConfig struct {
	// 最大不可用
	MaxUnavailable int `json:"maxUnavailable"`

	// 超时
	Timeout string `json:"timeout"`
}

// WorkersUpgradeConfig Workers 升级配置
type WorkersUpgradeConfig struct {
	// 并发数
	Concurrency int `json:"concurrency"`

	// 最大不可用
	MaxUnavailable int `json:"maxUnavailable"`

	// 超时
	Timeout string `json:"timeout"`
}

// ComponentUpgradeConfig 组件升级配置
type ComponentUpgradeConfig struct {
	// 组件名称
	Name string `json:"name"`

	// 目标版本
	Version string `json:"version"`

	// 升级时机: before, after, parallel
	When string `json:"when"`
}

// ClusterVersionStatus 集群版本当前状态
type ClusterVersionStatus struct {
	// 当前版本
	CurrentVersion string `json:"currentVersion"`

	// 阶段
	Phase string `json:"phase"`

	// 进度
	Progress int `json:"progress"`

	// ControlPlane 版本
	ControlPlaneVersion ComponentVersionStatus `json:"controlPlaneVersion"`

	// Worker 版本
	WorkerVersion WorkerVersionStatus `json:"workerVersion"`

	// 组件版本
	ComponentVersions map[string]ComponentVersionStatus `json:"componentVersions"`

	// 升级计划
	UpgradePlan []UpgradePlanStep `json:"upgradePlan,omitempty"`

	// 历史
	History []VersionHistory `json:"history,omitempty"`

	// 条件
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// ComponentVersionStatus 组件版本状态
type ComponentVersionStatus struct {
	Current  string `json:"current"`
	Desired  string `json:"desired"`
	Upgrading bool   `json:"upgrading"`
	Healthy  bool   `json:"healthy"`
}

// WorkerVersionStatus Worker 版本状态
type WorkerVersionStatus struct {
	Current            string                   `json:"current"`
	Desired            string                   `json:"desired"`
	Upgrading          bool                     `json:"upgrading"`
	VersionDistribution []VersionDistribution   `json:"versionDistribution"`
	VersionSkew        int                      `json:"versionSkew"`
	IsWithinPolicy     bool                     `json:"isWithinPolicy"`
}

// VersionDistribution 版本分布
type VersionDistribution struct {
	Version string `json:"version"`
	Count   int    `json:"count"`
	Ready   int    `json:"ready"`
}

// UpgradePlanStep 升级计划步骤
type UpgradePlanStep struct {
	Version         string       `json:"version"`
	Phase           string       `json:"phase"`
	Status          string       `json:"status"`
	StartedTime     *metav1.Time `json:"startedTime,omitempty"`
	CompletionTime  *metav1.Time `json:"completionTime,omitempty"`
	Message         string       `json:"message,omitempty"`
}

// +kubebuilder:object:root=true

// ClusterVersionList 集群版本列表
type ClusterVersionList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ClusterVersion `json:"items"`
}

func init() {
	SchemeBuilder.Register(&ClusterVersion{}, &ClusterVersionList{})
}
```
### 四、控制器实现
#### 4.1 集群安装控制器
```go
// controllers/capbke/bkecluster_install.go

package capbke

import (
	"context"
	"fmt"
	"time"

	"github.com/pkg/errors"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	bkev1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/capbke/v1beta1"
	confv1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/bkecommon/v1beta1"
	"gopkg.openfuyao.cn/cluster-api-provider-bke/pkg/phaseframe"
	"gopkg.openfuyao.cn/cluster-api-provider-bke/utils/capbke/condition"
)

// InstallReconciler 处理集群安装
type InstallReconciler struct {
	client.Client
}

// ReconcileInstall 处理集群安装
func (r *InstallReconciler) ReconcileInstall(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)

	// 创建阶段框架
	phaseFrame := phaseframe.NewPhaseFrame("cluster-install")

	// 阶段 1: 预检查
	if err := r.preCheck(ctx, bkeCluster, phaseFrame); err != nil {
		return ctrl.Result{}, err
	}

	// 阶段 2: 初始化控制面
	if err := r.initControlPlane(ctx, bkeCluster, phaseFrame); err != nil {
		return ctrl.Result{}, err
	}

	// 阶段 3: 加入 Worker 节点
	if err := r.joinWorkers(ctx, bkeCluster, phaseFrame); err != nil {
		return ctrl.Result{}, err
	}

	// 阶段 4: 安装 Addons
	if err := r.installAddons(ctx, bkeCluster, phaseFrame); err != nil {
		return ctrl.Result{}, err
	}

	// 阶段 5: 后置检查
	if err := r.postCheck(ctx, bkeCluster, phaseFrame); err != nil {
		return ctrl.Result{}, err
	}

	// 更新状态
	bkeCluster.Status.Phase = confv1beta1.ClusterPhaseProvisioned
	bkeCluster.Status.ClusterStatus = confv1beta1.ClusterStatusRunning
	condition.SetClusterProvisioned(bkeCluster)

	log.Info("Cluster installation completed")
	return ctrl.Result{}, nil
}

// preCheck 预检查
func (r *InstallReconciler) preCheck(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, pf *phaseframe.PhaseFrame) error {
	log := ctrl.LoggerFrom(ctx)
	phase := pf.NewPhase("pre-check")

	log.Info("Running pre-check")

	// 1. 检查节点连通性
	for _, node := range bkeCluster.Spec.Nodes {
		if err := r.checkNodeConnectivity(ctx, node.IP); err != nil {
			phase.Fail(fmt.Sprintf("Node %s connectivity check failed: %v", node.IP, err))
			return errors.Wrapf(err, "node %s connectivity check failed", node.IP)
		}
	}

	// 2. 检查端口可用性
	if err := r.checkPortsAvailable(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Port check failed: %v", err))
		return errors.Wrap(err, "port check failed")
	}

	// 3. 检查系统要求
	if err := r.checkSystemRequirements(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("System requirements check failed: %v", err))
		return errors.Wrap(err, "system requirements check failed")
	}

	phase.Complete()
	return nil
}

// initControlPlane 初始化控制面
func (r *InstallReconciler) initControlPlane(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, pf *phaseframe.PhaseFrame) error {
	log := ctrl.LoggerFrom(ctx)
	phase := pf.NewPhase("init-control-plane")

	log.Info("Initializing control plane")

	// 获取控制面节点
	controlPlaneNodes := r.getControlPlaneNodes(bkeCluster)
	if len(controlPlaneNodes) == 0 {
		err := errors.New("no control plane nodes found")
		phase.Fail(err.Error())
		return err
	}

	// 1. 初始化第一个控制面节点
	firstNode := controlPlaneNodes[0]
	log.Info("Initializing first control plane node", "node", firstNode.IP)

	if err := r.initFirstControlPlaneNode(ctx, bkeCluster, firstNode); err != nil {
		phase.Fail(fmt.Sprintf("Failed to initialize first control plane node: %v", err))
		return errors.Wrap(err, "failed to initialize first control plane node")
	}

	// 2. 加入其他控制面节点
	for i := 1; i < len(controlPlaneNodes); i++ {
		node := controlPlaneNodes[i]
		log.Info("Joining control plane node", "node", node.IP)

		if err := r.joinControlPlaneNode(ctx, bkeCluster, node); err != nil {
			phase.Fail(fmt.Sprintf("Failed to join control plane node %s: %v", node.IP, err))
			return errors.Wrapf(err, "failed to join control plane node %s", node.IP)
		}
	}

	// 3. 等待控制面就绪
	if err := r.waitForControlPlaneReady(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Control plane not ready: %v", err))
		return errors.Wrap(err, "control plane not ready")
	}

	phase.Complete()
	return nil
}

// joinWorkers 加入 Worker 节点
func (r *InstallReconciler) joinWorkers(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, pf *phaseframe.PhaseFrame) error {
	log := ctrl.LoggerFrom(ctx)
	phase := pf.NewPhase("join-workers")

	log.Info("Joining worker nodes")

	// 获取 Worker 节点
	workerNodes := r.getWorkerNodes(bkeCluster)
	if len(workerNodes) == 0 {
		log.Info("No worker nodes to join")
		phase.Complete()
		return nil
	}

	// 并发加入 Worker 节点
	for _, node := range workerNodes {
		log.Info("Joining worker node", "node", node.IP)

		if err := r.joinWorkerNode(ctx, bkeCluster, node); err != nil {
			phase.Fail(fmt.Sprintf("Failed to join worker node %s: %v", node.IP, err))
			return errors.Wrapf(err, "failed to join worker node %s", node.IP)
		}
	}

	// 等待所有 Worker 节点就绪
	if err := r.waitForWorkersReady(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Workers not ready: %v", err))
		return errors.Wrap(err, "workers not ready")
	}

	phase.Complete()
	return nil
}

// installAddons 安装 Addons
func (r *InstallReconciler) installAddons(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, pf *phaseframe.PhaseFrame) error {
	log := ctrl.LoggerFrom(ctx)
	phase := pf.NewPhase("install-addons")

	log.Info("Installing addons")

	for _, addon := range bkeCluster.Spec.ClusterConfig.Addons {
		log.Info("Installing addon", "addon", addon.Name)

		if err := r.installAddon(ctx, bkeCluster, addon); err != nil {
			phase.Fail(fmt.Sprintf("Failed to install addon %s: %v", addon.Name, err))
			return errors.Wrapf(err, "failed to install addon %s", addon.Name)
		}
	}

	phase.Complete()
	return nil
}

// postCheck 后置检查
func (r *InstallReconciler) postCheck(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, pf *phaseframe.PhaseFrame) error {
	log := ctrl.LoggerFrom(ctx)
	phase := pf.NewPhase("post-check")

	log.Info("Running post-check")

	// 1. 检查所有节点状态
	if err := r.checkAllNodesReady(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Nodes not ready: %v", err))
		return errors.Wrap(err, "nodes not ready")
	}

	// 2. 检查组件健康
	if err := r.checkComponentsHealth(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Components not healthy: %v", err))
		return errors.Wrap(err, "components not healthy")
	}

	// 3. 检查网络连通性
	if err := r.checkNetworkConnectivity(ctx, bkeCluster); err != nil {
		phase.Fail(fmt.Sprintf("Network connectivity check failed: %v", err))
		return errors.Wrap(err, "network connectivity check failed")
	}

	phase.Complete()
	return nil
}

// Helper methods

func (r *InstallReconciler) getControlPlaneNodes(bkeCluster *bkev1beta1.BKECluster) []confv1beta1.NodeConfig {
	var nodes []confv1beta1.NodeConfig
	for _, node := range bkeCluster.Spec.Nodes {
		if node.Role == "control-plane" {
			nodes = append(nodes, node)
		}
	}
	return nodes
}

func (r *InstallReconciler) getWorkerNodes(bkeCluster *bkev1beta1.BKECluster) []confv1beta1.NodeConfig {
	var nodes []confv1beta1.NodeConfig
	for _, node := range bkeCluster.Spec.Nodes {
		if node.Role == "worker" {
			nodes = append(nodes, node)
		}
	}
	return nodes
}

func (r *InstallReconciler) checkNodeConnectivity(ctx context.Context, ip string) error {
	// 实现 SSH 连通性检查
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) checkPortsAvailable(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现端口可用性检查
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) checkSystemRequirements(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现系统要求检查
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) initFirstControlPlaneNode(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig) error {
	// 实现第一个控制面节点初始化
	// 1. 生成 kubeadm 配置
	// 2. 执行 kubeadm init
	// 3. 安装 CNI
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) joinControlPlaneNode(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig) error {
	// 实现控制面节点加入
	// 1. 获取 join token
	// 2. 执行 kubeadm join --control-plane
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) joinWorkerNode(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig) error {
	// 实现 Worker 节点加入
	// 1. 获取 join token
	// 2. 执行 kubeadm join
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) waitForControlPlaneReady(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 等待控制面就绪
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) waitForWorkersReady(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 等待 Worker 节点就绪
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) installAddon(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, addon confv1beta1.AddonConfig) error {
	// 安装 Addon
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) checkAllNodesReady(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 检查所有节点就绪
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) checkComponentsHealth(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 检查组件健康
	// TODO: 实现具体逻辑
	return nil
}

func (r *InstallReconciler) checkNetworkConnectivity(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 检查网络连通性
	// TODO: 实现具体逻辑
	return nil
}
```
#### 4.2 集群升级控制器
```go
// controllers/capbke/bkecluster_upgrade.go

package capbke

import (
	"context"
	"fmt"
	"time"

	"github.com/pkg/errors"
	"github.com/blang/semver/v4"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	bkev1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/capbke/v1beta1"
	confv1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/bkecommon/v1beta1"
	"gopkg.openfuyao.cn/cluster-api-provider-bke/pkg/phaseframe"
	"gopkg.openfuyao.cn/cluster-api-provider-bke/utils/capbke/condition"
)

// UpgradeReconciler 处理集群升级
type UpgradeReconciler struct {
	client.Client
}

// ReconcileUpgrade 处理集群升级
func (r *UpgradeReconciler) ReconcileUpgrade(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)

	// 检查是否需要升级
	if !r.needsUpgrade(bkeCluster) {
		return ctrl.Result{}, nil
	}

	// 检查是否暂停升级
	if bkeCluster.Spec.UpgradeConfig != nil && bkeCluster.Spec.UpgradeConfig.Paused {
		log.Info("Upgrade is paused")
		return ctrl.Result{}, nil
	}

	// 初始化升级状态
	if bkeCluster.Status.UpgradeStatus == nil {
		r.initUpgradeStatus(bkeCluster)
	}

	// 根据升级阶段执行相应逻辑
	switch bkeCluster.Status.UpgradeStatus.Phase {
	case confv1beta1.UpgradePhasePending:
		return r.handlePendingPhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhasePreCheck:
		return r.handlePreCheckPhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhaseControlPlane:
		return r.handleControlPlanePhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhaseWorkers:
		return r.handleWorkersPhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhasePostCheck:
		return r.handlePostCheckPhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhaseCompleted:
		return r.handleCompletedPhase(ctx, bkeCluster)
	case confv1beta1.UpgradePhaseFailed:
		return r.handleFailedPhase(ctx, bkeCluster)
	default:
		return ctrl.Result{}, nil
	}
}

// needsUpgrade 检查是否需要升级
func (r *UpgradeReconciler) needsUpgrade(bkeCluster *bkev1beta1.BKECluster) bool {
	if bkeCluster.Spec.UpgradeConfig == nil {
		return false
	}

	if bkeCluster.Spec.UpgradeConfig.DesiredVersion == "" {
		return false
	}

	if bkeCluster.Status.KubernetesVersion == bkeCluster.Spec.UpgradeConfig.DesiredVersion {
		return false
	}

	return true
}

// initUpgradeStatus 初始化升级状态
func (r *UpgradeReconciler) initUpgradeStatus(bkeCluster *bkev1beta1.BKECluster) {
	bkeCluster.Status.UpgradeStatus = &confv1beta1.UpgradeStatus{
		CurrentVersion: bkeCluster.Status.KubernetesVersion,
		DesiredVersion: bkeCluster.Spec.UpgradeConfig.DesiredVersion,
		Phase:          confv1beta1.UpgradePhasePending,
		Progress:       0,
		StartedTime:    &metav1.Time{Time: time.Now()},
	}

	// 生成升级计划
	bkeCluster.Status.UpgradeStatus.UpgradePlan = r.generateUpgradePlan(
		bkeCluster.Status.KubernetesVersion,
		bkeCluster.Spec.UpgradeConfig.DesiredVersion,
	)
}

// generateUpgradePlan 生成升级计划
func (r *UpgradeReconciler) generateUpgradePlan(currentVersion, desiredVersion string) []confv1beta1.UpgradeStep {
	plan := []confv1beta1.UpgradeStep{}

	current, err := semver.ParseTolerant(currentVersion)
	if err != nil {
		return plan
	}

	desired, err := semver.ParseTolerant(desiredVersion)
	if err != nil {
		return plan
	}

	// 生成中间版本步骤
	if current.Major == desired.Major {
		for minor := current.Minor + 1; minor <= desired.Minor; minor++ {
			version := fmt.Sprintf("v%d.%d.%d", current.Major, minor, 0)
			plan = append(plan, confv1beta1.UpgradeStep{
				Version: version,
				Phase:   confv1beta1.UpgradePhasePending,
				Status:  confv1beta1.StepStatusPending,
			})
		}
	}

	// 添加最终版本
	if len(plan) == 0 || plan[len(plan)-1].Version != desiredVersion {
		plan = append(plan, confv1beta1.UpgradeStep{
			Version: desiredVersion,
			Phase:   confv1beta1.UpgradePhasePending,
			Status:  confv1beta1.StepStatusPending,
		})
	}

	return plan
}

// handlePendingPhase 处理 Pending 阶段
func (r *UpgradeReconciler) handlePendingPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Handling pending phase")

	// 更新阶段
	bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhasePreCheck
	bkeCluster.Status.UpgradeStatus.Progress = 5

	return ctrl.Result{Requeue: true}, nil
}

// handlePreCheckPhase 处理 PreCheck 阶段
func (r *UpgradeReconciler) handlePreCheckPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Handling pre-check phase")

	// 1. 检查集群健康状态
	if bkeCluster.Status.ClusterHealthState != confv1beta1.ClusterHealthStateHealthy {
		err := errors.New("cluster is not healthy")
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 2. 检查版本兼容性
	if err := r.checkVersionCompatibility(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 3. 检查资源充足性
	if err := r.checkResourceAvailability(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 4. 备份关键数据
	if err := r.backupClusterData(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 更新阶段
	bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseControlPlane
	bkeCluster.Status.UpgradeStatus.Progress = 10

	return ctrl.Result{Requeue: true}, nil
}

// handleControlPlanePhase 处理 ControlPlane 升级阶段
func (r *UpgradeReconciler) handleControlPlanePhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Handling control plane upgrade phase")

	// 获取当前升级步骤
	currentStep := r.getCurrentUpgradeStep(bkeCluster)
	if currentStep == nil {
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseWorkers
		return ctrl.Result{Requeue: true}, nil
	}

	log.Info("Upgrading control plane to version", "version", currentStep.Version)

	// 获取控制面节点
	controlPlaneNodes := r.getControlPlaneNodes(bkeCluster)

	// 逐个升级控制面节点
	for _, node := range controlPlaneNodes {
		// 检查节点是否已升级
		if r.isNodeUpgraded(ctx, bkeCluster, node, currentStep.Version) {
			continue
		}

		// 升级节点
		if err := r.upgradeControlPlaneNode(ctx, bkeCluster, node, currentStep.Version); err != nil {
			bkeCluster.Status.UpgradeStatus.Error = fmt.Sprintf("Failed to upgrade control plane node %s: %v", node.IP, err)
			bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
			return ctrl.Result{}, err
		}

		// 等待节点就绪
		if err := r.waitForNodeReady(ctx, bkeCluster, node); err != nil {
			bkeCluster.Status.UpgradeStatus.Error = fmt.Sprintf("Control plane node %s not ready: %v", node.IP, err)
			bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
			return ctrl.Result{}, err
		}

		// 更新进度
		bkeCluster.Status.UpgradeStatus.Progress = 10 + (40 * len(controlPlaneNodes) / len(controlPlaneNodes))
	}

	// 标记当前步骤完成
	currentStep.Status = confv1beta1.StepStatusCompleted
	currentStep.CompletionTime = &metav1.Time{Time: time.Now()}

	// 检查是否还有下一个步骤
	if r.hasNextUpgradeStep(bkeCluster) {
		r.moveToNextUpgradeStep(bkeCluster)
		return ctrl.Result{Requeue: true}, nil
	}

	// 所有步骤完成，进入 Workers 升级阶段
	bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseWorkers
	bkeCluster.Status.UpgradeStatus.Progress = 50

	return ctrl.Result{Requeue: true}, nil
}

// handleWorkersPhase 处理 Workers 升级阶段
func (r *UpgradeReconciler) handleWorkersPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Handling workers upgrade phase")

	// 获取 Worker 节点
	workerNodes := r.getWorkerNodes(bkeCluster)
	if len(workerNodes) == 0 {
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhasePostCheck
		return ctrl.Result{Requeue: true}, nil
	}

	// 获取目标版本
	targetVersion := bkeCluster.Spec.UpgradeConfig.DesiredVersion

	// 获取并发配置
	concurrency := 1
	if bkeCluster.Spec.UpgradeConfig != nil && bkeCluster.Spec.UpgradeConfig.Strategy.RollingUpdate != nil {
		concurrency = bkeCluster.Spec.UpgradeConfig.Strategy.RollingUpdate.MaxSurge
	}

	// 批量升级 Worker 节点
	upgraded := 0
	for _, node := range workerNodes {
		// 检查节点是否已升级
		if r.isNodeUpgraded(ctx, bkeCluster, node, targetVersion) {
			upgraded++
			continue
		}

		// 检查并发限制
		if upgraded >= concurrency {
			break
		}

		// 升级节点
		if err := r.upgradeWorkerNode(ctx, bkeCluster, node, targetVersion); err != nil {
			log.Error(err, "Failed to upgrade worker node", "node", node.IP)
			continue
		}

		upgraded++
	}

	// 更新进度
	progress := 50 + (40 * upgraded / len(workerNodes))
	bkeCluster.Status.UpgradeStatus.Progress = progress

	// 检查是否所有节点都已升级
	if upgraded == len(workerNodes) {
		// 等待所有节点就绪
		if err := r.waitForAllWorkersReady(ctx, bkeCluster); err != nil {
			bkeCluster.Status.UpgradeStatus.Error = err.Error()
			bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
			return ctrl.Result{}, err
		}

		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhasePostCheck
		return ctrl.Result{Requeue: true}, nil
	}

	// 继续升级剩余节点
	return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}

// handlePostCheckPhase 处理 PostCheck 阶段
func (r *UpgradeReconciler) handlePostCheckPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Handling post-check phase")

	// 1. 检查所有节点状态
	if err := r.checkAllNodesReady(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 2. 检查组件健康
	if err := r.checkComponentsHealth(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 3. 检查工作负载正常
	if err := r.checkWorkloadsHealthy(ctx, bkeCluster); err != nil {
		bkeCluster.Status.UpgradeStatus.Error = err.Error()
		bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseFailed
		return ctrl.Result{}, err
	}

	// 更新阶段
	bkeCluster.Status.UpgradeStatus.Phase = confv1beta1.UpgradePhaseCompleted
	bkeCluster.Status.UpgradeStatus.Progress = 100
	bkeCluster.Status.UpgradeStatus.CompletionTime = &metav1.Time{Time: time.Now()}

	return ctrl.Result{Requeue: true}, nil
}

// handleCompletedPhase 处理 Completed 阶段
func (r *UpgradeReconciler) handleCompletedPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Info("Upgrade completed")

	// 更新集群版本
	bkeCluster.Status.KubernetesVersion = bkeCluster.Spec.UpgradeConfig.DesiredVersion
	bkeCluster.Status.Phase = confv1beta1.ClusterPhaseProvisioned
	bkeCluster.Status.ClusterStatus = confv1beta1.ClusterStatusRunning

	// 添加版本历史
	bkeCluster.Status.VersionHistory = append(bkeCluster.Status.VersionHistory, confv1beta1.VersionHistory{
		Version:        bkeCluster.Spec.UpgradeConfig.DesiredVersion,
		StartedTime:    *bkeCluster.Status.UpgradeStatus.StartedTime,
		CompletionTime: bkeCluster.Status.UpgradeStatus.CompletionTime,
		Status:         "Completed",
	})

	// 清除升级配置
	bkeCluster.Spec.UpgradeConfig = nil
	bkeCluster.Status.UpgradeStatus = nil

	// 设置条件
	condition.SetClusterUpgraded(bkeCluster)

	return ctrl.Result{}, nil
}

// handleFailedPhase 处理 Failed 阶段
func (r *UpgradeReconciler) handleFailedPhase(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)
	log.Error(errors.New(bkeCluster.Status.UpgradeStatus.Error), "Upgrade failed")

	// 设置条件
	condition.SetClusterUpgradeFailed(bkeCluster, bkeCluster.Status.UpgradeStatus.Error)

	// 可以选择自动重试或等待人工干预
	return ctrl.Result{}, nil
}

// Helper methods

func (r *UpgradeReconciler) getCurrentUpgradeStep(bkeCluster *bkev1beta1.BKECluster) *confv1beta1.UpgradeStep {
	for i := range bkeCluster.Status.UpgradeStatus.UpgradePlan {
		step := &bkeCluster.Status.UpgradeStatus.UpgradePlan[i]
		if step.Status == confv1beta1.StepStatusPending || step.Status == confv1beta1.StepStatusRunning {
			return step
		}
	}
	return nil
}

func (r *UpgradeReconciler) hasNextUpgradeStep(bkeCluster *bkev1beta1.BKECluster) bool {
	for _, step := range bkeCluster.Status.UpgradeStatus.UpgradePlan {
		if step.Status == confv1beta1.StepStatusPending {
			return true
		}
	}
	return false
}

func (r *UpgradeReconciler) moveToNextUpgradeStep(bkeCluster *bkev1beta1.BKECluster) {
	for i := range bkeCluster.Status.UpgradeStatus.UpgradePlan {
		step := &bkeCluster.Status.UpgradeStatus.UpgradePlan[i]
		if step.Status == confv1beta1.StepStatusPending {
			step.Status = confv1beta1.StepStatusRunning
			step.StartedTime = &metav1.Time{Time: time.Now()}
			return
		}
	}
}

func (r *UpgradeReconciler) checkVersionCompatibility(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现版本兼容性检查
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) checkResourceAvailability(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现资源可用性检查
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) backupClusterData(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现集群数据备份
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) isNodeUpgraded(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig, targetVersion string) bool {
	// 实现节点版本检查
	// TODO: 实现具体逻辑
	return false
}

func (r *UpgradeReconciler) upgradeControlPlaneNode(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig, version string) error {
	// 实现控制面节点升级
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) upgradeWorkerNode(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig, version string) error {
	// 实现 Worker 节点升级
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) waitForNodeReady(ctx context.Context, bkeCluster *bkev1beta1.BKECluster, node confv1beta1.NodeConfig) error {
	// 实现等待节点就绪
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) waitForAllWorkersReady(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现等待所有 Worker 就绪
	// TODO: 实现具体逻辑
	return nil
}

func (r *UpgradeReconciler) checkWorkloadsHealthy(ctx context.Context, bkeCluster *bkev1beta1.BKECluster) error {
	// 实现工作负载健康检查
	// TODO: 实现具体逻辑
	return nil
}
```
### 五、测试与验证
#### 5.1 单元测试
```go
// controllers/capbke/bkecluster_upgrade_test.go

package capbke

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client/fake"

	bkev1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/capbke/v1beta1"
	confv1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/bkecommon/v1beta1"
)

func TestGenerateUpgradePlan(t *testing.T) {
	scheme := runtime.NewScheme()
	_ = bkev1beta1.AddToScheme(scheme)

	tests := []struct {
		name            string
		currentVersion  string
		desiredVersion  string
		expectedSteps   int
	}{
		{
			name:           "patch version upgrade",
			currentVersion: "v1.28.0",
			desiredVersion: "v1.28.5",
			expectedSteps:  1,
		},
		{
			name:           "minor version upgrade",
			currentVersion: "v1.28.0",
			desiredVersion: "v1.29.0",
			expectedSteps:  1,
		},
		{
			name:           "multi minor version upgrade",
			currentVersion: "v1.28.0",
			desiredVersion: "v1.30.0",
			expectedSteps:  2,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			r := &UpgradeReconciler{}
			plan := r.generateUpgradePlan(tt.currentVersion, tt.desiredVersion)
			assert.Equal(t, tt.expectedSteps, len(plan))
		})
	}
}

func TestNeedsUpgrade(t *testing.T) {
	tests := []struct {
		name       string
		cluster    *bkev1beta1.BKECluster
		needsUpgrade bool
	}{
		{
			name: "no upgrade config",
			cluster: &bkev1beta1.BKECluster{
				Spec: bkev1beta1.BKEClusterSpec{},
				Status: bkev1beta1.BKEClusterStatus{
					KubernetesVersion: "v1.28.0",
				},
			},
			needsUpgrade: false,
		},
		{
			name: "same version",
			cluster: &bkev1beta1.BKECluster{
				Spec: bkev1beta1.BKEClusterSpec{
					UpgradeConfig: &confv1beta1.UpgradeConfig{
						DesiredVersion: "v1.28.0",
					},
				},
				Status: bkev1beta1.BKEClusterStatus{
					KubernetesVersion: "v1.28.0",
				},
			},
			needsUpgrade: false,
		},
		{
			name: "needs upgrade",
			cluster: &bkev1beta1.BKECluster{
				Spec: bkev1beta1.BKEClusterSpec{
					UpgradeConfig: &confv1beta1.UpgradeConfig{
						DesiredVersion: "v1.29.0",
					},
				},
				Status: bkev1beta1.BKEClusterStatus{
					KubernetesVersion: "v1.28.0",
				},
			},
			needsUpgrade: true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			r := &UpgradeReconciler{}
			result := r.needsUpgrade(tt.cluster)
			assert.Equal(t, tt.needsUpgrade, result)
		})
	}
}
```
#### 5.2 集成测试
```go
// test/integration/upgrade_test.go

package integration

import (
	"context"
	"fmt"
	"testing"
	"time"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/envtest"

	bkev1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/capbke/v1beta1"
	confv1beta1 "gopkg.openfuyao.cn/cluster-api-provider-bke/api/bkecommon/v1beta1"
)

var (
	testEnv *envtest.Environment
	k8sClient client.Client
)

func TestUpgrade(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Upgrade Suite")
}

var _ = BeforeSuite(func() {
	// Setup test environment
	testEnv = &envtest.Environment{}
	cfg, err := testEnv.Start()
	Expect(err).NotTo(HaveOccurred())

	scheme := runtime.NewScheme()
	_ = bkev1beta1.AddToScheme(scheme)
	_ = clusterv1.AddToScheme(scheme)

	k8sClient, err = client.New(cfg, client.Options{Scheme: scheme})
	Expect(err).NotTo(HaveOccurred())
})

var _ = AfterSuite(func() {
	Expect(testEnv.Stop()).To(Succeed())
})

var _ = Describe("Cluster Upgrade", func() {
	var (
		ctx      context.Context
		cluster  *bkev1beta1.BKECluster
		namespace string
	)

	BeforeEach(func() {
		ctx = context.Background()
		namespace = fmt.Sprintf("test-%d", time.Now().UnixNano())

		// Create namespace
		ns := &corev1.Namespace{
			ObjectMeta: metav1.ObjectMeta{
				Name: namespace,
			},
		}
		Expect(k8sClient.Create(ctx, ns)).To(Succeed())

		// Create cluster
		cluster = &bkev1beta1.BKECluster{
			ObjectMeta: metav1.ObjectMeta{
				Name:      "test-cluster",
				Namespace: namespace,
			},
			Spec: bkev1beta1.BKEClusterSpec{
				KubernetesVersion: "v1.28.0",
				Nodes: []confv1beta1.NodeConfig{
					{
						Role: "control-plane",
						IP:   "192.168.1.10",
					},
					{
						Role: "worker",
						IP:   "192.168.1.11",
					},
				},
			},
		}
		Expect(k8sClient.Create(ctx, cluster)).To(Succeed())
	})

	AfterEach(func() {
		Expect(k8sClient.Delete(ctx, cluster)).To(Succeed())
	})

	It("should trigger upgrade when desired version changes", func() {
		// Update desired version
		cluster.Spec.UpgradeConfig = &confv1beta1.UpgradeConfig{
			DesiredVersion: "v1.29.0",
		}
		Expect(k8sClient.Update(ctx, cluster)).To(Succeed())

		// Wait for upgrade to start
		Eventually(func() bool {
			Expect(k8sClient.Get(ctx, client.ObjectKeyFromObject(cluster), cluster)).To(Succeed())
			return cluster.Status.UpgradeStatus != nil &&
				cluster.Status.UpgradeStatus.Phase == confv1beta1.UpgradePhasePreCheck
		}, 10*time.Second, 1*time.Second).Should(BeTrue())
	})
})
```
### 六、部署与使用
#### 6.1 部署控制器
```bash
# 构建 Docker 镜像
make docker-build IMG=openfuyao/cluster-api-provider-bke:latest

# 推送镜像
make docker-push IMG=openfuyao/cluster-api-provider-bke:latest

# 部署到管理集群
make deploy IMG=openfuyao/cluster-api-provider-bke:latest
```
#### 6.2 创建集群
```yaml
# config/samples/cluster.yaml
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: openfuyao-cluster
  namespace: default
spec:
  kubernetesVersion: v1.28.0
  installMode: UPI
  controlPlaneEndpoint:
    host: 192.168.1.10
    port: 6443
  clusterConfig:
    network:
      podCIDR: "10.244.0.0/16"
      serviceCIDR: "10.96.0.0/12"
      cni:
        type: calico
        version: v3.26.0
    containerRuntime:
      cri: containerd
      version: "1.7.2"
    etcd:
      version: "3.5.9"
      dataDir: "/var/lib/etcd"
  nodes:
  - role: control-plane
    ip: 192.168.1.10
    hostname: master-1
  - role: control-plane
    ip: 192.168.1.11
    hostname: master-2
  - role: control-plane
    ip: 192.168.1.12
    hostname: master-3
  - role: worker
    ip: 192.168.1.20
    hostname: worker-1
  - role: worker
    ip: 192.168.1.21
    hostname: worker-2
```
#### 6.3 触发升级
```yaml
# 升级到 v1.29.0
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: openfuyao-cluster
  namespace: default
spec:
  kubernetesVersion: v1.28.0
  upgradeConfig:
    desiredVersion: v1.29.0
    channel: stable
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1
        maxSurge: 1
```
```bash
# 应用升级配置
kubectl apply -f cluster-upgrade.yaml

# 查看升级状态
kubectl get bkecluster openfuyao-cluster -o yaml

# 查看升级进度
kubectl get bkecluster openfuyao-cluster -o jsonpath='{.status.upgradeStatus.progress}'
```
### 七、最佳实践
1. **升级前备份**: 始终在升级前备份 etcd 和关键配置
2. **逐版本升级**: 不要跳过中间版本，遵循 Kubernetes 版本升级路径
3. **监控升级进度**: 使用 `kubectl get` 监控升级状态和进度
4. **测试升级流程**: 在生产环境前，在测试环境验证升级流程
5. **准备回滚方案**: 虽然不支持自动回滚，但要准备手动回滚方案

这个开发指南提供了完整的 UPI 模式 openFuyao 集群安装升级功能的开发框架，包括 CRD 设计、控制器实现、测试和部署。开发者可以基于此框架进行具体实现和扩展。
        
