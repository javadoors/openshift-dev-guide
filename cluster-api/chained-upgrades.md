# Cluster-API Chained Upgrades 功能说明与开发指南

## 一、功能概述

### 1.1 背景
Kubernetes 官方升级策略要求控制面和 Worker 之间最多只能相差 **1 个 minor 版本**。这意味着从 v1.28 升级到 v1.32 需要经过多个中间版本：
```
v1.28 → v1.29 → v1.30 → v1.31 → v1.32
```
CAPI 的 **Chained Upgrades** 功能允许用户通过修改一次 `spec.topology.version`，自动编排整个链式升级过程。

### 1.2 核心概念
| 概念 | 说明 |
|------|------|
| **Chained Upgrade** | 从当前版本到目标版本的完整升级序列，包含所有中间版本 |
| **Efficient Upgrade** | 在满足版本倾斜策略前提下，最小化升级步骤 |
| **Upgrade Plan** | 由系统或扩展生成的升级路径，包含控制面和 Worker 各自的升级步骤 |

### 1.3 当前状态
| 属性 | 值 |
|------|-----|
| **成熟度** | GA (通过 `GenerateUpgradePlan` Hook) |
| **引入版本** | CAPI v1.9+ |
| **依赖** | 必须使用 ClusterClass (Managed Topology) |

## 二、功能列表

### 2.1 核心能力
| 能力 | 说明 |
|------|------|
| **自动生成升级路径** | 系统根据 `ClusterClass.spec.kubernetesVersions` 自动计算中间版本 |
| **控制面/Worker 独立路径** | 支持控制面和 Worker 有不同的升级步骤 |
| **版本倾斜策略强制** | 自动确保升级过程不违反 K8s 版本倾斜策略 |
| **Runtime Extension 集成** | 通过 `GenerateUpgradePlan` Hook 自定义升级路径 |
| **Lifecycle Hooks 集成** | 每个中间版本升级都会触发相应的 Lifecycle Hooks |

### 2.2 升级流程
```
用户修改 Cluster.spec.topology.version: v1.28 → v1.32
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 1. 系统计算升级路径                                      │
│    控制面: v1.28 → v1.29 → v1.30 → v1.31 → v1.32        │
│    Worker:  v1.28 → v1.29 → v1.30 → v1.31 → v1.32      │
│    (或更高效的路径，如果版本倾斜策略允许)                  │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 2. 对每个中间版本执行:                                    │
│    ├── BeforeClusterUpgrade (仅首次)                     │
│    ├── BeforeControlPlaneUpgrade                         │
│    ├── 升级控制面                                        │
│    ├── AfterControlPlaneUpgrade                          │
│    ├── BeforeWorkersUpgrade                              │
│    ├── 升级 Worker                                       │
│    └── AfterWorkersUpgrade                               │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 3. AfterClusterUpgrade (全部完成后)                      │
└─────────────────────────────────────────────────────────┘
```

### 2.3 高效升级示例
```
场景: 从 v1.29.0 升级到 v1.32.0

标准链式升级:
  控制面: v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
  Worker:  v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0

高效升级 (利用版本倾斜策略):
  控制面: v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
  Worker:  v1.29.0 ─────────────→ v1.31.0 → v1.32.0
           (Worker 可以跳过 v1.30.0，因为与控制面 v1.31.0 相差 ≤1)
```

## 三、API 类型详解

### 3.1 ClusterClass 中的版本列表
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: my-cluster-class
spec:
  # 定义允许的 K8s 版本列表 (必须按从旧到新排序)
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.31.0
    - v1.32.0
```
**规则**:
- 列表必须按从旧到新的顺序排列
- 第一个和最后一个版本之间的每个 minor 都必须至少有一个版本
- 升级时使用每个 minor 的**最新**版本

### 3.2 GenerateUpgradePlan Hook
**请求结构**:
```go
type GenerateUpgradePlanRequest struct {
    Cluster                       clusterv1.Cluster `json:"cluster"`
    FromControlPlaneKubernetesVersion string        `json:"fromControlPlaneKubernetesVersion"`
    FromWorkersKubernetesVersion  string            `json:"fromWorkersKubernetesVersion"`
    ToKubernetesVersion           string            `json:"toKubernetesVersion"`
}
```

**响应结构**:
```go
type GenerateUpgradePlanResponse struct {
    Status           ResponseStatus `json:"status"`
    Message          string         `json:"message"`
    ControlPlaneUpgrades []UpgradeStep `json:"controlPlaneUpgrades"`
    WorkersUpgrades      []UpgradeStep `json:"workersUpgrades"`
}

type UpgradeStep struct {
    Version string `json:"version"`
}
```
**规则**:

| 规则 | 控制面升级路径 | Worker 升级路径 |
|------|---------------|----------------|
| **起始版本** | > `fromControlPlaneKubernetesVersion` | > `fromWorkersKubernetesVersion` |
| **结束版本** | 必须等于 `toKubernetesVersion` | 必须等于 `toKubernetesVersion` |
| **中间版本** | 每个 minor 至少一个版本 | 可以为空 (系统自动计算) |
| **版本倾斜** | - | 必须与控制面版本兼容 |

## 四、开发指南

### 4.1 使用系统自动生成的升级路径
如果不需要自定义升级路径，只需在 ClusterClass 中定义 `kubernetesVersions`：
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: my-cluster-class
spec:
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.31.0
    - v1.32.0
  # ... 其他配置
```

用户升级时只需修改 Cluster 版本：
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: my-cluster
spec:
  topology:
    classRef:
      name: my-cluster-class
    version: v1.32.0  # 从 v1.29.0 升级到 v1.32.0
```

### 4.2 自定义升级路径 (Runtime Extension)
**实现 GenerateUpgradePlan Hook**:
```go
package handlers

import (
    "context"
    "fmt"

    ctrl "sigs.k8s.io/controller-runtime"

    runtimehooksv1 "sigs.k8s.io/cluster-api/api/runtime/hooks/v1alpha1"
)

type UpgradePlanHandlers struct {
    client client.Client
}

func NewUpgradePlanHandlers(c client.Client) *UpgradePlanHandlers {
    return &UpgradePlanHandlers{client: c}
}

// DoGenerateUpgradePlan 生成自定义升级路径
func (h *UpgradePlanHandlers) DoGenerateUpgradePlan(
    ctx context.Context,
    request *runtimehooksv1.GenerateUpgradePlanRequest,
    response *runtimehooksv1.GenerateUpgradePlanResponse,
) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("GenerateUpgradePlan hook called",
        "fromCP", request.FromControlPlaneKubernetesVersion,
        "fromWorkers", request.FromWorkersKubernetesVersion,
        "to", request.ToKubernetesVersion)

    // 解析版本
    fromCP := parseVersion(request.FromControlPlaneKubernetesVersion)
    to := parseVersion(request.ToKubernetesVersion)

    // 生成控制面升级路径
    var controlPlaneUpgrades []runtimehooksv1.UpgradeStep
    for minor := fromCP.Minor + 1; minor <= to.Minor; minor++ {
        // 使用每个 minor 的最新版本
        version := fmt.Sprintf("v1.%d.0", minor)
        controlPlaneUpgrades = append(controlPlaneUpgrades, runtimehooksv1.UpgradeStep{
            Version: version,
        })
    }

    // 生成 Worker 升级路径 (高效路径)
    // Worker 可以跳过一些版本，只要不违反版本倾斜策略
    var workersUpgrades []runtimehooksv1.UpgradeStep
    for minor := fromCP.Minor + 1; minor <= to.Minor; minor++ {
        // Worker 只需要在控制面升级完成后升级到兼容版本
        // 这里简化处理，实际需要根据版本倾斜策略计算
        version := fmt.Sprintf("v1.%d.0", minor)
        workersUpgrades = append(workersUpgrades, runtimehooksv1.UpgradeStep{
            Version: version,
        })
    }

    response.ControlPlaneUpgrades = controlPlaneUpgrades
    response.WorkersUpgrades = workersUpgrades
    response.Status = runtimehooksv1.ResponseStatusSuccess
}

func parseVersion(version string) *semver.Version {
    // 实现版本解析逻辑
    v, _ := semver.ParseTolerant(version)
    return &v
}
```

**注册 Hook**:
```go
if err := webhookServer.AddExtensionHandler(server.ExtensionHandler{
    Hook:        runtimehooksv1.GenerateUpgradePlan,
    Name:        "generate-upgrade-plan",
    HandlerFunc: upgradePlanHandlers.DoGenerateUpgradePlan,
}); err != nil {
    os.Exit(1)
}
```

### 4.3 结合 Lifecycle Hooks 实现完整升级流程
```go
// BeforeClusterUpgrade - 升级前验证
func (h *LifecycleHandlers) DoBeforeClusterUpgrade(
    ctx context.Context,
    request *runtimehooksv1.BeforeClusterUpgradeRequest,
    response *runtimehooksv1.BeforeClusterUpgradeResponse,
) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("BeforeClusterUpgrade",
        "from", request.FromKubernetesVersion,
        "to", request.ToKubernetesVersion,
        "controlPlaneUpgrades", request.ControlPlaneUpgrades,
        "workersUpgrades", request.WorkersUpgrades)

    // 验证升级路径
    if len(request.ControlPlaneUpgrades) == 0 {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = "No control plane upgrade steps defined"
        return
    }

    // 创建升级前快照
    if err := h.createSnapshot(ctx, request.Cluster); err != nil {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = fmt.Sprintf("Snapshot failed: %v", err)
        response.RetryAfterSeconds = 60
        return
    }

    response.Status = runtimehooksv1.ResponseStatusSuccess
}

// AfterClusterUpgrade - 升级完成后处理
func (h *LifecycleHandlers) DoAfterClusterUpgrade(
    ctx context.Context,
    request *runtimehooksv1.AfterClusterUpgradeRequest,
    response *runtimehooksv1.AfterClusterUpgradeResponse,
) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("AfterClusterUpgrade", "version", request.KubernetesVersion)

    // 升级 Add-ons 到兼容版本
    if err := h.upgradeAddons(ctx, request.Cluster, request.KubernetesVersion); err != nil {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = fmt.Sprintf("Addon upgrade failed: %v", err)
        return
    }

    response.Status = runtimehooksv1.ResponseStatusSuccess
}
```

## 五、版本倾斜策略约束

### 5.1 标准 K8s 版本倾斜策略
| 组件 | 相对于 kube-apiserver |
|------|----------------------|
| kubelet | 最多低 1 个 minor 版本 |
| kube-proxy | 最多低 1 个 minor 版本 |
| CoreDNS | 最多低 1 个 minor 版本 |

### 5.2 CAPI 升级时的约束
```
升级过程中:
  控制面版本: vCP
  Worker 版本: vW

  必须满足: vCP - vW ≤ 1 (minor 版本差)
```

### 5.3 高效升级计算示例
```
目标: v1.29.0 → v1.32.0

步骤 1: 控制面升级到 v1.30.0
  控制面: v1.30.0
  Worker:  v1.29.0  (差 1 minor, 允许)
         ↓
步骤 2: Worker 升级到 v1.30.0  ← 关键：Worker 必须先跟上
  控制面: v1.30.0
  Worker:  v1.30.0  (差 0 minor)
         ↓
步骤 3: 控制面升级到 v1.31.0
  控制面: v1.31.0
  Worker:  v1.30.0  (差 1 minor, 允许)
         ↓
步骤 4: Worker 升级到 v1.31.0
  控制面: v1.31.0
  Worker:  v1.31.0  (差 0 minor)
         ↓
步骤 5: 控制面升级到 v1.32.0
  控制面: v1.32.0
  Worker:  v1.31.0  (差 1 minor, 允许)
         ↓
步骤 6: Worker 升级到 v1.32.0
  控制面: v1.32.0
  Worker:  v1.32.0  (完成)
```

#### 为什么不会导致业务中断
| 原因 | 说明 |
|------|------|
| **CAPI 强制约束** | CAPI 控制器在升级前会检查版本倾斜策略，不满足条件时不会继续升级 |
| **滚动升级** | 控制面和 Worker 都是逐个节点升级，不会同时中断所有节点 |
| **LB 流量分发** | API Server 通过负载均衡器分发请求，部分节点升级不影响整体服务 |
| **kubelet 兼容性** | kubelet 设计上支持与 apiserver 相差 1 个 minor 版本正常工作 |

#### 高效升级路径 (Efficient Upgrade)
CAPI 会尝试最小化升级步骤，但仍遵守版本倾斜策略：
```
目标: v1.29.0 → v1.32.0

高效路径:
  控制面: v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
  Worker:  v1.29.0 → v1.30.0 ────────→ v1.31.0 → v1.32.0
  
  (Worker 在控制面到达 v1.31.0 后，可以从 v1.30.0 直接升级到 v1.31.0)
```

## 六、完整配置示例

### 6.1 ClusterClass 配置
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: production-class
spec:
  # 定义允许的 K8s 版本
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.31.0
    - v1.32.0

  infrastructure:
    templateRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalClusterTemplate
      name: production-infra

  controlPlane:
    templateRef:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta2
      kind: KubeadmControlPlaneTemplate
      name: production-cp
    machineInfrastructure:
      templateRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: production-cp-machine

  workers:
    machineDeployments:
      - class: default-workers
        template:
          bootstrap:
            templateRef:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
              kind: KubeadmConfigTemplate
              name: production-md-bootstrap
          infrastructure:
            templateRef:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
              kind: BareMetalMachineTemplate
              name: production-md-machine

  variables:
    - name: registryEndpoint
      required: true
      schema:
        openAPIV3Schema:
          type: string

  patches:
    - name: registry
      definitions:
        - selector:
            apiVersion: controlplane.cluster.x-k8s.io/v1beta2
            kind: KubeadmControlPlaneTemplate
            matchResources:
              controlPlane: true
          jsonPatches:
            - op: add
              path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/imageRepository
              valueFrom:
                variable: registryEndpoint
```

### 6.2 Cluster 配置
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: my-production-cluster
spec:
  topology:
    classRef:
      name: production-class
    version: v1.32.0  # 升级目标版本
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: default-workers
          name: md-0
          replicas: 5
    variables:
      - name: registryEndpoint
        value: "registry.example.com/k8s"
```

## 七、监控与调试

### 7.1 查看升级状态
```bash
# 查看 Cluster 状态
kubectl get cluster my-production-cluster -o yaml

# 查看升级进度
kubectl describe cluster my-production-cluster
# Events:
#   Type     Reason              Age   From                    Message
#   ----     ------              ----  ----                    -------
#   Normal   Upgrading           2m    kubeadm-control-plane   Upgrading control plane to v1.30.0
#   Normal   UpgradeComplete     1m    kubeadm-control-plane   Control plane upgraded to v1.30.0
#   Normal   Upgrading           50s   machine-deployment      Upgrading workers to v1.30.0
```

### 7.2 查看 Machine 版本
```bash
kubectl get machines
# NAME              CLUSTER                  READY   VERSION
# cp-node-abc12     my-production-cluster    True    v1.30.0
# cp-node-def34     my-production-cluster    True    v1.30.0
# cp-node-ghi56     my-production-cluster    False   v1.29.0  # 正在升级
# md-0-xyz78        my-production-cluster    True    v1.29.0
```

## 八、注意事项
| 事项 | 说明 |
|------|------|
| **必须使用 ClusterClass** | Chained Upgrades 仅在 Managed Topology 模式下可用 |
| **版本列表必须连续** | `kubernetesVersions` 中不能有 minor 版本跳跃 |
| **升级不可逆** | CAPI 不支持版本降级 |
| **Extension 可选** | 如果不注册 `GenerateUpgradePlan` Hook，系统会自动计算升级路径 |
| **每个版本都会触发 Hooks** | 每个中间版本升级都会触发完整的 Lifecycle Hooks 序列 |
| **升级时间** | 跨多个 minor 版本升级可能耗时较长，需做好规划 |

# ClusterClass 中 `kubernetesVersions` 的作用

## 一、核心作用
`kubernetesVersions` 是 ClusterClass 中用于**定义允许使用的 Kubernetes 版本列表**的字段。
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: my-cluster-class
spec:
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.31.0
    - v1.32.0
```

## 二、具体功能

### 2.1 版本白名单验证
当用户创建或更新 Cluster 时，CAPI 会验证 `spec.topology.version` 是否在 `kubernetesVersions` 列表中：
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
spec:
  topology:
    classRef:
      name: my-cluster-class
    version: v1.31.0  # ← 必须存在于 kubernetesVersions 中
```
**如果版本不在列表中，Cluster 创建/更新将被拒绝。**

### 2.2 链式升级路径计算
当用户从一个版本升级到另一个版本时，CAPI 使用此列表计算中间版本路径：
```
用户请求: v1.29.0 → v1.32.0

系统根据 kubernetesVersions 计算升级路径:
  控制面: v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
  Worker:  v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
```
**规则**:
- 列表必须按从旧到新排序
- 第一个和最后一个版本之间的**每个 minor 版本**都必须至少有一个版本
- 升级时使用每个 minor 的**最新版本**

### 2.3 与 GenerateUpgradePlan Hook 协同
如果注册了 `GenerateUpgradePlan` Runtime Extension，扩展可以使用此列表作为参考来生成自定义升级路径：
```go
func (h *UpgradePlanHandlers) DoGenerateUpgradePlan(
    ctx context.Context,
    request *runtimehooksv1.GenerateUpgradePlanRequest,
    response *runtimehooksv1.GenerateUpgradePlanResponse,
) {
    // 可以读取 ClusterClass 的 kubernetesVersions
    // 用于生成自定义升级路径
    ...
}
```

## 三、配置规则

### 3.1 格式要求
| 规则 | 说明 | 示例 |
|------|------|------|
| **排序** | 必须从旧到新 | `v1.29.0, v1.30.0, v1.31.0` |
| **连续性** | 每个 minor 至少一个版本 | 不能有 `v1.29.0, v1.31.0` (缺少 v1.30.x) |
| **数量限制** | 1-100 个版本 | - |
| **版本格式** | 语义化版本，带 `v` 前缀 | `v1.31.0` |

### 3.2 正确示例
```yaml
# 正确: 每个 minor 都有版本，按顺序排列
kubernetesVersions:
  - v1.29.0
  - v1.30.0
  - v1.31.0
  - v1.32.0
```

### 3.3 错误示例
```yaml
# 错误 1: 缺少 v1.30.x
kubernetesVersions:
  - v1.29.0
  - v1.31.0  # ← 跳过了 v1.30

# 错误 2: 顺序错误
kubernetesVersions:
  - v1.31.0
  - v1.30.0  # ← 比前一个版本旧

# 错误 3: 缺少 v 前缀
kubernetesVersions:
  - 1.31.0   # ← 应该是 v1.31.0
```

## 四、使用场景

### 4.1 标准升级
```bash
# 用户修改 Cluster 版本
kubectl patch cluster my-cluster --type merge -p '{"spec":{"topology":{"version":"v1.32.0"}}}'

# CAPI 自动:
# 1. 验证 v1.32.0 在 kubernetesVersions 中
# 2. 计算升级路径: v1.31.0 → v1.32.0
# 3. 执行升级
```

### 4.2 跨多版本升级
```bash
# 从 v1.29.0 升级到 v1.32.0
kubectl patch cluster my-cluster --type merge -p '{"spec":{"topology":{"version":"v1.32.0"}}}'

# CAPI 自动:
# 1. 验证 v1.32.0 在 kubernetesVersions 中
# 2. 计算升级路径: v1.29.0 → v1.30.0 → v1.31.0 → v1.32.0
# 3. 逐步执行每个中间版本升级
```

## 五、与 Cluster 版本的关系
```
ClusterClass.spec.kubernetesVersions
    │
    ├── 定义允许的版本集合
    │   ├── v1.29.0
    │   ├── v1.30.0
    │   ├── v1.31.0
    │   └── v1.32.0
    │
    ▼
Cluster.spec.topology.version
    │
    ├── 必须是 kubernetesVersions 中的一个
    │   └── v1.31.0  ← 当前目标版本
    │
    ▼
实际升级路径
    │
    ├── 控制面: v1.30.0 → v1.31.0
    └── Worker:  v1.30.0 → v1.31.0
```

## 六、注意事项
| 事项 | 说明 |
|------|------|
| **可选字段** | `kubernetesVersions` 是可选的，但如果不定义，链式升级将无法工作 |
| **仅 Managed Topology** | 此字段仅在使用 ClusterClass (Managed Topology) 时有效 |
| **升级前必须包含目标版本** | 如果要升级到新版本，必须先将该版本添加到列表中 |
| **与 CAPI 版本支持矩阵无关** | 此列表是 ClusterClass 作者定义的，不受 CAPI 官方支持矩阵限制 |
| **变更需谨慎** | 修改此列表可能影响现有集群的升级路径 |

# "升级时使用每个 minor 的最新版本" 详解

## 一、核心概念区分
需要区分两个概念：

| 概念 | 说明 |
|------|------|
| **kubernetesVersions 列表** | 定义**允许使用的所有版本**（可包含同一 minor 的多个 patch 版本） |
| **升级路径计算** | 系统自动选择每个 minor 的**最新 patch 版本**作为中间步骤 |

## 二、示例说明

### 2.1 ClusterClass 配置
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
spec:
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.30.1
    - v1.30.2
    - v1.31.0
    - v1.31.1
    - v1.32.0
```
**注意**：这个列表中包含了同一 minor 的多个 patch 版本（如 v1.30.0, v1.30.1, v1.30.2）。

### 2.2 用户创建集群
用户可以使用列表中的**任何版本**创建集群：
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
spec:
  topology:
    classRef:
      name: my-cluster-class
    version: v1.30.1  # ← 可以使用列表中的任意版本
```

### 2.3 升级路径计算
当用户从 v1.29.0 升级到 v1.32.0 时：
```
kubernetesVersions 中的版本:
  v1.29.x: v1.29.0
  v1.30.x: v1.30.0, v1.30.1, v1.30.2  ← 最新: v1.30.2
  v1.31.x: v1.31.0, v1.31.1           ← 最新: v1.31.1
  v1.32.x: v1.32.0                    ← 最新: v1.32.0

系统计算的升级路径:
  v1.29.0 → v1.30.2 → v1.31.1 → v1.32.0
  
  (使用每个 minor 的最新 patch 版本)
```

## 三、为什么这样设计？

### 3.1 灵活性
- 用户可以创建使用特定 patch 版本的集群（如 v1.30.1）
- 但升级时自动选择最稳定的路径（最新 patch）

### 3.2 安全性
- 最新 patch 版本通常包含最多的 bug 修复和安全补丁
- 升级路径使用最新 patch 可以减少升级过程中的问题

### 3.3 简化管理
- ClusterClass 作者只需维护一个版本列表
- 系统自动计算最优升级路径

## 四、回答您的问题

### Q1: "升级时使用每个 minor 的最新版本" 如何理解？
**A**: 当计算链式升级路径时，如果 `kubernetesVersions` 中有同一 minor 的多个 patch 版本，系统会选择该 minor 中**版本号最大**的那个作为升级路径中的中间版本。

### Q2: ClusterClass 中的 kubernetesVersions 定义的都是最新版本吗？
**A**: **不一定**。`kubernetesVersions` 可以包含：
- 同一 minor 的多个 patch 版本（如 v1.30.0, v1.30.1, v1.30.2）
- 用户可以使用其中**任何版本**创建集群
- 但在**计算升级路径**时，系统会自动选择每个 minor 的最新版本

## 五、最佳实践建议
```yaml
# 推荐: 只包含每个 minor 的最新版本 (简化配置)
kubernetesVersions:
  - v1.29.0
  - v1.30.2
  - v1.31.1
  - v1.32.0

# 或者: 包含所有版本 (更灵活，但配置更复杂)
kubernetesVersions:
  - v1.29.0
  - v1.30.0
  - v1.30.1
  - v1.30.2
  - v1.31.0
  - v1.31.1
  - v1.32.0
```
**建议**: 对于大多数场景，只需包含每个 minor 的最新版本即可，这样可以简化配置并减少维护成本。

