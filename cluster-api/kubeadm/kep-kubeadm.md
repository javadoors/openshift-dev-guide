# kubeadm + Cluster-API 控制面组件分离完整方案

## 一、背景与目标

### 1.1 当前 kubeadm 架构限制
kubeadm 原生**不支持**控制面组件分离部署。所有控制面组件（API Server、Scheduler、Controller Manager）必须部署在同一节点上，这是由 kubeadm 的架构决定的：
- 所有组件通过 static pod manifest 部署在 `/etc/kubernetes/manifests/`
- `kubeadm init` 和 `kubeadm join --control-plane` 会部署全部三个组件
- 没有配置项可以禁止某个节点运行特定组件

### 1.2 方案设计目标
| 目标 | 说明 |
|------|------|
| **向后兼容** | 现有 kubeadm 配置和行为不受影响 |
| **最小侵入** | 仅修改必要的代码路径 |
| **可演进** | 分阶段实现，每阶段可独立使用 |
| **社区可接受** | 遵循 kubeadm 设计哲学，不引入外部依赖 |
| **CAPI 集成** | 与 Cluster-API 生态无缝集成 |

## 二、kubeadm 侧实现方案

### 2.1 API 类型扩展
**文件**: `cmd/kubeadm/app/apis/kubeadm/types.go`
```go
// ControlPlaneComponentMode 定义控制面组件部署模式
type ControlPlaneComponentMode string

const (
    // 全组件模式 (默认，向后兼容)
    ControlPlaneModeAll ControlPlaneComponentMode = "all"
    
    // 仅部署 API Server
    ControlPlaneModeAPIServerOnly ControlPlaneComponentMode = "apiserver-only"
    
    // 仅部署 Scheduler
    ControlPlaneModeSchedulerOnly ControlPlaneComponentMode = "scheduler-only"
    
    // 仅部署 Controller Manager
    ControlPlaneModeControllerManagerOnly ControlPlaneComponentMode = "controller-manager-only"
    
    // 仅部署 etcd
    ControlPlaneModeEtcdOnly ControlPlaneComponentMode = "etcd-only"
    
    // 部署 API Server + etcd
    ControlPlaneModeAPIServerEtcd ControlPlaneComponentMode = "apiserver-etcd"
)

// ControlPlane 新增配置结构
type ControlPlane struct {
    // 组件部署模式
    Mode ControlPlaneComponentMode `json:"mode,omitempty"`
    
    // 外部组件端点配置
    ExternalComponents ExternalControlPlaneComponents `json:"externalComponents,omitempty"`
    
    // etcd 集群配置
    EtcdCluster EtcdClusterConfig `json:"etcdCluster,omitempty"`
}

// ExternalControlPlaneComponents 定义外部组件连接信息
type ExternalControlPlaneComponents struct {
    Scheduler     ExternalComponent `json:"scheduler,omitempty"`
    ControllerManager ExternalComponent `json:"controllerManager,omitempty"`
}

// ExternalComponent 定义单个外部组件连接信息
type ExternalComponent struct {
    Endpoints      []string `json:"endpoints,omitempty"`
    KubeConfigPath string   `json:"kubeConfigPath,omitempty"`
}

// EtcdClusterConfig 定义 etcd 集群配置
type EtcdClusterConfig struct {
    Managed         bool     `json:"managed,omitempty"`
    Endpoints       []string `json:"endpoints,omitempty"`
    CertsDir        string   `json:"certsDir,omitempty"`
    DataDir         string   `json:"dataDir,omitempty"`
    ClusterToken    string   `json:"clusterToken,omitempty"`
}

// 修改 ClusterConfiguration
type ClusterConfiguration struct {
    // ... 现有字段 ...
    
    // ControlPlane 控制面组件部署配置
    ControlPlane ControlPlane `json:"controlPlane,omitempty"`
}
```

### 2.2 完整配置示例

#### API Server 节点配置
```yaml
apiVersion: kubeadm.k8s.io/v1beta5
kind: ClusterConfiguration
kubernetesVersion: v1.32.0
controlPlaneEndpoint: "lb.example.com:6443"
controlPlane:
  mode: apiserver-only
  externalComponents:
    scheduler:
      endpoints:
        - "https://sched-1:10259"
        - "https://sched-2:10259"
    controllerManager:
      endpoints:
        - "https://cm-1:10257"
        - "https://cm-2:10257"
  etcdCluster:
    managed: true
    endpoints:
      - "https://etcd-1:2379"
      - "https://etcd-2:2379"
      - "https://etcd-3:2379"
    certsDir: /etc/kubernetes/pki/etcd
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
    etcd-servers: "https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379"
    etcd-cafile: /etc/kubernetes/pki/etcd/ca.crt
    etcd-certfile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    etcd-keyfile: /etc/kubernetes/pki/apiserver-etcd-client.key
---
apiVersion: kubeadm.k8s.io/v1beta5
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.1.101"
  bindPort: 6443
```

#### Scheduler 节点配置
```yaml
apiVersion: kubeadm.k8s.io/v1beta5
kind: ClusterConfiguration
kubernetesVersion: v1.32.0
controlPlane:
  mode: scheduler-only
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
---
apiVersion: kubeadm.k8s.io/v1beta5
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: "lb.example.com:6443"
    token: "abcdef.0123456789abcdef"
    caCertHashes:
      - "sha256:..."
controlPlane:
  localAPIEndpoint:
    advertiseAddress: "192.168.1.201"
```

#### etcd-only 节点配置
```yaml
apiVersion: kubeadm.k8s.io/v1beta5
kind: ClusterConfiguration
kubernetesVersion: v1.32.0
controlPlane:
  mode: etcd-only
  etcdCluster:
    managed: true
    dataDir: /var/lib/etcd
    certsDir: /etc/kubernetes/pki/etcd
    clusterToken: "etcd-cluster-token-abc123"
---
apiVersion: kubeadm.k8s.io/v1beta5
kind: InitConfiguration
```

### 2.3 Phases 机制扩展
**文件**: `cmd/kubeadm/app/cmd/phases/init.go`
```go
// 修改 init 阶段的组件部署逻辑
func runControlPlanePhases(c *initData) error {
    mode := c.Cfg().ControlPlane.Mode
    if mode == "" {
        mode = kubeadmapi.ControlPlaneModeAll
    }
    
    // 部署 API Server (除非是纯 scheduler/CM/etcd 节点)
    if mode != kubeadmapi.ControlPlaneModeSchedulerOnly && 
       mode != kubeadmapi.ControlPlaneModeControllerManagerOnly &&
       mode != kubeadmapi.ControlPlaneModeEtcdOnly {
        if err := phaseapiserver.DeployAPIServer(c); err != nil {
            return err
        }
    }
    
    // 部署 Scheduler (全模式或 scheduler-only)
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeSchedulerOnly {
        if err := phasescheduler.DeployScheduler(c); err != nil {
            return err
        }
    }
    
    // 部署 Controller Manager (全模式或 controller-manager-only)
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeControllerManagerOnly {
        if err := phasecontrollermanager.DeployControllerManager(c); err != nil {
            return err
        }
    }
    
    // 部署 etcd (全模式、apiserver-etcd 或 etcd-only)
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeAPIServerEtcd ||
       mode == kubeadmapi.ControlPlaneModeEtcdOnly {
        if err := phaseetcd.DeployEtcd(c); err != nil {
            return err
        }
    }
    
    return nil
}
```

### 2.4 Static Pod Manifest 生成修改
**文件**: `cmd/kubeadm/app/phases/controlplane/controlplane.go`
```go
// CreateStaticPodFiles 修改支持分离模式
func CreateStaticPodFiles(c controlPlaneComponent, cfg *kubeadmapi.ClusterConfiguration, 
    manifestsDir string, apiServerExtraArgs, schedulerExtraArgs, 
    controllerManagerExtraArgs map[string]string) error {
    
    mode := cfg.ControlPlane.Mode
    if mode == "" {
        mode = kubeadmapi.ControlPlaneModeAll
    }
    
    switch c {
    case apiServer:
        if mode == kubeadmapi.ControlPlaneModeSchedulerOnly ||
           mode == kubeadmapi.ControlPlaneModeControllerManagerOnly ||
           mode == kubeadmapi.ControlPlaneModeEtcdOnly {
            klog.V(1).Infof("Skipping API server deployment in %s mode", mode)
            return nil
        }
        return createAPIServerStaticPodFiles(cfg, manifestsDir, apiServerExtraArgs)
        
    case scheduler:
        if mode != kubeadmapi.ControlPlaneModeAll && 
           mode != kubeadmapi.ControlPlaneModeSchedulerOnly {
            klog.V(1).Infof("Skipping scheduler deployment in %s mode", mode)
            return nil
        }
        return createSchedulerStaticPodFiles(cfg, manifestsDir, schedulerExtraArgs)
        
    case controllerManager:
        if mode != kubeadmapi.ControlPlaneModeAll && 
           mode != kubeadmapi.ControlPlaneModeControllerManagerOnly {
            klog.V(1).Infof("Skipping controller-manager deployment in %s mode", mode)
            return nil
        }
        return createControllerManagerStaticPodFiles(cfg, manifestsDir, controllerManagerExtraArgs)
        
    case etcd:
        if mode == kubeadmapi.ControlPlaneModeAPIServerOnly ||
           mode == kubeadmapi.ControlPlaneModeSchedulerOnly ||
           mode == kubeadmapi.ControlPlaneModeControllerManagerOnly {
            klog.V(1).Infof("Skipping etcd deployment in %s mode", mode)
            return nil
        }
        return createEtcdStaticPodFiles(cfg, manifestsDir)
    }
    
    return nil
}
```

### 2.5 etcd 成员管理
**文件**: `cmd/kubeadm/app/phases/etcd/members.go` (新增)
```go
package etcd

// AddEtcdMember 添加新成员到现有 etcd 集群
func AddEtcdMember(cfg *kubeadmapi.EtcdClusterConfig, newMemberName, newMemberPeerURL string) error {
    client, err := newEtcdClient(cfg.Endpoints, cfg.CertsDir)
    if err != nil {
        return err
    }
    defer client.Close()
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    resp, err := client.MemberAdd(ctx, []string{newMemberPeerURL})
    if err != nil {
        return fmt.Errorf("failed to add etcd member: %w", err)
    }
    
    klog.V(1).Infof("Added etcd member %s with ID %d", newMemberName, resp.Member.ID)
    return nil
}

// RemoveEtcdMember 从 etcd 集群移除成员
func RemoveEtcdMember(cfg *kubeadmapi.EtcdClusterConfig, memberID uint64) error {
    client, err := newEtcdClient(cfg.Endpoints, cfg.CertsDir)
    if err != nil {
        return err
    }
    defer client.Close()
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    _, err = client.MemberRemove(ctx, memberID)
    if err != nil {
        return fmt.Errorf("failed to remove etcd member: %w", err)
    }
    
    return nil
}

// GetEtcdMemberList 获取 etcd 成员列表
func GetEtcdMemberList(cfg *kubeadmapi.EtcdClusterConfig) ([]*clientv3.Member, error) {
    client, err := newEtcdClient(cfg.Endpoints, cfg.CertsDir)
    if err != nil {
        return nil, err
    }
    defer client.Close()
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    resp, err := client.MemberList(ctx)
    if err != nil {
        return nil, err
    }
    
    return resp.Members, nil
}
```

### 2.6 kubeadm join 扩展
**文件**: `cmd/kubeadm/app/cmd/phases/join.go`
```go
// 修改 join 阶段的控制面组件部署
func runControlPlaneJoinPhases(c *joinData) error {
    mode := c.Cfg().ControlPlane.Mode
    if mode == "" {
        mode = kubeadmapi.ControlPlaneModeAll
    }
    
    // 下载控制面证书
    if err := phaseuploadconfig.DownloadCerts(c); err != nil {
        return err
    }
    
    // 根据模式部署组件
    if mode != kubeadmapi.ControlPlaneModeSchedulerOnly && 
       mode != kubeadmapi.ControlPlaneModeControllerManagerOnly &&
       mode != kubeadmapi.ControlPlaneModeEtcdOnly {
        if err := phaseapiserver.DeployAPIServer(c); err != nil {
            return err
        }
    }
    
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeSchedulerOnly {
        if err := phasescheduler.DeployScheduler(c); err != nil {
            return err
        }
    }
    
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeControllerManagerOnly {
        if err := phasecontrollermanager.DeployControllerManager(c); err != nil {
            return err
        }
    }
    
    if mode == kubeadmapi.ControlPlaneModeAll || 
       mode == kubeadmapi.ControlPlaneModeAPIServerEtcd ||
       mode == kubeadmapi.ControlPlaneModeEtcdOnly {
        if err := phaseetcd.DeployEtcd(c); err != nil {
            return err
        }
    }
    
    return nil
}
```

### 2.7 kubeadm upgrade 兼容
**文件**: `cmd/kubeadm/app/cmd/upgrade.go`
```go
// 升级时保持 controlPlane 配置
func upgradeConfigMap(oldCfg, newCfg *kubeadmapi.ClusterConfiguration) {
    // 保留 controlPlane 配置
    if oldCfg.ControlPlane.Mode != "" {
        newCfg.ControlPlane.Mode = oldCfg.ControlPlane.Mode
    }
    if len(oldCfg.ControlPlane.ExternalComponents.Scheduler.Endpoints) > 0 {
        newCfg.ControlPlane.ExternalComponents = oldCfg.ControlPlane.ExternalComponents
    }
    if len(oldCfg.ControlPlane.EtcdCluster.Endpoints) > 0 {
        newCfg.ControlPlane.EtcdCluster = oldCfg.ControlPlane.EtcdCluster
    }
    
    // ... 其他升级逻辑 ...
}
```

### 2.8 kubeadm-config ConfigMap 扩展
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeadm-config
  namespace: kube-system
data:
  ClusterConfiguration: |
    apiVersion: kubeadm.k8s.io/v1beta5
    kind: ClusterConfiguration
    kubernetesVersion: v1.32.0
    controlPlane:
      mode: apiserver-only
      externalComponents:
        scheduler:
          endpoints:
            - "https://sched-1:10259"
            - "https://sched-2:10259"
        controllerManager:
          endpoints:
            - "https://cm-1:10257"
            - "https://cm-2:10257"
      etcdCluster:
        managed: true
        endpoints:
          - "https://etcd-1:2379"
          - "https://etcd-2:2379"
          - "https://etcd-3:2379"
    # ... 其他配置 ...
```

### 2.9 完整部署架构
```
┌────────────────────────────────────────────────────────────────────┐
│                    完全分离的控制面架构                              │
│                                                                    │
│  API Server 节点池 (3 节点)           etcd 节点池 (3/5/7 节点)       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐ ┌──────────┐ │
│  │ APIS-1   │ │ APIS-2   │ │ APIS-3   │  │ etcd-1   │ │ etcd-2   │ │
│  │ mode:    │ │ mode:    │ │ mode:    │  │ mode:    │ │ mode:    │ │
│  │apiserver │ │apiserver │ │apiserver │  │ etcd-only│ │ etcd-only│ │
│  │-only     │ │-only     │ │-only     │  │          │ │          │ │
│  └──────────┘ └──────────┘ └──────────┘  └──────────┘ └──────────┘ │
│                                                                    │
│  Scheduler 节点池 (2 节点)     Controller Manager 节点池 (2 节点)    │
│  ┌──────────┐ ┌──────────┐     ┌──────────┐ ┌──────────┐           │
│  │ Sched-1  │ │ Sched-2  │     │ CM-1     │ │ CM-2     │           │
│  │ mode:    │ │ mode:    │     │ mode:    │ │ mode:    │           │
│  │scheduler │ │scheduler │     │controller│ │controller│           │
│  │-only     │ │-only     │     │-mgr-only │ │-mgr-only │           │
│  └──────────┘ └──────────┘     └──────────┘ └──────────┘           │
│                                                                    │
│  关键设计:                                                          │
│  1. 各组件池独立扩缩容                                               │
│  2. 通过 externalComponents 配置跨组件连接                           │
│  3. etcd 证书通过 Secret 分发到所有需要连接的节点                     │
│  4. API Server 节点持有 etcd client 证书                            │
│  5. 健康检查跨组件池进行                                             │
└────────────────────────────────────────────────────────────────────┘
```

## 三、Cluster-API 支持控制面组件分离的设计方案

### 3.1 架构设计
```
┌─────────────────────────────────────────────────────────────────────┐
│                    CAPI 分离控制面架构                               │
│                                                                     │
│  用户定义 SeparatedControlPlane CRD                                 │
│      │                                                              │
│      ├── APIServerPool (MachineDeployment)                          │
│      │   ├── replicas: 3                                            │
│      │   ├── mode: apiserver-only                                   │
│      │   └── 关联的 KubeadmConfigTemplate                           │
│      │                                                              │
│      ├── SchedulerPool (MachineDeployment)                          │
│      │   ├── replicas: 2                                            │
│      │   ├── mode: scheduler-only                                   │
│      │   └── 关联的 KubeadmConfigTemplate                           │
│      │                                                              │
│      ├── ControllerManagerPool (MachineDeployment)                  │
│      │   ├── replicas: 2                                            │
│      │   ├── mode: controller-manager-only                          │
│      │   └── 关联的 KubeadmConfigTemplate                           │
│      │                                                              │
│      └── EtcdPool (MachineDeployment)                               │
│          ├── replicas: 3                                            │
│          ├── mode: etcd-only                                        │
│          └── 关联的 KubeadmConfigTemplate                           │
│                                                                    │
│  SeparatedControlPlane Controller                                  │
│      │                                                             │
│      ├── 协调各组件池的生命周期                                      │
│      ├── 管理组件间连接配置                                          │
│      ├── 处理升级顺序 (etcd → APIServer → CM → Scheduler)           │
│      └── 聚合健康状态                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 CRD 设计
**文件**: `api/controlplane/separated/v1alpha1/separatedcontrolplane_types.go`
```go
package v1alpha1

// SeparatedControlPlane 定义分离的控制面
type SeparatedControlPlane struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   SeparatedControlPlaneSpec   `json:"spec,omitempty"`
    Status SeparatedControlPlaneStatus `json:"status,omitempty"`
}

type SeparatedControlPlaneSpec struct {
    Version string `json:"version"`
    
    APIServer         ComponentPoolSpec `json:"apiServer"`
    Scheduler         ComponentPoolSpec `json:"scheduler"`
    ControllerManager ComponentPoolSpec `json:"controllerManager"`
    Etcd              EtcdPoolSpec      `json:"etcd"`
    
    LoadBalancer LoadBalancerSpec `json:"loadBalancer,omitempty"`
    Upgrade      UpgradeStrategy  `json:"upgrade,omitempty"`
}

// ComponentPoolSpec 定义组件池配置
type ComponentPoolSpec struct {
    Replicas            int32               `json:"replicas"`
    MachineTemplate     MachineTemplateRef  `json:"machineTemplate"`
    KubeadmConfigTemplate ConfigTemplateRef `json:"kubeadmConfigTemplate"`
    RolloutStrategy     RolloutStrategy     `json:"rolloutStrategy,omitempty"`
    HealthCheck         HealthCheckSpec     `json:"healthCheck,omitempty"`
}

// EtcdPoolSpec 定义 etcd 池配置
type EtcdPoolSpec struct {
    ComponentPoolSpec `json:",inline"`
    ClusterConfig     EtcdClusterConfig `json:"clusterConfig,omitempty"`
    ManagedMembership bool              `json:"managedMembership,omitempty"`
}

// EtcdClusterConfig etcd 集群配置
type EtcdClusterConfig struct {
    DataDir               string `json:"dataDir,omitempty"`
    SnapshotCount         int32  `json:"snapshotCount,omitempty"`
    AutoCompactionRetention string `json:"autoCompactionRetention,omitempty"`
    QuotaBackendBytes     string `json:"quotaBackendBytes,omitempty"`
}

// LoadBalancerSpec 负载均衡器配置
type LoadBalancerSpec struct {
    Endpoint    string         `json:"endpoint"`
    Port        int32          `json:"port"`
    HealthCheck *LBHealthCheck `json:"healthCheck,omitempty"`
}

// UpgradeStrategy 升级策略
type UpgradeStrategy struct {
    Order                 []ComponentType      `json:"order,omitempty"`
    MaxUnavailable        intstr.IntOrString   `json:"maxUnavailable,omitempty"`
    WaitBetweenUpgrades   metav1.Duration      `json:"waitBetweenUpgrades,omitempty"`
}

type ComponentType string

const (
    ComponentTypeEtcd              ComponentType = "etcd"
    ComponentTypeAPIServer         ComponentType = "apiServer"
    ComponentTypeControllerManager ComponentType = "controllerManager"
    ComponentTypeScheduler         ComponentType = "scheduler"
)

// SeparatedControlPlaneStatus 状态
type SeparatedControlPlaneStatus struct {
    Initialization ControlPlaneInitializationStatus `json:"initialization,omitempty"`
    
    APIServerStatus         PoolStatus     `json:"apiServerStatus,omitempty"`
    SchedulerStatus         PoolStatus     `json:"schedulerStatus,omitempty"`
    ControllerManagerStatus PoolStatus     `json:"controllerManagerStatus,omitempty"`
    EtcdStatus              EtcdPoolStatus `json:"etcdStatus,omitempty"`
    
    Available  bool               `json:"available,omitempty"`
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// PoolStatus 组件池状态
type PoolStatus struct {
    Replicas          int32  `json:"replicas"`
    ReadyReplicas     int32  `json:"readyReplicas"`
    AvailableReplicas int32  `json:"availableReplicas"`
    Version           string `json:"version"`
    MachineDeploymentName string `json:"machineDeploymentName"`
    Conditions        []metav1.Condition `json:"conditions,omitempty"`
}

// EtcdPoolStatus etcd 池状态
type EtcdPoolStatus struct {
    PoolStatus `json:",inline"`
    Members        []EtcdMemberStatus `json:"members,omitempty"`
    ClusterHealthy bool               `json:"clusterHealthy,omitempty"`
    Leader         string             `json:"leader,omitempty"`
}

// EtcdMemberStatus etcd 成员状态
type EtcdMemberStatus struct {
    ID       string `json:"id"`
    Name     string `json:"name"`
    Endpoint string `json:"endpoint"`
    Healthy  bool   `json:"healthy"`
    IsLeader bool   `json:"isLeader"`
}
```

### 3.3 Controller 实现
**文件**: `controlplane/separated/internal/controllers/separatedcontrolplane_controller.go`
```go
package controllers

type SeparatedControlPlaneReconciler struct {
    client.Client
    APIReader client.Reader
    Recorder  record.EventRecorder
}

func (r *SeparatedControlPlaneReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    scp := &separatedv1alpha1.SeparatedControlPlane{}
    if err := r.Get(ctx, req.NamespacedName, scp); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if !controllerutil.ContainsFinalizer(scp, separatedv1alpha1.Finalizer) {
        controllerutil.AddFinalizer(scp, separatedv1alpha1.Finalizer)
        return ctrl.Result{}, r.Update(ctx, scp)
    }
    
    if !scp.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, scp)
    }
    
    return r.reconcile(ctx, scp)
}

func (r *SeparatedControlPlaneReconciler) reconcile(ctx context.Context, scp *separatedv1alpha1.SeparatedControlPlane) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Reconciling SeparatedControlPlane")
    
    // 1. 创建/更新 etcd 池
    if res, err := r.reconcileEtcdPool(ctx, scp); err != nil || !res.IsZero() {
        return res, err
    }
    
    // 2. 等待 etcd 就绪
    if !r.isEtcdReady(scp) {
        log.Info("Waiting for etcd pool to be ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 3. 创建/更新 API Server 池
    if res, err := r.reconcileAPIServerPool(ctx, scp); err != nil || !res.IsZero() {
        return res, err
    }
    
    // 4. 等待 API Server 就绪
    if !r.isAPIServerReady(scp) {
        log.Info("Waiting for API Server pool to be ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 5. 创建/更新 Controller Manager 池
    if res, err := r.reconcileControllerManagerPool(ctx, scp); err != nil || !res.IsZero() {
        return res, err
    }
    
    // 6. 创建/更新 Scheduler 池
    if res, err := r.reconcileSchedulerPool(ctx, scp); err != nil || !res.IsZero() {
        return res, err
    }
    
    // 7. 更新状态
    if err := r.updateStatus(ctx, scp); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}

// reconcileEtcdPool 调和 etcd 池
func (r *SeparatedControlPlaneReconciler) reconcileEtcdPool(ctx context.Context, 
    scp *separatedv1alpha1.SeparatedControlPlane) (ctrl.Result, error) {
    
    md := &clusterv1.MachineDeployment{}
    err := r.Get(ctx, types.NamespacedName{
        Namespace: scp.Namespace,
        Name:      scp.Name + "-etcd",
    }, md)
    
    if apierrors.IsNotFound(err) {
        md = r.buildEtcdMachineDeployment(scp)
        if err := r.Create(ctx, md); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    desiredMD := r.buildEtcdMachineDeployment(scp)
    if !reflect.DeepEqual(md.Spec, desiredMD.Spec) {
        md.Spec = desiredMD.Spec
        return ctrl.Result{}, r.Update(ctx, md)
    }
    
    return ctrl.Result{}, nil
}

// buildEtcdMachineDeployment 构建 etcd MachineDeployment
func (r *SeparatedControlPlaneReconciler) buildEtcdMachineDeployment(
    scp *separatedv1alpha1.SeparatedControlPlane) *clusterv1.MachineDeployment {
    
    return &clusterv1.MachineDeployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      scp.Name + "-etcd",
            Namespace: scp.Namespace,
            Labels: map[string]string{
                clusterv1.ClusterNameLabel: scp.Name,
                "component":                "etcd",
            },
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(scp, separatedv1alpha1.GroupVersion.WithKind("SeparatedControlPlane")),
            },
        },
        Spec: clusterv1.MachineDeploymentSpec{
            Replicas: ptr.To(scp.Spec.Etcd.Replicas),
            Selector: metav1.LabelSelector{
                MatchLabels: map[string]string{
                    clusterv1.ClusterNameLabel: scp.Name,
                    "component":                "etcd",
                },
            },
            Template: clusterv1.MachineTemplateSpec{
                Spec: clusterv1.MachineSpec{
                    ClusterName: scp.Name,
                    Version:     ptr.To(scp.Spec.Version),
                    Bootstrap: clusterv1.Bootstrap{
                        ConfigRef: &corev1.ObjectReference{
                            APIVersion: scp.Spec.Etcd.KubeadmConfigTemplate.APIVersion,
                            Kind:       scp.Spec.Etcd.KubeadmConfigTemplate.Kind,
                            Name:       scp.Name + "-etcd-config",
                        },
                    },
                    InfrastructureRef: corev1.ObjectReference{
                        APIVersion: scp.Spec.Etcd.MachineTemplate.APIVersion,
                        Kind:       scp.Spec.Etcd.MachineTemplate.Kind,
                        Name:       scp.Spec.Etcd.MachineTemplate.Name,
                    },
                },
            },
        },
    }
}

// buildAPIServerKubeadmConfigTemplate 构建 API Server 的 KubeadmConfigTemplate
func (r *SeparatedControlPlaneReconciler) buildAPIServerKubeadmConfigTemplate(
    scp *separatedv1alpha1.SeparatedControlPlane) *bootstrapv1.KubeadmConfigTemplate {
    
    etcdEndpoints := r.getEtcdEndpoints(scp)
    
    return &bootstrapv1.KubeadmConfigTemplate{
        ObjectMeta: metav1.ObjectMeta{
            Name:      scp.Name + "-apiserver-config",
            Namespace: scp.Namespace,
        },
        Spec: bootstrapv1.KubeadmConfigTemplateSpec{
            Template: bootstrapv1.KubeadmConfigTemplateResource{
                Spec: bootstrapv1.KubeadmConfigSpec{
                    ClusterConfiguration: &bootstrapv1.ClusterConfiguration{
                        APIServer: bootstrapv1.APIServer{
                            ControlPlaneComponent: bootstrapv1.ControlPlaneComponent{
                                ExtraArgs: []bootstrapv1.Arg{
                                    {Name: "etcd-servers", Value: strings.Join(etcdEndpoints, ",")},
                                },
                            },
                        },
                        Etcd: bootstrapv1.Etcd{
                            External: &bootstrapv1.ExternalEtcd{
                                Endpoints: etcdEndpoints,
                                CAFile:    "/etc/kubernetes/pki/etcd/ca.crt",
                                CertFile:  "/etc/kubernetes/pki/apiserver-etcd-client.crt",
                                KeyFile:   "/etc/kubernetes/pki/apiserver-etcd-client.key",
                            },
                        },
                    },
                },
            },
        },
    }
}
```

### 3.4 升级协调
**文件**: `controlplane/separated/internal/controllers/upgrade.go`
```go
// reconcileUpgrade 协调跨组件池的升级
func (r *SeparatedControlPlaneReconciler) reconcileUpgrade(ctx context.Context,
    scp *separatedv1alpha1.SeparatedControlPlane) (ctrl.Result, error) {
    
    log := ctrl.LoggerFrom(ctx)
    
    order := scp.Spec.Upgrade.Order
    if len(order) == 0 {
        order = []separatedv1alpha1.ComponentType{
            separatedv1alpha1.ComponentTypeEtcd,
            separatedv1alpha1.ComponentTypeAPIServer,
            separatedv1alpha1.ComponentTypeControllerManager,
            separatedv1alpha1.ComponentTypeScheduler,
        }
    }
    
    for _, component := range order {
        needsUpgrade, err := r.componentNeedsUpgrade(ctx, scp, component)
        if err != nil {
            return ctrl.Result{}, err
        }
        
        if !needsUpgrade {
            continue
        }
        
        log.Info("Upgrading component", "component", component)
        
        if res, err := r.upgradeComponent(ctx, scp, component); err != nil || !res.IsZero() {
            return res, err
        }
        
        if !r.isComponentUpgraded(ctx, scp, component) {
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
        
        if scp.Spec.Upgrade.WaitBetweenUpgrades.Duration > 0 {
            log.Info("Waiting between component upgrades", 
                "duration", scp.Spec.Upgrade.WaitBetweenUpgrades.Duration)
            return ctrl.Result{RequeueAfter: scp.Spec.Upgrade.WaitBetweenUpgrades.Duration}, nil
        }
    }
    
    return ctrl.Result{}, nil
}

// upgradeComponent 升级单个组件池
func (r *SeparatedControlPlaneReconciler) upgradeComponent(ctx context.Context,
    scp *separatedv1alpha1.SeparatedControlPlane, 
    component separatedv1alpha1.ComponentType) (ctrl.Result, error) {
    
    var mdName string
    switch component {
    case separatedv1alpha1.ComponentTypeEtcd:
        mdName = scp.Name + "-etcd"
    case separatedv1alpha1.ComponentTypeAPIServer:
        mdName = scp.Name + "-apiserver"
    case separatedv1alpha1.ComponentTypeControllerManager:
        mdName = scp.Name + "-controller-manager"
    case separatedv1alpha1.ComponentTypeScheduler:
        mdName = scp.Name + "-scheduler"
    }
    
    md := &clusterv1.MachineDeployment{}
    if err := r.Get(ctx, types.NamespacedName{Namespace: scp.Namespace, Name: mdName}, md); err != nil {
        return ctrl.Result{}, err
    }
    
    if md.Spec.Template.Spec.Version == nil || *md.Spec.Template.Spec.Version != scp.Spec.Version {
        original := md.DeepCopy()
        md.Spec.Template.Spec.Version = ptr.To(scp.Spec.Version)
        return ctrl.Result{}, r.Patch(ctx, md, client.MergeFrom(original))
    }
    
    return ctrl.Result{}, nil
}
```

### 3.5 健康检查
**文件**: `controlplane/separated/internal/controllers/health.go`
```go
// reconcileHealth 协调跨组件池的健康检查
func (r *SeparatedControlPlaneReconciler) reconcileHealth(ctx context.Context,
    scp *separatedv1alpha1.SeparatedControlPlane) error {
    
    // 1. 检查 etcd 集群健康
    etcdHealthy, err := r.checkEtcdClusterHealth(ctx, scp)
    if err != nil {
        conditions.Set(scp, metav1.Condition{
            Type:    separatedv1alpha1.EtcdClusterHealthyCondition,
            Status:  metav1.ConditionUnknown,
            Reason:  separatedv1alpha1.EtcdHealthCheckFailedReason,
            Message: err.Error(),
        })
        return err
    }
    
    if etcdHealthy {
        conditions.Set(scp, metav1.Condition{
            Type:   separatedv1alpha1.EtcdClusterHealthyCondition,
            Status: metav1.ConditionTrue,
            Reason: separatedv1alpha1.EtcdClusterHealthyReason,
        })
    }
    
    // 2. 检查 API Server 健康
    apiHealthy, err := r.checkAPIServerHealth(ctx, scp)
    
    // 3. 检查 Controller Manager 健康
    cmHealthy, err := r.checkControllerManagerHealth(ctx, scp)
    
    // 4. 检查 Scheduler 健康
    schedHealthy, err := r.checkSchedulerHealth(ctx, scp)
    
    // 5. 聚合整体可用状态
    available := etcdHealthy && apiHealthy && cmHealthy && schedHealthy
    scp.Status.Available = available
    
    if available {
        conditions.Set(scp, metav1.Condition{
            Type:   clusterv1.AvailableCondition,
            Status: metav1.ConditionTrue,
            Reason: clusterv1.AvailableReason,
        })
    }
    
    return nil
}

// checkEtcdClusterHealth 检查 etcd 集群健康
func (r *SeparatedControlPlaneReconciler) checkEtcdClusterHealth(ctx context.Context,
    scp *separatedv1alpha1.SeparatedControlPlane) (bool, error) {
    
    endpoints := r.getEtcdEndpoints(scp)
    if len(endpoints) == 0 {
        return false, nil
    }
    
    client, err := newEtcdClient(endpoints, scp.Spec.Etcd.ClusterConfig.DataDir+"/pki/etcd")
    if err != nil {
        return false, err
    }
    defer client.Close()
    
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    resp, err := client.AlarmList(ctx)
    if err != nil {
        return false, err
    }
    
    if len(resp.Alarms) > 0 {
        return false, fmt.Errorf("etcd cluster has alarms: %v", resp.Alarms)
    }
    
    members, err := client.MemberList(ctx)
    if err != nil {
        return false, err
    }
    
    expectedMembers := int(scp.Spec.Etcd.Replicas)
    if len(members.Members) != expectedMembers {
        return false, fmt.Errorf("etcd member count mismatch: expected %d, got %d", 
            expectedMembers, len(members.Members))
    }
    
    return true, nil
}
```

### 3.6 ClusterClass 集成
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: separated-control-plane-class
spec:
  kubernetesVersions:
    - v1.31.0
    - v1.32.0
  
  controlPlane:
    templateRef:
      apiVersion: controlplane.separated.cluster.x-k8s.io/v1alpha1
      kind: SeparatedControlPlaneTemplate
      name: separated-cp-template
  
  variables:
    - name: apiServerReplicas
      schema:
        openAPIV3Schema:
          type: integer
          default: 3
    - name: etcdReplicas
      schema:
        openAPIV3Schema:
          type: integer
          default: 3
    - name: schedulerReplicas
      schema:
        openAPIV3Schema:
          type: integer
          default: 2
    - name: controllerManagerReplicas
      schema:
        openAPIV3Schema:
          type: integer
          default: 2
```

### 3.7 完整部署示例
```yaml
apiVersion: controlplane.separated.cluster.x-k8s.io/v1alpha1
kind: SeparatedControlPlane
metadata:
  name: my-separated-cp
  namespace: default
spec:
  version: v1.32.0
  
  apiServer:
    replicas: 3
    machineTemplate:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalMachineTemplate
      name: apiserver-machine-template
    kubeadmConfigTemplate:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
      kind: KubeadmConfigTemplate
      name: apiserver-config-template
    healthCheck:
      interval: 10s
      timeout: 5s
  
  scheduler:
    replicas: 2
    machineTemplate:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalMachineTemplate
      name: scheduler-machine-template
    kubeadmConfigTemplate:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
      kind: KubeadmConfigTemplate
      name: scheduler-config-template
  
  controllerManager:
    replicas: 2
    machineTemplate:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalMachineTemplate
      name: cm-machine-template
    kubeadmConfigTemplate:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
      kind: KubeadmConfigTemplate
      name: cm-config-template
  
  etcd:
    replicas: 3
    machineTemplate:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalMachineTemplate
      name: etcd-machine-template
    kubeadmConfigTemplate:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
      kind: KubeadmConfigTemplate
      name: etcd-config-template
    managedMembership: true
    clusterConfig:
      dataDir: /var/lib/etcd
      autoCompactionRetention: "1"
      quotaBackendBytes: "8589934592"
  
  loadBalancer:
    endpoint: "lb.example.com"
    port: 6443
  
  upgrade:
    order:
      - etcd
      - apiServer
      - controllerManager
      - scheduler
    maxUnavailable: 1
    waitBetweenUpgrades: 30s
```

## 四、实现路线图
| 阶段 | 内容 | 工作量 |
|------|------|--------|
| **kubeadm 侧** | | |
| Phase 1 | API 类型扩展 (mode + externalComponents + etcdCluster) | 2 周 |
| Phase 2 | Phases 机制修改 (支持跳过组件部署) | 3 周 |
| Phase 3 | etcd-only 模式实现 + 成员管理 | 3 周 |
| Phase 4 | 证书分发机制扩展 | 2 周 |
| Phase 5 | kubeadm join/upgrade 兼容 | 2 周 |
| **CAPI 侧** | | |
| Phase 6 | SeparatedControlPlane CRD 定义 | 1 周 |
| Phase 7 | Controller 基础实现 (创建/更新组件池) | 3 周 |
| Phase 8 | 升级协调逻辑 | 2 周 |
| Phase 9 | 健康检查 + 状态聚合 | 2 周 |
| Phase 10 | ClusterClass 集成 + 测试 | 3 周 |
| **总计** | | **23 周 (~6 个月)** |

## 五、风险与缓解
| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 向后兼容性 | 现有配置可能受影响 | 默认 mode=all，行为不变 |
| 证书管理复杂度 | 多节点证书同步困难 | 复用现有证书分发机制 |
| 社区接受度 | KEP 审批周期长 | 先作为 Alpha 特性，feature gate 控制 |
| 升级兼容性 | 跨版本升级可能失败 | 保留旧配置格式支持 |
| etcd 成员管理 | 动态增删成员复杂 | 实现自动化成员管理控制器 |

## 六、KEP 要点
```yaml
title: kubeadm Control Plane Component Separation
authors:
  - "@your-name"
status: provisional
creation-date: 2024-01-01
stage: alpha

summary: >
  Add support for deploying control plane components (API Server, Scheduler, 
  Controller Manager, etcd) on separate nodes through kubeadm.

motivation: >
  Large-scale clusters may require physical separation of control plane 
  components for resource isolation, compliance, or architectural reasons.
  Currently kubeadm binds all components to the same node.

goals:
  - Allow deploying API Server, Scheduler, Controller Manager, etcd on separate nodes
  - Maintain backward compatibility with existing kubeadm configurations
  - Support both stacked and external etcd in separated mode
  - Integrate with Cluster-API for lifecycle management

non-goals:
  - Automatic component discovery or load balancing
  - Support for separating other Kubernetes components (kube-proxy, etc.)
```
