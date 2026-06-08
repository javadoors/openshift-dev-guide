# Cluster API的Paused设计
**结论先给你：Cluster API 的 *Paused* 设计是一套“全局暂停调谐（Reconciliation）”的机制，用来让用户在需要人工干预、迁移、升级、调试时，安全地阻止 CAPI 控制器对资源进行任何自动修改。**

它不是 kubeadm 的功能，也不是 Kubernetes 的功能，而是 **Cluster API 控制器层的调谐开关**。

下面我把它的设计理念、作用范围、触发链路、控制器行为、最佳实践全部讲清楚。

## 一句话总结  
**`cluster.x-k8s.io/paused` 是 Cluster API 的全局暂停开关，用来阻止 CAPI 控制器对 Cluster/Machine/MachineDeployment/MachineSet 等资源进行调谐。**

它的设计目标是：
> **让用户在需要“冻结集群状态”时，安全地阻止 CAPI 自动调谐。**

## 1. Paused 的设计目的（为什么需要它）

Cluster API 是一个 **声明式自动调谐系统**，会持续 reconcile：
- 创建/删除 Machine  
- 滚动升级  
- 修复节点  
- 调整副本数  
- 生成 bootstrap data  
- 调用 Infra Provider 创建 VM/BM  

但在某些场景下，你不希望它继续自动调谐，例如：
- 手动维护节点  
- 手动修复基础设施  
- 调试 bootstrap 问题  
- 升级 CAPI 版本  
- 迁移集群  
- 避免控制器误操作  

因此需要一个 **全局暂停按钮**。

## 2. Paused 的两种形式

## **Cluster 级暂停（最常用）**

```yaml
metadata:
  annotations:
    cluster.x-k8s.io/paused: "true"
```

效果：
> **暂停整个 Cluster 的所有 CAPI 控制器调谐。**

包括：
- Cluster  
- KubeadmControlPlane  
- Machine  
- MachineDeployment  
- MachineSet  
- InfraCluster  
- InfraMachine  

## **对象级暂停**

你也可以只暂停某个对象：
```yaml
metadata:
  annotations:
    cluster.x-k8s.io/paused: "true"
```
例如：
- 只暂停某个 MachineDeployment  
- 只暂停某个 Machine  
- 只暂停某个 InfraMachine  

效果：
> **只暂停该对象的调谐，不影响其他对象。**

## 3. Paused 的执行者是谁？

### ✔ **Cluster API 控制器（所有 CAPI controllers）**  
包括：
- Cluster controller  
- Machine controller  
- MachineDeployment controller  
- MachineSet controller  
- KubeadmControlPlane controller  
- Bootstrap provider（CABPK）  
- Infra provider（AWS/vSphere/Metal3/自研）  

都会检查 paused 状态。

### ❌ kubeadm 不知道 paused  
### ❌ kubelet 不知道 paused  
### ❌ Kubernetes scheduler 不知道 paused  
### ❌ cloud-init 不知道 paused  

Paused 只影响 **CAPI 控制器的 reconcile loop**。

## 4. Paused 的内部机制（控制器如何处理 paused）

每个 CAPI 控制器 reconcile 的第一步都是：
```go
if annotations["cluster.x-k8s.io/paused"] == "true" {
    return ctrl.Result{}, nil
}
```

也就是：
> **直接退出，不做任何调谐。**

不会：
- 创建/删除 Machine  
- 生成 bootstrap data  
- 调用 Infra Provider  
- 滚动升级  
- 修复节点  
- 更新状态  

## 5. Paused 的作用范围（非常关键）

| 资源 | 是否受 Paused 影响 |
|------|---------------------|
| **Cluster** | ✔ 是 |
| **KubeadmControlPlane** | ✔ 是 |
| **MachineDeployment** | ✔ 是 |
| **MachineSet** | ✔ 是 |
| **Machine** | ✔ 是 |
| **Bootstrap provider（CABPK）** | ✔ 是 |
| **Infra provider（AWS/vSphere/Metal3）** | ✔ 是 |
| **kubeadm** | ✘ 否 |
| **kubelet** | ✘ 否 |
| **Pod 调度** | ✘ 否 |
| **节点运行状态** | ✘ 否 |

## 6. Paused 的典型使用场景

## ✔ 1. 手动维护节点  
避免 CAPI 自动替换节点。

## ✔ 2. 调试 bootstrap 问题  
避免控制器不断重试。

## ✔ 3. 升级 CAPI 版本  
避免升级过程中触发 reconcile。

## ✔ 4. 迁移集群（clusterctl move）  
必须先 paused。

## ✔ 5. 避免误操作  
例如错误的 MachineDeployment 副本数。

## 7. Paused 的最佳实践

### ✔ 在执行 clusterctl move 前必须 paused  
否则源集群和目标集群会同时调谐，导致灾难。

### ✔ 在调试 bootstrap 失败时先 paused  
避免控制器不断创建/删除 Machine。

### ✔ 在手动修复基础设施时 paused  
例如：
- 修复裸金属节点  
- 修复 vSphere VM  
- 修复网络问题  

## 8. 一句话总结（再次强调）

> **Paused 是 Cluster API 的“全局调谐暂停开关”，用于阻止 CAPI 控制器对资源进行任何自动修改。它由用户设置，所有 CAPI 控制器都会检查它。**

告诉我你想继续哪个方向，我可以继续深入到控制器级别的细节。
