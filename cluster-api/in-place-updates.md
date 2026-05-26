# Cluster-API In-Place Updates 功能说明与开发指南

## 一、功能概述

### 1.1 背景
Cluster-API 默认使用 **Immutable Rollouts**（不可变滚动更新）策略：当需要更新 Machine 配置时，会创建新 Machine 并删除旧 Machine。这种方式简单可靠，但在某些场景下存在局限：
- **大规模集群**：数百/数千节点重新引导耗时极长（每节点 ~10-15 分钟）
- **资源受限环境**：单节点集群无法承受滚动更新时的资源开销
- **数据库等有状态负载**：重新克隆数据可能需要数小时
- **凭证轮换**：如 SSH 密钥轮换不需要重建机器

### 1.2 In-Place Updates 定义
**In-Place Update**：在不删除和重建 Machine 的情况下，直接修改现有 Machine 上的配置或软件版本。

### 1.3 当前状态
| 属性 | 值 |
|------|-----|
| **Feature Gate** | `InPlaceUpdates` |
| **默认值** | `false`（需显式启用） |
| **成熟度** | **Alpha** |
| **引入版本** | CAPI v1.8 (experimental) |

## 二、核心架构

### 2.1 三大 Hook
| Hook 名称 | 调用时机 | 用途 |
|-----------|----------|------|
| **CanUpdateMachine** | 升级规划阶段 (KCP) | 判断扩展是否能处理特定 Machine 的变更 |
| **CanUpdateMachineSet** | 升级规划阶段 (MD) | 判断扩展是否能处理 MachineSet 的变更 |
| **UpdateMachine** | 升级执行阶段 | 执行实际的原地更新操作 |

### 2.2 更新决策流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                    In-Place Update 决策流程                          │
│                                                                     │
│  1. 用户修改 KCP/MD spec (如 version, labels, etc.)                  │
│     ↓                                                               │
│  2. KCP/MD Controller 检测变更                                       │
│     ↓                                                               │
│  3. 调用 CanUpdateMachine / CanUpdateMachineSet Hook                │
│     ├── 请求: current state + desired state                         │
│     ├── 响应: 支持的变更 patches (JSONPatch/JSONMergePatch)          │
│     └── 判断: 所有变更是否都被扩展覆盖?                               │
│     ↓                                                              │
│  4. 决策分支                                                        │
│     ├── 全部覆盖 → 进入 In-Place Update 路径                         │
│     │   ├── 标记 Machine 为 Updating in-place                      │
│     │   ├── Machine Controller 调用 UpdateMachine Hook             │
│     │   ├── 扩展执行更新 (可轮询: RetryAfterSeconds > 0)            │
│     │   └── 更新完成 → 标记 Machine 为 UpToDate                     │
│     │                                                              │
│     └── 未全部覆盖 → Fallback 到 Rolling Update                     │
│         ├── 创建新 Machine (新配置)                                 │
│         ├── 等待新 Machine Ready                                    │
│         └── 删除旧 Machine                                          │
└────────────────────────────────────────────────────────────────────┘
```

### 2.3 关键设计原则
| 原则 | 说明 |
|------|------|
| **用户体验一致** | 触发方式与滚动更新相同（修改 spec） |
| **Fallback 机制** | 无法原地更新时自动回退到滚动更新 |
| **关注点分离** | 扩展决定能否更新，CAPI 决定何时更新 |
| **幂等性** | UpdateMachine Hook 必须幂等，可被多次调用 |
| **轮询支持** | 通过 RetryAfterSeconds 支持异步操作 |

## 三、API 类型详解

### 3.1 CanUpdateMachine Hook
**请求结构**:
```go
type CanUpdateMachineRequest struct {
    Current CanUpdateMachineRequestObjects `json:"current"`
    Desired CanUpdateMachineRequestObjects `json:"desired"`
}

type CanUpdateMachineRequestObjects struct {
    Machine               clusterv1.Machine       `json:"machine"`
    InfrastructureMachine runtime.RawExtension   `json:"infrastructureMachine"`
    BootstrapConfig       runtime.RawExtension   `json:"bootstrapConfig"`
}
```

**响应结构**:
```go
type CanUpdateMachineResponse struct {
    Status                       ResponseStatus `json:"status"`
    Message                      string         `json:"message"`
    MachinePatch                 Patch          `json:"machinePatch"`
    InfrastructureMachinePatch   Patch          `json:"infrastructureMachinePatch"`
    BootstrapConfigPatch         Patch          `json:"bootstrapConfigPatch"`
}
```

### 3.2 UpdateMachine Hook
**请求结构**:
```go
type UpdateMachineRequest struct {
    Desired UpdateMachineRequestObjects `json:"desired"`
}
```

**响应结构**:
```go
type UpdateMachineResponse struct {
    Status          ResponseStatus `json:"status"`
    Message         string         `json:"message"`
    RetryAfterSeconds int32        `json:"retryAfterSeconds"`
}
```

**响应状态说明**:

| Status | RetryAfterSeconds | 含义 |
|--------|-------------------|------|
| Success | 0 | 更新完成 |
| Success | > 0 | 更新进行中，X 秒后重试 |
| Failure | 任意值 | 更新失败 |

## 四、开发指南

### 4.1 项目结构
```
my-inplace-updater/
├── cmd/
│   └── main.go              # 入口
├── internal/
│   └── handlers/
│       ├── discovery.go     # Discovery Hook
│       └── inplace.go       # In-Place Update Hooks
├── go.mod
└── go.sum
```

### 4.2 完整实现示例
**main.go**:
```go
package main

import (
    "flag"
    "os"

    "github.com/spf13/pflag"
    "k8s.io/apimachinery/pkg/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"

    runtimehooksv1 "sigs.k8s.io/cluster-api/api/runtime/hooks/v1alpha1"
    runtimecatalog "sigs.k8s.io/cluster-api/exp/runtime/catalog"
    "sigs.k8s.io/cluster-api/exp/runtime/server"
    "my-inplace-updater/internal/handlers"
)

var (
    catalog = runtimecatalog.New()
    scheme  = runtime.NewScheme()
)

func init() {
    _ = clientgoscheme.AddToScheme(scheme)
    _ = runtimehooksv1.AddToCatalog(catalog)
}

func main() {
    var webhookPort int
    pflag.IntVar(&webhookPort, "webhook-port", 9443, "Webhook Server port")
    pflag.Parse()

    ctrl.SetLogger(ctrl.Log)

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{Scheme: scheme})
    if err != nil {
        os.Exit(1)
    }

    webhookServer, err := server.New(server.Options{
        Port:    webhookPort,
        Catalog: catalog,
    })
    if err != nil {
        os.Exit(1)
    }

    updaterHandlers := handlers.NewInPlaceUpdater(mgr.GetClient())

    // 注册 In-Place Update Hooks
    if err := webhookServer.AddExtensionHandler(server.ExtensionHandler{
        Hook:        runtimehooksv1.CanUpdateMachine,
        Name:        "can-update-machine",
        HandlerFunc: updaterHandlers.CanUpdateMachine,
    }); err != nil {
        os.Exit(1)
    }

    if err := webhookServer.AddExtensionHandler(server.ExtensionHandler{
        Hook:        runtimehooksv1.UpdateMachine,
        Name:        "update-machine",
        HandlerFunc: updaterHandlers.UpdateMachine,
    }); err != nil {
        os.Exit(1)
    }

    if err := mgr.AddReadyzCheck("webhook", webhookServer.StartedChecker()); err != nil {
        os.Exit(1)
    }

    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        os.Exit(1)
    }
}
```

**handlers/inplace.go**:
```go
package handlers

import (
    "context"
    "fmt"

    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    runtimehooksv1 "sigs.k8s.io/cluster-api/api/runtime/hooks/v1alpha1"
)

type InPlaceUpdater struct {
    client client.Client
}

func NewInPlaceUpdater(c client.Client) *InPlaceUpdater {
    return &InPlaceUpdater{client: c}
}

// CanUpdateMachine 判断是否支持原地更新
func (h *InPlaceUpdater) CanUpdateMachine(
    ctx context.Context,
    request *runtimehooksv1.CanUpdateMachineRequest,
    response *runtimehooksv1.CanUpdateMachineResponse,
) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("CanUpdateMachine called")

    currentVersion := request.Current.Machine.Spec.Version
    desiredVersion := request.Desired.Machine.Spec.Version

    // 检查是否为版本升级
    if currentVersion != desiredVersion {
        // 声明支持版本升级的原地更新
        response.MachinePatch = runtimehooksv1.Patch{
            PatchType: "JSONPatch",
            Patch: []byte(fmt.Sprintf(
                `[{"op":"replace","path":"/spec/version","value":"%s"}]`,
                desiredVersion,
            )),
        }
    }

    response.Status = runtimehooksv1.ResponseStatusSuccess
}

// UpdateMachine 执行原地更新
func (h *InPlaceUpdater) UpdateMachine(
    ctx context.Context,
    request *runtimehooksv1.UpdateMachineRequest,
    response *runtimehooksv1.UpdateMachineResponse,
) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("UpdateMachine called")

    machine := request.Desired.Machine
    targetVersion := machine.Spec.Version

    // 1. 获取机器连接信息 (SSH/IPMI 等)
    ip := h.getMachineIP(machine)
    creds := h.getMachineCredentials(machine)

    // 2. 连接到目标机器
    client, err := h.connectToMachine(ip, creds)
    if err != nil {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = fmt.Sprintf("Connection failed: %v", err)
        return
    }

    // 3. 执行升级脚本
    script := h.buildUpgradeScript(targetVersion)
    output, err := client.Run(script)
    if err != nil {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = fmt.Sprintf("Upgrade failed: %v\n%s", err, output)
        return
    }

    // 4. 验证升级结果
    verifyOutput, err := client.Run("kubeadm version -o short && kubelet --version")
    if err != nil {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.Message = fmt.Sprintf("Verification failed: %v", err)
        return
    }

    log.Info("Upgrade completed", "output", verifyOutput)
    response.Status = runtimehooksv1.ResponseStatusSuccess
    response.RetryAfterSeconds = 0
}

func (h *InPlaceUpdater) buildUpgradeScript(version string) string {
    return fmt.Sprintf(`#!/bin/bash
set -euo pipefail

# 备份当前版本
BACKUP_DIR=/opt/k8s-backup/$(date +%%Y%%m%%d_%%H%%M%%S)
mkdir -p "$BACKUP_DIR"
cp /usr/bin/kubeadm "$BACKUP_DIR/" 2>/dev/null || true
cp /usr/bin/kubelet "$BACKUP_DIR/" 2>/dev/null || true

# 下载并安装新版本
curl -fSL -o /usr/bin/kubeadm https://dl.k8s.io/release/%s/bin/linux/amd64/kubeadm
curl -fSL -o /usr/bin/kubelet https://dl.k8s.io/release/%s/bin/linux/amd64/kubelet
chmod +x /usr/bin/kubeadm /usr/bin/kubelet

# 执行升级
kubeadm upgrade apply %s -y

# 重启 kubelet
systemctl daemon-reload
systemctl restart kubelet
`, version, version, version)
}
```

### 4.3 部署配置
**ExtensionConfig**:
```yaml
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: my-inplace-updater
spec:
  clientConfig:
    service:
      name: my-inplace-updater-service
      namespace: capi-system
      port: 443
  namespaceSelector:
    matchLabels:
      inplace-updates-enabled: "true"
```

**启用 Feature Gate**:
```bash
# 在 CAPI 控制器启动参数中添加
--feature-gates=InPlaceUpdates=true
```

## 五、使用示例

### 5.1 触发 In-Place Update

```yaml
# 修改 KCP 版本触发原地升级
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: my-cluster-cp
spec:
  replicas: 3
  version: v1.32.0  # 从 v1.31.0 升级到 v1.32.0
```

### 5.2 观察更新状态
```bash
# 查看 Machine 状态
kubectl get machines
# 输出示例:
# NAME           CLUSTER     READY   UPDATING   VERSION
# cp-node-1      my-cluster  True    True       v1.32.0

# 查看 Machine 条件
kubectl describe machine cp-node-1
# Conditions:
#   Type: Updating
#   Status: True
#   Reason: InPlaceUpdating
#   Message: In-place update in progress
```

## 六、限制与注意事项
| 限制 | 说明 |
|------|------|
| **不支持自动回滚** | 更新失败需手动修复或替换机器 |
| **当前仅支持 1 个 Updater** | 未来会支持多个扩展协同 |
| **Alpha 阶段** | API 可能变化，不建议生产使用 |
| **Remediation 行为** | 更新期间 Machine 可能不健康，MHC 行为需注意 |
| **Infrastructure 变更** | 如镜像变更，扩展需决定如何处理 |

## 七、与 Rolling Update 对比
| 维度 | In-Place Update | Rolling Update |
|------|-----------------|----------------|
| **机器重建** | 否 | 是 |
| **速度** | 快 (无需重新引导) | 慢 (需完整引导流程) |
| **资源开销** | 低 | 高 (需额外资源) |
| **数据保留** | 保留 | 丢失 (除非持久化) |
| **回滚** | 手动 | 自动 (修改版本回退) |
| **成熟度** | Alpha | GA |
| **适用场景** | 大规模/资源受限/有状态 | 通用场景 |

## 八、未来演进方向
| 方向 | 说明 |
|------|------|
| **多 Updater 支持** | 支持多个扩展协同处理不同变更 |
| **自动回滚** | 更新失败时自动回退到上一版本 |
| **排序与依赖** | 定义多个更新扩展的执行顺序 |
| **MHC 集成** | 更新期间的健康检查特殊处理 |
| **进度可观测** | 更详细的更新进度和状态报告 |
| **Beta/GA** | API 稳定化，提升成熟度 |


# **`Machine.Spec.Version` 是 Kubernetes 版本**。
在 Cluster-API 中，这个字段用于指定该 Machine 上应该运行的 Kubernetes 版本。

### 关键说明
| 属性 | 说明 |
|------|------|
| **格式** | 语义化版本，如 `v1.31.0`、`v1.32.0` |
| **用途** | 告诉 Bootstrap Provider 和 Infrastructure Provider 该节点应使用哪个 K8s 版本 |
| **设置者** | 由 KCP (KubeadmControlPlane) 或 MachineDeployment 控制器自动设置 |
| **传递路径** | KCP/MD → Machine → Bootstrap Provider → 节点上的 kubelet/kubeadm |

### 源码中的定义
```go
// MachineSpec 中的 Version 字段
type MachineSpec struct {
    // ...
    // version defines the desired Kubernetes version.
    // This field is meant to be optionally used by bootstrap providers.
    // +optional
    Version string `json:"version,omitempty"`
    // ...
}
```

### 实际使用示例
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Machine
metadata:
  name: my-cluster-cp-abc12
spec:
  clusterName: my-cluster
  version: v1.31.0        # ← 这就是 Kubernetes 版本
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
      kind: KubeadmConfig
      name: my-cluster-cp-abc12
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: BareMetalMachine
    name: my-cluster-cp-abc12
```

### 版本升级时的行为
当用户修改 KCP 的 `spec.version` 时：
```yaml
# 修改前
spec:
  version: v1.31.0

# 修改后
spec:
  version: v1.32.0   # ← 触发滚动更新或原地更新
```
CAPI 控制器会：
1. 检测到版本变更
2. 将新版本传播到 Machine 的 `spec.version`
3. 触发 Machine 的更新流程（滚动替换或原地升级）

# **`CanUpdateMachine` 原地更新条件判断清单**

## 支持原地更新的场景
- **配置参数调整**  
  - kubelet 配置（如 `--max-pods`、`--eviction-hard`）  
  - 内核参数或 sysctl 调整  
  - 不涉及重启的轻量化配置更新  

- **镜像替换但版本不变**  
  - 基础镜像相同，仅替换容器运行时或工具包  
  - 不影响 kubelet 或 API Server 版本  

- **标签/注解更新**  
  - 节点标签、污点、注解的修改  
  - 不涉及节点重建  

## 不支持原地更新的场景
- **Kubernetes 版本升级**  
  - `spec.version` 发生变化，需要滚动替换节点  
- **操作系统镜像变更**  
  - 例如从 Ubuntu → CentOS，必须重建节点  
- **硬件规格变更**  
  - CPU、内存、磁盘大小调整，云厂商通常要求重新创建实例  

## 判断逻辑示例
```go
if request.Machine.Spec.Version == request.DesiredVersion &&
   request.Machine.Spec.Image == request.DesiredImage {
    response.Status = runtimehooksv1.ResponseStatusSuccess
    response.CanUpdate = true
    response.Reason = "Safe in-place update: config change only"
} else {
    response.Status = runtimehooksv1.ResponseStatusFailure
    response.CanUpdate = false
    response.Reason = "Requires rolling update: version or image change"
}
```

## 判断为支持原地更新的方式
在响应对象 `response *runtimehooksv1.CanUpdateMachineResponse` 中，需要设置：
- **允许更新标志**  
  ```go
  response.Status = runtimehooksv1.ResponseStatusSuccess
  response.CanUpdate = true
  ```
  表示 Hook 执行成功，并且允许原地更新。

- **原因说明**  
  可以设置 `response.Reason` 字段，说明为什么支持原地更新，例如：
  ```go
  response.Reason = "In-place update supported for kubelet config change"
  ```
### 示例代码片段
```go
func (h *InPlaceUpdater) CanUpdateMachine(
    ctx context.Context,
    request *runtimehooksv1.CanUpdateMachineRequest,
    response *runtimehooksv1.CanUpdateMachineResponse,
) {
    // 判断条件，例如只允许 kubelet 配置更新
    if request.Machine.Spec.Version == request.DesiredVersion {
        response.Status = runtimehooksv1.ResponseStatusSuccess
        response.CanUpdate = true
        response.Reason = "Version unchanged, safe to update in place"
    } else {
        response.Status = runtimehooksv1.ResponseStatusFailure
        response.CanUpdate = false
        response.Reason = "Version change requires rolling update"
    }
}
```
### 总结
- **支持原地更新** → `response.Status = Success` 且 `response.CanUpdate = true`。  
- **不支持原地更新** → `response.Status = Failure` 或 `response.CanUpdate = false`。  
- **原因说明** → 用 `response.Reason` 字段补充。  

- **支持原地更新** → 配置、标签、轻量化调整。  
- **不支持原地更新** → 版本升级、镜像/OS 变更、硬件规格调整。  
- **响应判断** → `response.Status = Success` 且 `response.CanUpdate = true`。  

