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
