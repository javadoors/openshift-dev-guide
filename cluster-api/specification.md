# Controller 实现规范性分析
## 一、Cluster API Provider 开发规范要求
根据 Cluster API 官方规范，Infrastructure Provider 需要满足以下核心要求：
### 1. CRD 定义规范
| 规范项 | 要求 | 说明 |
|--------|------|------|
| **ControlPlaneEndpoint** | 必须在 Spec 中定义 | 用于访问 Kubernetes API Server 的端点 |
| **Ready 字段** | 必须在 Status 中定义 | 标识基础设施是否就绪 |
| **FailureDomains** | 应该在 Status 中定义 | 支持多可用区部署 |
| **Conditions** | 应该使用 clusterv1.Conditions | 标准化的状态条件 |
| **OwnerReference** | 必须正确设置 | BKECluster 应该被 Cluster 对象拥有 |
### 2. Controller 实现规范
| 规范项 | 要求 | 说明 |
|--------|------|------|
| **Reconcile 逻辑** | 幂等、可重入 | 支持多次执行不产生副作用 |
| **Owner Cluster 获取** | 使用 util.GetOwnerCluster | 正确获取父 Cluster 对象 |
| **Finalizer 处理** | 必须实现 | 支持资源清理 |
| **Patch Helper** | 推荐使用 | 优化状态更新 |
| **Conditions 更新** | 使用 conditions 包 | 标准化状态管理 |
