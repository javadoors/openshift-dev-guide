# OpenShift CVO 的 Operator 升级机制
## OpenShift CVO 的 Operator 升级机制
**不是简单的 `kubectl apply`**，CVO 采用了更精密的声明式协调机制。核心流程如下：
### 1. CVO 的整体架构
```
┌─────────────────────────────────────────────────────────┐
│                    Cluster Version Operator               │
│                                                         │
│  ClusterVersion CR (用户期望版本)                         │
│       │                                                 │
│       ▼                                                 │
│  Release Image (目标版本的容器镜像)                        │
│       │                                                 │
│       ▼                                                 │
│  内嵌的 release-manifests                                │
│    (CVO 从 Release Image 中提取)                         │
│       │                                                 │
│       ▼                                                 │
│  Manifests 按组件拆分 → 各 Operator 的 YAML 清单          │
│       │                                                 │
│       ▼                                                 │
│  CVO 协调器 (逐个 Apply 到集群)                           │
└─────────────────────────────────────────────────────────┘
```
### 2. Release Image — 版本的载体
OpenShift 的版本不是通过单独的 YAML 文件分发，而是打包在 **Release Image** 中：
```
quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64
```
Release Image 内部结构：
```
/
├── release-manifests/          # 所有组件的 YAML 清单
│   ├── 0000_50_cluster-autoscaler_00-deployment.yaml
│   ├── 0000_50_cluster-autoscaler_01-configmap.yaml
│   ├── 0000_50_ingress-operator_00-namespace.yaml
│   ├── 0000_50_ingress-operator_01-deployment.yaml
│   ├── ... (数百个清单文件)
│   └── image-references        # 各组件镜像版本映射
└── sha256:...                  # 各组件的镜像层
```
### 3. CVO 的升级流程（非简单 kubectl apply）
```
用户更新 ClusterVersion CR
  │  (spec.desiredUpdate.version: "4.15.0")
  │
  ▼
CVO 拉取目标版本的 Release Image
  │
  ▼
从 Release Image 提取 release-manifests
  │
  ▼
解析 image-references 获取各组件镜像版本
  │
  ▼
按优先级排序清单文件（文件名前缀数字决定顺序）
  │  0000_ → 先执行（CRD、Namespace等基础资源）
  │  50_   → 中间执行（Operator 部署）
  │  99_   → 最后执行（状态检查等）
  │
  ▼
逐个组件协调（不是一次性 apply 所有）
  │
  ├── 1. 计算 Diff：对比当前集群状态 vs 目标清单
  ├── 2. 条件更新：只更新有变化的资源
  ├── 3. 等待就绪：等待当前组件健康后才继续下一个
  ├── 4. 状态回写：更新 ClusterVersion.Status
  │
  ▼
升级完成
```
### 4. CVO vs kubectl apply 的关键区别
| 维度 | `kubectl apply` | CVO 协调 |
|------|----------------|---------|
| **执行方式** | 一次性提交所有清单 | **逐组件串行协调**，等待健康检查 |
| **版本来源** | 本地 YAML 文件 | **Release Image 内嵌清单** |
| **失败处理** | 报错停止，需手动恢复 | **自动暂停**，保留已完成状态，可回滚 |
| **状态追踪** | 无 | **ClusterVersion.Status** 记录每个组件的升级状态 |
| **顺序控制** | 无保证 | **文件名前缀排序**（0000→50→99） |
| **幂等性** | 依赖 `last-applied-configuration` annotation | **基于 owned annotations** 的声明式管理 |
| **所有权** | 任何人都可修改 | CVO 通过 annotation 标记**所有权**，防止误改 |
### 5. CVO 的资源所有权标记
CVO 不会简单 `kubectl apply`，它通过 annotation 标记自己管理的资源：
```yaml
metadata:
  annotations:
    olm.operatorGroup: openshift-cluster-version
    release.openshift.io/create-only: "true"     # 只创建不更新
    # 或
    release.openshift.io/owner: cluster-version-operator  # CVO 管理
  labels:
    app: cluster-version-operator
    owning-component: "ingress-operator"          # 归属组件
```
CVO 协调时会检查这些标记，决定：
- **create-only** 资源：只在不存在时创建，不更新
- **owned 资源**：CVO 完全管理，外部修改会被覆盖
- **unmanaged 资源**：CVO 不管理，用户可自由修改
### 6. 各 Operator 的自升级机制
OpenShift 的 Operator 升级分为两层：
```
┌─────────────────────────────────────────────────────────┐
│  第一层：CVO 升级 Operator 的 Deployment                  │
│                                                         │
│  CVO 直接管理各 Operator 的 Deployment 清单               │
│  更新 Deployment 中的镜像 tag → 触发滚动更新              │
│  等待新 Pod 就绪 → 标记该 Operator 升级完成               │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  第二层：各 Operator 协调自己的 Operand                   │
│                                                         │
│  Operator 启动后，根据自身逻辑协调其管理的资源             │
│  例如：ingress-operator 协调 HAProxy Ingress Controller  │
│  例如：monitoring-operator 协调 Prometheus/Alertmanager  │
└─────────────────────────────────────────────────────────┘
```
### 7. 与当前 BKE 项目的对比
| 维度 | OpenShift CVO | BKE 当前方案 |
|------|--------------|------------|
| **版本载体** | Release Image（内嵌清单） | VersionConfig YAML（ConfigMap） |
| **升级触发** | 更新 ClusterVersion CR | 更新 BKECluster.Spec 中的 openFuyaoVersion |
| **清单分发** | 从 Release Image 提取 | 从 ConfigMap 读取 |
| **组件升级** | CVO 直接管理 Deployment 镜像 | ensure_component_upgrade 更新 Pod 对应的 Controller 镜像 |
| **升级顺序** | 文件名前缀排序 | Phase 顺序 + Addon 顺序 |
| **状态追踪** | ClusterVersion.Status 条件 | BKECluster.Status 条件 |
| **回滚能力** | 支持（通过 Release Image 切换） | 有限（依赖镜像回退） |
### 总结
OpenShift CVO 的 Operator 升级**不是简单的 `kubectl apply`**，而是一个**声明式协调系统**：
1. **版本打包在 Release Image 中**，而非独立 YAML 文件
2. **逐组件串行升级**，每步等待健康检查
3. **通过 annotation 管理资源所有权**，防止冲突
4. **两层升级架构**：CVO 升级 Operator → Operator 升级 Operand
5. **完整的状态追踪和回滚机制**

这种设计思路对 BKE 重构有参考价值：将 VersionConfig 作为版本清单的权威来源，通过 ConfigMap 传递，controller 基于声明式协调逐组件升级，而非一次性 apply 所有清单。
        
