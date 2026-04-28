# 驱逐节点上的 Pod
在 Kubernetes 中，**驱逐节点上的 Pod 并没有单独的“驱逐节点 API”，而是通过 Pod 的 Eviction API 来实现**。这也是 `kubectl drain` 命令背后的机制。  

## 📑 驱逐相关 API
- **Eviction API**  
  - 路径：  
    ```
    POST /api/v1/namespaces/{namespace}/pods/{pod}/eviction
    ```
  - 版本：`policy/v1`（Kubernetes v1.22+）  
  - 请求体示例：  
    ```yaml
    apiVersion: policy/v1
    kind: Eviction
    metadata:
      name: mypod
      namespace: default
    ```
  - 效果：触发 Pod 的优雅终止，遵守 **PodDisruptionBudget (PDB)** 和 `terminationGracePeriodSeconds`。  

- **DELETE Pod API**  
  - 路径：  
    ```
    DELETE /api/v1/namespaces/{namespace}/pods/{pod}
    ```
  - 与 Eviction API 的区别：直接删除 Pod，不考虑 PDB，属于强制操作。  

## ⚙️ 对应命令
- **驱逐节点上的 Pod（推荐方式）**  
  ```bash
  kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
  ```
  - 背后调用的是 Eviction API。  
  - 会自动跳过 DaemonSet Pod 和静态 Pod。  
  - 遵守 PodDisruptionBudget，确保不会破坏应用的高可用性。  

- **强制删除 Pod（不推荐）**  
  ```bash
  kubectl delete pod <pod-name> -n <namespace>
  ```
  - 直接调用 Pod DELETE API，不考虑 PDB。  

## 📊 总结
- 驱逐节点的 API 本质是 **Pod 的 Eviction API**。  
- 推荐使用 `kubectl drain`，它会批量调用 Eviction API，安全驱逐节点上的 Pod。  
- 如果直接 `kubectl delete pod`，则绕过 PDB，风险更高。  
