# 详细总结 KCP 和 MachineDeployment 在扩缩容时的约束与限制：

## KCP 与 MachineDeployment 扩缩容约束与限制

### 一、KubeadmControlPlane (KCP) 的约束

#### **1. Preflight Checks 阻止扩缩容的场景**

位置：`controlplane/kubeadm/internal/controllers/scale.go:188-276`

| 场景 | 说明 | 影响 |
|------|------|------|
| **Topology 版本不匹配** | 使用 ClusterClass 时，Cluster 的 topology 版本与 KCP 版本不一致 | 阻止扩缩容 |
| **证书缺失** | 集群证书不可用或未知（仅影响扩容） | 阻止扩容 |
| **有 Machine 正在删除** | 存在 DeletionTimestamp 的 Machine | 阻止新的扩缩容操作 |
| **控制平面组件不健康** | API Server、Controller Manager、Scheduler 不健康 | 阻止扩缩容 |
| **etcd 集群不健康** | etcd Pod 不健康或 etcd 成员不健康（仅管理 etcd 时） | 阻止扩缩容 |

#### **2. 特殊豁免场景**

```
 Remediation 期间的扩容：
 当 KCP 有 RemediationInProgressAnnotation 注解时，
 会执行更宽松的健康检查（checkHealthinessWhileRemediationInProgress），
 允许在部分节点不健康时创建替换节点。
```

#### **3. 具体实例**

**场景 A：证书缺失阻止扩容**
```yaml
# 当前状态
KCP:
  replicas: 1 → 3  # 尝试扩容
  
# 如果 KubeadmControlPlaneCertificatesAvailableCondition = False
# 结果：扩容被阻止，日志显示 "Certificates are missing or unknown, can't join a new machine"
```

**场景 B：有节点正在删除**
```yaml
# 当前状态
KCP:
  replicas: 3 → 5  # 尝试扩容
  
Machines:
- control-plane-abc12 (Deleting)  # 正在删除中
- control-plane-def34 (Running)
- control-plane-ghi56 (Running)

# 结果：扩容被阻止，等待 control-plane-abc12 删除完成
```

**场景 C：etcd 不健康阻止操作**
```yaml
# 当前状态（管理 etcd）
Machines:
- control-plane-abc12: EtcdPodHealthyCondition = False
- control-plane-def34: EtcdPodHealthyCondition = True
- control-plane-ghi56: EtcdPodHealthyCondition = True

# 结果：扩缩容都被阻止，等待 etcd 恢复健康
```

### 二、MachineDeployment 的约束

#### **1. 模板引用缺失**

位置：`internal/controllers/machinedeployment/machinedeployment_status.go:265`

| 场景 | 说明 |
|------|------|
| **BootstrapTemplate 不存在** | 引用的 KubeadmConfigTemplate 等资源被删除 |
| **InfrastructureMachineTemplate 不存在** | 引用的 AWSMachineTemplate、DockerMachineTemplate 等被删除 |

**结果：** 状态显示 "Scaling up would be blocked because XXX does not exist"

#### **2. RollingUpdate 策略约束**

位置：`internal/controllers/machinedeployment/machinedeployment_rollout_rollingupdate.go`

MachineDeployment 使用 `maxSurge` 和 `maxUnavailable` 控制滚动更新：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 最多允许超出期望副本数的数量
    maxUnavailable: 0  # 最多允许不可用的副本数
```

**约束规则：**

| 场景 | 行为 |
|------|------|
| **maxSurge 限制** | 当前 Machine 数 < replicas + maxSurge 时才创建新 Machine |
| **maxUnavailable 限制** | 可用副本数不能低于 replicas - maxUnavailable |
| **新 MS 未完全可用** | 等待新 MachineSet 的所有副本变为 Available 后才继续 |

#### **3. 死锁检测与解除**

位置：`machinedeployment_rollout_rollingupdate.go:570-625`

**死锁场景：**
```
MD: replicas=3, maxSurge=1, maxUnavailable=0

OldMS: 3 replicas, 2 available (1 unavailable)
NewMS: 1 replica, 1 available

问题：
- 无法扩容 NewMS（已达 maxSurge 限制）
- 无法缩容 OldMS（会违反 maxUnavailable）

解决：
Controller 检测到死锁后，强制缩容 OldMS 1 个副本，
删除那个 unavailable 的 Machine
```

#### **4. 具体实例**

**场景 A：模板缺失阻止扩容**
```yaml
MachineDeployment:
  spec:
    replicas: 3
    template:
      spec:
        bootstrap:
          configRef:
            name: my-kubeadm-config-template  # 此资源已被删除
        infrastructureRef:
          name: my-aws-machine-template       # 此资源已被删除

# 结果：状态显示 "Scaling up would be blocked because 
#        KubeadmBootstrapTemplate and DockerMachineTemplate do not exist"
```

**场景 B：maxUnavailable=0 时的滚动更新**
```yaml
MachineDeployment:
  replicas: 6
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 不允许任何不可用

滚动更新过程：
1. 创建 NewMS，扩容到 1 (6 + 1 = 7，达到 maxSurge 限制)
2. 等待 NewMS 的 1 个 Machine 变为 Available
3. 缩容 OldMS 到 5
4. 重复步骤 1-3，直到全部更新完成

注意：如果 NewMS 的 Machine 无法变为 Available，滚动更新将卡住
```

**场景 C：死锁自动解除**
```yaml
MachineDeployment:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

状态：
OldMS: 3 replicas (2 available, 1 unavailable - 节点故障)
NewMS: 1 replica (1 available)

死锁检测触发：
- OldMS 有 unavailable 副本
- NewMS 已 fully available
- 没有扩缩容在进行中

自动解除：
OldMS 被强制缩容到 2，删除那个 unavailable 的 Machine
```

### 三、对比总结

| 特性 | KCP | MachineDeployment |
|------|-----|-------------------|
| **健康检查** | 严格的 Preflight Checks（组件健康、etcd 健康） | 无严格健康检查 |
| **证书依赖** | 证书缺失阻止扩容 | 不依赖证书 |
| **删除中阻塞** | 有 Machine 删除时阻止新操作 | 不阻塞 |
| **滚动策略** | 无 maxSurge/maxUnavailable 概念 | 严格遵守 maxSurge/maxUnavailable |
| **死锁处理** | 无死锁检测 | 有自动死锁解除机制 |
| **模板缺失** | 会记录状态但仍可操作 | 阻止扩容并显示警告 |
| **版本升级阻塞** | Topology 版本不匹配时阻塞 | 版本变化触发滚动更新 |

### 四、常见"修改副本数但不生效"的场景

| 场景 | KCP | MachineDeployment |
|------|-----|-------------------|
| 有节点正在删除 | ✅ 阻止 | ❌ 不阻止 |
| 控制平面组件不健康 | ✅ 阻止 | ❌ 不阻止 |
| etcd 不健康 | ✅ 阻止 | ❌ 不适用 |
| 证书缺失 | ✅ 阻止扩容 | ❌ 不阻止 |
| BootstrapTemplate 缺失 | ❌ 不阻止 | ✅ 阻止扩容 |
| InfrastructureTemplate 缺失 | ❌ 不阻止 | ✅ 阻止扩容 |
| 滚动更新中（maxSurge 限制） | ❌ 不适用 | ✅ 等待 |
| 滚动更新死锁 | ❌ 不适用 | ✅ 自动解除 |

# 缩容时，CAPI Controller 能选择删除具体的哪些节点吗？
**结论先说：**  
**是的，CAPI（Cluster API）在缩容时 *可以选择删除具体哪些节点*，而且它有一套明确的“优先级算法（delete policy）”来决定删除顺序。**  
你也可以通过注解、label、finalizer 等方式 **强制指定要删除或不删除的节点**。

下面我把整个机制讲得非常清晰，包含：  
- 缩容时谁决定删除哪个节点  
- 删除优先级算法  
- 如何强制指定删除某个节点  
- 如何保护某个节点不被删  
- KCP / MachineDeployment / MachineSet 的差异  

## 一句话总结  
**缩容时，CAPI Controller（MachineSet / MachineDeployment / KCP）会根据 deletePolicy 和节点状态自动选择要删除的 Machine。  
你也可以通过注解强制指定删除或保护某个节点。**

## 1. 缩容时，谁负责选择要删除的节点？

取决于你缩容的是哪种资源：

| 资源类型 | 负责选择删除节点的控制器 |
|---------|---------------------------|
| **MachineDeployment** | MachineDeployment Controller |
| **MachineSet** | MachineSet Controller |
| **KubeadmControlPlane** | KCP Controller |

**Infra Provider（如 Metal3、vSphere、Baremetal）不决定删除顺序，只执行删除动作。**

## 2. CAPI 的删除优先级算法（核心）

CAPI 的删除顺序由 `deletePolicy` 决定：
```yaml
spec:
  deletePolicy: Oldest
```
支持三种策略：

### ① **Oldest**（默认）
优先删除最早创建的 Machine。

### ② **Newest**
优先删除最新创建的 Machine。

### ③ **Random**
随机删除。

## 3. 除了 deletePolicy，CAPI 还有“隐式优先级”

即使 deletePolicy 是 Oldest，CAPI 仍然会优先删除：

### ✔ **NotReady 的节点**  
### ✔ **Failed 的 Machine**  
### ✔ **Provisioning 失败的 Machine**  
### ✔ **没有 NodeRef 的 Machine**  
### ✔ **被标记为要删除的 Machine（见下一节）**

也就是说：
> **不健康节点优先被删。**

## 4. 如何强制指定删除某个节点？（你肯定会用到）

给 Machine 加注解：
```yaml
metadata:
  annotations:
    cluster.x-k8s.io/delete-machine: "true"
```
效果：
> **该 Machine 会在下一次缩容时被优先删除。**

这是 CAPI 官方支持的“强制删除”机制。

## 5. 如何保护某个节点不被删除？

给 Machine 加注解：
```yaml
metadata:
  annotations:
    cluster.x-k8s.io/delete-machine: "false"
```

或加 finalizer：
```yaml
metadata:
  finalizers:
  - my-protection-finalizer
```

效果：
> **该 Machine 在缩容时不会被删除，除非你移除 finalizer。**

## 6. KubeadmControlPlane（KCP）缩容的特殊逻辑

KCP 缩容时：
- 不能删除 etcd quorum 关键节点  
- 会优先删除 **非 etcd 成员**（如果 external etcd）  
- 会优先删除 **最新加入的控制平面节点**  
- 会确保控制平面节点数量安全  

删除顺序：
1. 被标记 delete-machine 的 Machine  
2. NotReady / unhealthy 的 Machine  
3. 最新创建的控制平面节点（Newest）  
4. 最后才按 deletePolicy（默认 Oldest）

## 7. MachineDeployment 缩容的删除顺序

MachineDeployment → MachineSet → Machine

删除顺序：
1. 被标记 delete-machine 的 Machine  
2. 不健康 Machine  
3. 根据 deletePolicy（Oldest / Newest / Random）

## 8. 一句话总结（再次强调）

> **CAPI 在缩容时会自动选择要删除的节点，依据 deletePolicy + 健康状态 + 注解。  
你可以通过注解强制指定删除或保护某个节点。**

# KCP Controller在缩容时是只根据Replicas判断吧，在Replicas没有改变，而对Machine标记删除的情况，不会触发缩容的吧
## 关键结论

**KCP Controller 缩容只根据 `numMachines > desiredReplicas` 判断，不会因删除注解而主动触发缩容。**

### 代码证据

位置：`controller.go:550-576`
```go
numMachines := len(controlPlane.Machines)
desiredReplicas := int(*controlPlane.KCP.Spec.Replicas)

switch {
case numMachines < desiredReplicas && numMachines == 0:
    return r.initializeControlPlane(ctx, controlPlane)  // 初始化
case numMachines < desiredReplicas && numMachines > 0:
    return r.scaleUpControlPlane(ctx, controlPlane)     // 扩容
case numMachines > desiredReplicas:
    return r.scaleDownControlPlane(ctx, controlPlane, machineToDelete)  // 缩容
}
```
**只有 `numMachines > desiredReplicas` 时才会触发缩容。**

### 删除注解的真实作用

`DeleteMachineAnnotation` 注解有两个作用：

#### 1. **在缩容时优先被选中**（当缩容已经触发时）

位置：`scale.go:403-407`
```go
case controlPlane.MachineWithDeleteAnnotation(outdatedMachines).Len() > 0:
    eligibleMachines = controlPlane.MachineWithDeleteAnnotation(outdatedMachines)
case controlPlane.MachineWithDeleteAnnotation(controlPlane.Machines).Len() > 0:
    eligibleMachines = controlPlane.MachineWithDeleteAnnotation(controlPlane.Machines)
```
这意味着：**当缩容已经触发时**，有注解的节点会被优先删除。

#### 2. **不参与 in-place update**

位置：`filters.go:69-71`
```go
if _, ok := machine.Annotations[clusterv1.DeleteMachineAnnotation]; ok {
    res.EligibleForInPlaceUpdate = false
}
```

### 具体场景示例

#### **场景 1：Replicas 不变，标记删除注解**
```yaml
# KCP 配置
spec:
  replicas: 3  # 保持不变

# 当前状态
Machines:
- control-plane-abc12 (有 delete-machine 注解)
- control-plane-def34
- control-plane-ghi56
```
**结果：**
- `numMachines (3) == desiredReplicas (3)`
- **不会触发缩容**
- 有注解的节点不会被 KCP 删除
- 该节点不会参与 in-place update

#### **场景 2：缩容 + 标记删除注解**

```yaml
# KCP 配置
spec:
  replicas: 3 → 2  # 缩容触发

# 当前状态
Machines:
- control-plane-abc12 (有 delete-machine 注解)
- control-plane-def34
- control-plane-ghi56
```
**结果：**
- `numMachines (3) > desiredReplicas (2)`
- **触发缩容**
- 有注解的 `control-plane-abc12` 会被**优先选中删除**

#### **场景 3：用户手动删除 Machine**

```yaml
# 用户直接删除 Machine 资源
kubectl delete machine control-plane-abc12

# 之后状态
Machines:
- control-plane-def34
- control-plane-ghi56
```
**结果：**
- `numMachines (2) < desiredReplicas (3)`
- **触发扩容**，KCP 会创建新的 Machine 补充到 3 个

### 总结

| 操作 | Replicas 变化 | KCP 行为 |
|------|--------------|----------|
| 仅标记删除注解 | 不变 | **无操作**，不会缩容 |
| 减少 Replicas | 减少 | 缩容，有注解的节点优先删除 |
| 手动删除 Machine | 不变 | **扩容**，补充到目标副本数 |

**如果您想通过注解删除特定节点，需要同时减少 Replicas 值。**

# `cluster.x-k8s.io/delete-machine` 和 `cluster.x-k8s.io/deletion-priority`
**`delete-machine` 是“强制删除”，  
`deletion-priority` 是“删除排序 hint”。**

两者都影响 **CAPI 在缩容或替换节点时删除哪个 Machine**，但强度完全不同。

## 1. **cluster.x-k8s.io/delete-machine**  
### ✔ 语义：**强制删除该 Machine（最高优先级）**

```yaml
metadata:
  annotations:
    cluster.x-k8s.io/delete-machine: "true"
```

效果：
- CAPI 会在下一次 reconcile 时 **立即删除该 Machine**  
- 无论 replicas 是否变化  
- 无论 deletePolicy 是什么  
- 无论 deletion-priority 是多少  

这是一个 **强制删除指令**，用于：
- 手动替换节点  
- 强制删除不健康节点  
- 强制删除卡住的 Machine  
- 自研 provider 中的节点生命周期控制  

### ✔ 删除后会发生什么？

- 如果 replicas 没变 → **KCP/MD 会创建一个新的 Machine（替换）**  
- 如果 replicas 变小 → **这是缩容**  

## 2. **cluster.x-k8s.io/deletion-priority**  
### ✔ 语义：**删除排序 hint（数值越小越优先删除）**

```yaml
metadata:
  annotations:
    cluster.x-k8s.io/deletion-priority: "5"
```
- 不是强制删除  
- 只影响排序  
- 数值越小越优先  
- 可以是负数（负数优先级最高）  

用于：
- 控制缩容时删除哪个节点  
- 控制替换时优先删除哪个节点  
- 多 AZ / 多机房的删除策略  
- 自研 provider 的节点生命周期管理  

## 3. **两者的优先级（非常关键）**

CAPI 删除 Machine 的排序规则如下：
1. **delete-machine=true**（最高优先级）  
2. NotReady / unhealthy 的 Machine  
3. **deletion-priority 数值最小**  
4. deletePolicy（Oldest/Newest/Random）

也就是说：

> **delete-machine 永远优先于 deletion-priority。**

## 4. **两者的行为差异**

| 注解 | 强制删除 | 影响排序 | 是否触发替换 | 是否触发缩容 |
|------|-----------|-----------|----------------|----------------|
| **delete-machine** | ✔ 是 | ✘ 否 | ✔ 是（如果 replicas 不变） | ✔ 是（如果 replicas 变小） |
| **deletion-priority** | ✘ 否 | ✔ 是 | ✘ 否 | ✔ 是（仅在缩容时） |

## 5. **KCP（控制平面）中的行为**

KCP 删除 Machine 的顺序：
1. delete-machine  
2. unhealthy  
3. deletion-priority  
4. newest（默认）  

所以：
- 你标记 delete-machine → KCP 会删除并补一个新的控制平面节点  
- 你设置 deletion-priority → KCP 会在缩容或替换时优先删除该节点  

## 6. **MachineDeployment / MachineSet 中的行为**

删除顺序：
1. delete-machine  
2. unhealthy  
3. deletion-priority  
4. deletePolicy（Oldest 默认）  

适用于：
- worker 节点缩容  
- 滚动升级时的旧节点删除  
- 多 AZ worker 节点管理  

## 7. **典型使用场景**

### ✔ 强制替换某个节点  
```yaml
cluster.x-k8s.io/delete-machine: "true"
```

### ✔ 缩容时优先删除某些节点  
```yaml
cluster.x-k8s.io/deletion-priority: "1"
```

### ✔ 多机房 / 多 AZ 删除策略  
- 先删非核心机房  
- 先删成本高的机房  

### ✔ 自研 Baremetal Provider 的节点生命周期控制  
- 旧硬件优先删除  
- 故障率高的节点优先删除  

## 8. 一句话总结（再次强调）

> **delete-machine = 强制删除（立即执行）  
> deletion-priority = 删除排序 hint（数值越小越优先）**

两者一起构成 CAPI 的 **节点删除决策系统**。

