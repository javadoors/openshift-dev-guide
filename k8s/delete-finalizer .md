# k8s中CR资源删除的逻辑
在 Kubernetes 中，**CR（Custom Resource）资源的删除逻辑**是由 API Server、垃圾回收机制（GC）、以及 CRD 控制器共同完成的。整体流程如下：
## 📌 删除流程逻辑
1. **用户请求删除**
   - 用户执行 `kubectl delete <CR>`，API Server 接收到删除请求。
2. **Finalizers 检查**
   - 如果该 CR 对象的 `metadata.finalizers` 字段非空，API Server不会立即删除对象，而是将其标记为 **“deletionTimestamp”**。  
   - 控制器需要在看到这个标记后执行清理逻辑（例如释放外部资源），完成后移除 finalizer。
3. **控制器清理**
   - CRD 对应的控制器（Operator）检测到对象进入删除流程。  
   - 控制器执行自定义清理逻辑（如删除云资源、关闭服务）。  
   - 完成后移除 finalizer。
4. **对象真正删除**
   - 当 finalizer 被移除，API Server 才会真正删除该对象。  
   - 删除后，etcd 中的存储记录被清理。
5. **级联删除（Cascading Deletion）**
   - 如果 CR 对象被其他资源引用（OwnerReference），垃圾回收器会根据 `propagationPolicy` 决定是否级联删除子资源。  
     - **Foreground**：先删除子资源，再删除父对象。  
     - **Background**：父对象立即删除，子资源后台异步删除。  
     - **Orphan**：仅删除父对象，子资源保留。
## 📊 关键点总结
| 阶段 | 逻辑 |
|------|------|
| 用户请求 | `kubectl delete` 发起删除 |
| Finalizer | 阻塞删除，等待控制器清理外部资源 |
| 控制器清理 | Operator 执行自定义清理逻辑 |
| 真正删除 | Finalizer 移除后，API Server 删除对象 |
| 垃圾回收 | 根据 OwnerReference 和策略决定子资源是否删除 |

## 🔑 设计价值
- **安全性**：通过 finalizer 确保外部资源不会被遗留。  
- **一致性**：保证 CR 删除时，相关依赖资源也能正确清理。  
- **灵活性**：支持不同的级联策略，满足不同场景需求。  

👉 总结：Kubernetes 中 CR 的删除逻辑是 **用户请求 → Finalizer 阻塞 → 控制器清理 → Finalizer 移除 → API Server 真正删除 → 垃圾回收**。  

# 时序图
直观展示 Kubernetes 中 CR 删除过程里 **API Server、Finalizer、控制器、垃圾回收器**之间的交互逻辑：
```mermaid
sequenceDiagram
    participant User as 用户
    participant APIServer as API Server
    participant Finalizer as Finalizer
    participant Controller as 控制器-Operator
    participant GC as 垃圾回收器-Garbage Collector

    User->>APIServer: 发起删除 CR 请求-kubectl delete
    APIServer->>APIServer: 设置 deletionTimestamp
    APIServer->>Finalizer: 检查 Finalizers 列表
    alt 存在 Finalizer
        APIServer->>Controller: 通知对象进入删除流程
        Controller->>Controller: 执行清理逻辑-外部资源释放
        Controller->>Finalizer: 移除 Finalizer
        Finalizer->>APIServer: Finalizer 清空
    end

    APIServer->>APIServer: 真正删除对象-从 etcd 移除
    APIServer->>GC: 根据 OwnerReference 触发级联删除
    GC->>GC: 按策略删除子资源-Foreground/Background/Orphan
```
### 🔑 图解说明
- **用户请求**：通过 `kubectl delete` 发起删除。  
- **API Server**：设置 `deletionTimestamp`，检查是否有 Finalizer。  
- **Finalizer**：阻塞删除，等待控制器完成清理。  
- **控制器**：执行自定义清理逻辑（如释放云资源），移除 Finalizer。  
- **垃圾回收器**：根据 OwnerReference 和策略决定是否级联删除子资源。  
### 📊 总结
- 删除流程是 **用户请求 → Finalizer 阻塞 → 控制器清理 → Finalizer 移除 → API Server 真正删除 → 垃圾回收器级联处理**。  
- **安装与升级的依赖顺序一致性**和 **CR 删除逻辑**一样，都是围绕依赖链和控制器驱动的声明式流程。  

# 检查 Finalizers 列表,是如何找到控制器的？
在 Kubernetes 中，**Finalizers 并不会直接“找到”控制器**，而是通过一种约定机制让控制器自己来发现和处理。逻辑如下：
## 📌 Finalizer 的工作机制
- **Finalizer 是一个字符串标识**：例如 `"example.com/finalizer"`。  
- **API Server 行为**：当对象带有 Finalizer 时，API Server 在删除时只会设置 `deletionTimestamp`，不会真正删除对象。  
- **控制器发现**：控制器在监听该 CR 时，会看到对象进入删除流程（`deletionTimestamp` 已设置）。  
- **匹配逻辑**：控制器代码里通常会检查对象的 `metadata.finalizers` 列表，如果包含自己负责的标识（如 `"example.com/finalizer"`），就执行清理逻辑。  
- **完成清理**：控制器清理完外部资源后，主动移除该 Finalizer。  
- **最终删除**：当所有 Finalizer 都被移除，API Server 才真正删除对象。
## 🔑 关键点
- Finalizer 本质上是一个 **约定字符串**，不是 API Server 去“找”控制器，而是控制器自己通过监听和匹配来发现。  
- 控制器必须事先在代码里声明并处理对应的 Finalizer 名称。  
- 这种机制保证了删除流程的可扩展性：不同控制器可以通过不同 Finalizer 名称来实现自定义清理逻辑。  
## 📊 总结
- **Finalizer 是标识，不是指针**。  
- **控制器通过监听 CR 并匹配 Finalizer 名称来触发清理逻辑**。  
- **API Server 只负责阻塞删除，真正的清理由控制器完成**。  
## 时序图
直观展示对象删除时 API Server 设置 `deletionTimestamp`，控制器发现 Finalizer 并执行清理，最后移除 Finalizer 的完整交互过程：
```mermaid
sequenceDiagram
    participant User as 用户
    participant APIServer as API Server
    participant Controller as 控制器-Operator
    participant Finalizer as Finalizer 标识

    User->>APIServer: kubectl delete CR
    APIServer->>APIServer: 设置 deletionTimestamp
    APIServer->>Finalizer: 检查 Finalizers 列表

    alt 存在 Finalizer
        APIServer->>Controller: 通知对象进入删除流程
        Controller->>Controller: 执行清理逻辑-释放外部资源
        Controller->>Finalizer: 移除 Finalizer 标识
        Finalizer->>APIServer: Finalizer 列表为空
    end

    APIServer->>APIServer: 真正删除对象-从 etcd 移除
```
### 🔑 图解说明
- **用户请求**：通过 `kubectl delete` 发起删除。  
- **API Server**：设置 `deletionTimestamp`，检查对象是否有 Finalizer。  
- **Finalizer**：如果存在，阻塞删除。  
- **控制器**：监听到对象进入删除流程，执行清理逻辑，完成后移除 Finalizer。  
- **最终删除**：当 Finalizer 列表为空时，API Server 才真正删除对象。  

👉 总结：**Finalizer 是一种约定机制**，API Server 不会直接“找到”控制器，而是由控制器监听 CR 并匹配 Finalizer 名称，主动执行清理逻辑并移除 Finalizer，最终由 API Server 完成删除。  

# Finalizer

在 Kubernetes / OpenShift 中，**Finalizer** 是一种保护机制，可以防止资源在删除时被立即清理，从而让控制器有机会执行必要的清理逻辑。你提到的“添加 Finalizer 保护集群不被误删”，通常是指在关键资源（例如 `Cluster`、`Namespace`、`CustomResource`）上加上 Finalizer，确保删除操作必须经过控制器的处理，避免误删导致集群不可恢复。  

## 📖 Finalizer 的作用

- **阻止直接删除**：当资源带有 Finalizer 时，用户执行 `kubectl delete` 并不会立即删除，而是进入 “Terminating” 状态。  
- **执行清理逻辑**：控制器必须移除 Finalizer 才能真正删除资源，这期间可以执行备份、资源回收、日志记录等操作。  
- **防止误操作**：避免管理员或自动化脚本一键删除关键资源，提供额外的保护层。  

## 🛠️ 添加 Finalizer 示例

假设你要保护一个自定义资源 `Cluster`，可以在 YAML 中添加 Finalizer：

```yaml
apiVersion: cluster.example.com/v1
kind: Cluster
metadata:
  name: demo-cluster
  finalizers:
    - protect.cluster.finalizer
```

这样，当有人执行：
```bash
kubectl delete cluster demo-cluster
```
资源不会立即消失，而是进入 **Terminating** 状态，直到控制器移除 `protect.cluster.finalizer`。  

## 📊 使用场景

| **场景** | **Finalizer作用** |
|----------|-------------------|
| 删除集群 | 确保先清理节点、存储、网络，再允许删除 |
| 删除命名空间 | 确保所有子资源（Pod、Service、PVC）安全清理 |
| 删除 CRD | 确保相关 Operator 执行回收逻辑 |
| 删除关键资源 | 防止误删，要求额外确认或自动备份 |

## ✅ 总结
在 OpenShift 中，**Finalizer 是保护集群和关键资源的有效机制**。通过在资源上添加 Finalizer，可以避免误删，确保删除前执行必要的清理或确认步骤。  

# Finalizer 最佳实践清单

在 Kubernetes / OpenShift 集群中合理使用 Finalizer 来保护关键资源，避免误删或清理不完整：  

## 📖 Finalizer 最佳实践清单

- **命名规范**  
  - 使用清晰的命名空间前缀，例如：`protect.cluster.finalizer` 或 `cleanup.namespace.finalizer`。  
  - 避免过于通用的名字，确保能明确区分不同控制器的职责。  

- **关键资源保护**  
  - 在 `Cluster`、`Namespace`、`CRD` 等关键资源上添加 Finalizer，防止误删导致不可恢复的后果。  
  - 删除时必须由控制器移除 Finalizer，确保执行清理逻辑。  

- **清理逻辑完整性**  
  - 在删除前执行必要的操作：释放存储卷、回收网络资源、注销外部服务。  
  - 确保 Finalizer 的清理逻辑幂等（可重复执行而不会出错）。  

- **避免资源卡死**  
  - 控制器必须在清理完成后移除 Finalizer，否则资源会一直停留在 `Terminating` 状态。  
  - 建议在控制器中加入超时机制，防止无限阻塞。  

- **审计与安全**  
  - 在 Finalizer 清理逻辑中记录审计日志，便于追踪删除操作。  
  - 对关键资源的 Finalizer 删除操作进行 RBAC 限制，避免普通用户绕过保护。  

- **测试与演练**  
  - 在测试环境中演练 Finalizer 的删除流程，确保逻辑正确。  
  - 定期检查集群中是否存在“卡死”的资源，避免遗留问题。  

## 📊 使用场景示例

| **场景** | **Finalizer作用** |
|----------|-------------------|
| 删除集群 | 确保先清理节点、存储、网络，再允许删除 |
| 删除命名空间 | 确保所有子资源安全清理，避免孤儿资源 |
| 删除 CRD | 确保 Operator 执行回收逻辑 |
| 删除 PVC | 确保存储卷在外部存储系统中正确释放 |

## ✅ 总结
Finalizer 是一种 **防误删保护机制**，最佳实践包括：规范命名、保护关键资源、保证清理逻辑完整、避免卡死、加强审计与安全，并在测试环境中演练。这样可以确保集群在删除操作时安全、可控。  

