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

