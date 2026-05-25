# Cluster-API 基于 ClusterClass 与非 ClusterClass 模式开发指南

## 一、两种模式对比
| 维度 | ClusterClass 模式 (Managed Topology) | 非 ClusterClass 模式 (Classic) |
|------|--------------------------------------|-------------------------------|
| **声明方式** | 声明式模板 + 变量覆盖 | 手动编写每个资源 YAML |
| **资源创建** | Topology Controller 自动生成 | 用户手动创建 |
| **升级方式** | 修改 `spec.topology.version` | 逐个修改 MachineDeployment/KCP |
| **扩展机制** | Runtime Extensions (GeneratePatches) | 直接修改资源配置 |
| **适用场景** | 多集群标准化、大规模管理 | 小规模、高度定制化 |
| **学习曲线** | 较高 (需理解 ClusterClass/Patch) | 较低 (直接操作资源) |

## 二、非 ClusterClass 模式开发指南

### 2.1 架构模型
```
用户手动创建所有资源
         │
         ├── Cluster
         ├── InfrastructureCluster (如 BareMetalCluster)
         ├── KubeadmControlPlane
         ├── InfrastructureMachineTemplate
         ├── KubeadmConfigTemplate
         └── MachineDeployment
```

### 2.2 完整示例
```yaml
# 1. Cluster - 集群定义
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["10.244.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneEndpoint:
    host: "lb.example.com"
    port: 6443
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: BareMetalCluster
    name: my-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: my-cluster-cp
---
# 2. InfrastructureCluster - 基础设施集群
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalCluster
metadata:
  name: my-cluster
  namespace: default
spec:
  controlPlaneEndpoint:
    host: "lb.example.com"
    port: 6443
---
# 3. KubeadmControlPlane - 控制面
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: my-cluster-cp
  namespace: default
spec:
  replicas: 3
  version: v1.31.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalMachineTemplate
      name: my-cluster-cp-template
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          - name: "audit-log-path"
            value: "/var/log/kubernetes/audit.log"
      etcd:
        local:
          dataDir: /var/lib/etcd
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          - name: "max-pods"
            value: "250"
---
# 4. InfrastructureMachineTemplate - 控制面机器模板
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalMachineTemplate
metadata:
  name: my-cluster-cp-template
  namespace: default
spec:
  template:
    spec:
      hostName: "cp-{{ .machine.name }}"
      credentialsRef:
        name: cp-credentials
---
# 5. MachineDeployment - 工作节点
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  name: my-cluster-md-0
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  template:
    spec:
      clusterName: my-cluster
      version: v1.31.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
          kind: KubeadmConfigTemplate
          name: my-cluster-md-template
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: my-cluster-md-template
---
# 6. KubeadmConfigTemplate - 工作节点引导配置
apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
kind: KubeadmConfigTemplate
metadata:
  name: my-cluster-md-template
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            - name: "max-pods"
              value: "500"
---
# 7. InfrastructureMachineTemplate - 工作节点机器模板
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalMachineTemplate
metadata:
  name: my-cluster-md-template
  namespace: default
spec:
  template:
    spec:
      hostName: "worker-{{ .machine.name }}"
      credentialsRef:
        name: worker-credentials
```

### 2.3 升级操作
```bash
# 升级控制面
kubectl patch kubeadmcontrolplane my-cluster-cp --type merge -p '{"spec":{"version":"v1.32.0"}}'

# 升级工作节点
kubectl patch machinedeployment my-cluster-md-0 --type merge -p '{"spec":{"template":{"spec":{"version":"v1.32.0"}}}}'
```

### 2.4 优缺点
| 优点 | 缺点 |
|------|------|
| 简单直接，易于理解 | 多集群管理时 YAML 重复严重 |
| 完全控制每个资源 | 升级需手动操作多个资源 |
| 适合小规模/实验环境 | 无法使用 Runtime Extensions |
| 无需学习 ClusterClass 概念 | 配置变更难以标准化 |

## 三、ClusterClass 模式开发指南

### 3.1 架构模型
```
用户定义 Cluster + Topology
         │
         ▼
   Topology Controller
         │
         ├── 读取 ClusterClass 模板
         ├── 应用 Patches (GeneratePatches Hook)
         ├── 注入 Variables
         └── 自动生成所有资源
                  │
                  ├── Cluster
                  ├── InfrastructureCluster
                  ├── KubeadmControlPlane
                  ├── MachineDeployment(s)
                  └── 所有关联资源
```

### 3.2 ClusterClass 定义
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: my-cluster-class
  namespace: default
spec:
  # Kubernetes 版本列表
  kubernetesVersions:
    - v1.30.0
    - v1.31.0
    - v1.32.0

  # 基础设施集群模板
  infrastructure:
    templateRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: BareMetalClusterTemplate
      name: my-cluster-class-infra

  # 控制面模板
  controlPlane:
    templateRef:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta2
      kind: KubeadmControlPlaneTemplate
      name: my-cluster-class-cp
    machineInfrastructure:
      templateRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: my-cluster-class-cp-machine

  # Worker 节点类
  workers:
    machineDeployments:
      - class: default-workers
        template:
          bootstrap:
            templateRef:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
              kind: KubeadmConfigTemplate
              name: my-cluster-class-md-bootstrap
          infrastructure:
            templateRef:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
              kind: BareMetalMachineTemplate
              name: my-cluster-class-md-machine

  # 变量定义
  variables:
    - name: registryEndpoint
      required: true
      schema:
        openAPIV3Schema:
          type: string
          description: "镜像仓库地址"
    - name: ntpServers
      required: false
      schema:
        openAPIV3Schema:
          type: array
          items:
            type: string
    - name: kubeletMaxPods
      required: false
      schema:
        openAPIV3Schema:
          type: integer
          default: 250

  # Patches - 将变量注入到模板中
  patches:
    - name: registry
      definitions:
        - selector:
            apiVersion: controlplane.cluster.x-k8s.io/v1beta2
            kind: KubeadmControlPlaneTemplate
            matchResources:
              controlPlane: true
          jsonPatches:
            - op: add
              path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/imageRepository
              valueFrom:
                variable: registryEndpoint
    - name: kubelet-config
      definitions:
        - selector:
            apiVersion: controlplane.cluster.x-k8s.io/v1beta2
            kind: KubeadmControlPlaneTemplate
            matchResources:
              controlPlane: true
          jsonPatches:
            - op: add
              path: /spec/template/spec/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs
              valueFrom:
                template: |
                  - name: max-pods
                    value: "{{ .kubeletMaxPods }}"
```

### 3.3 Cluster 定义 (使用 ClusterClass)
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["10.244.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  topology:
    classRef:
      name: my-cluster-class
    version: v1.31.0
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: default-workers
          name: md-0
          replicas: 5
    variables:
      - name: registryEndpoint
        value: "registry.example.com/k8s"
      - name: ntpServers
        value:
          - "ntp1.example.com"
          - "ntp2.example.com"
      - name: kubeletMaxPods
        value: 300
```

### 3.4 升级操作
```bash
# 只需修改一个字段 - 整个集群自动升级
kubectl patch cluster my-cluster --type merge -p '{"spec":{"topology":{"version":"v1.32.0"}}}'

# 修改控制面副本数
kubectl patch cluster my-cluster --type merge -p '{"spec":{"topology":{"controlPlane":{"replicas":5}}}}'

# 修改变量 (触发 Patch 重新计算)
kubectl patch cluster my-cluster --type merge -p '{"spec":{"topology":{"variables":[{"name":"kubeletMaxPods","value":500}]}}}'
```

### 3.5 Runtime Extensions 集成
```yaml
# ExtensionConfig - 注册扩展服务
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: my-extension
spec:
  clientConfig:
    service:
      name: my-extension-service
      namespace: capi-system
      port: 443

# ClusterClass 中使用外部 Patch
  patches:
    - name: external-patch
      external:
        generateExtension: my-extension/generate-patches
```

### 3.6 优缺点
| 优点 | 缺点 |
|------|------|
| 多集群标准化 | 学习曲线较陡 |
| 一键升级整个集群 | 调试 Patch 较复杂 |
| 变量覆盖实现定制化 | 初始配置工作量大 |
| 支持 Runtime Extensions | 需要理解拓扑计算逻辑 |
| 配置变更可审计 | 不适合高度差异化集群 |

## 四、模式选择决策树
```
是否需要管理多个相似集群？
    ├── 是 → ClusterClass 模式
    │         │
    │         ├── 需要动态注入配置？
    │         │   ├── 是 → 使用 Variables + Patches
    │         │   └── 否 → 仅使用模板引用
    │         │
    │         └── 需要复杂逻辑？
    │             ├── 是 → 使用 Runtime Extensions
    │             └── 否 → 使用内置 Patches
    │
    └── 否 → 非 ClusterClass 模式
              │
              ├── 是否只需要少量集群？
              │   ├── 是 (1-3个) → 直接编写 YAML
              │   └── 否 → 考虑 ClusterClass
              │
              └── 是否需要高度定制？
                  ├── 是 → 非 ClusterClass 模式
                  └── 否 → 考虑 ClusterClass
```

## 五、混合模式
可以在同一个管理集群中同时使用两种模式：
```yaml
# 集群 A - 使用 ClusterClass
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: cluster-a
spec:
  topology:
    classRef:
      name: production-class
    version: v1.31.0

# 集群 B - 不使用 ClusterClass
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: cluster-b
spec:
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: cluster-b-cp
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: BareMetalCluster
    name: cluster-b
```

## 六、最佳实践
| 场景 | 推荐模式 | 理由 |
|------|----------|------|
| 生产环境多集群 | ClusterClass | 标准化、易升级 |
| 测试/开发环境 | 非 ClusterClass | 快速搭建、灵活 |

| 边缘计算节点 | ClusterClass | 批量管理 |
| 特殊定制集群 | 非 ClusterClass | 完全控制 |
| CI/CD 集成 | ClusterClass | 声明式、可版本化 |
