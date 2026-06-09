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

