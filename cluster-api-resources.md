      
# Cluster API 核心资源规格、设计思路及使用样例
## 1. Cluster - 集群蓝图
### 1.1 设计思路
**类比：Deployment**

**核心思想：**
- Cluster是Cluster API的顶层资源，定义了整个Kubernetes集群的期望状态
- 采用组合模式，通过引用ControlPlane和Infrastructure资源来构建完整集群
- 支持两种模式：传统模式（显式引用）和拓扑模式（使用ClusterClass）

**解决的问题：**
1. **统一管理入口**：一个资源管理整个集群生命周期
2. **解耦关注点**：将控制平面、基础设施、工作节点分离管理
3. **声明式配置**：用户只需声明期望状态，无需关心实现细节
4. **可复用性**：通过ClusterClass实现集群模板复用

**架构关系：**
```
Cluster (顶层)
├── ControlPlaneRef → ControlPlane (控制平面)
├── InfrastructureRef → InfrastructureCluster (基础设施)
└── Topology → ClusterClass (可选，模板化部署)
```
### 1.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: <cluster-name>
  namespace: <namespace>
spec:
  # 暂停集群协调
  paused: false
  
  # 集群网络配置
  clusterNetwork:
    pods:
      cidrBlocks: ["10.128.0.0/14"]
    services:
      cidrBlocks: ["172.30.0.0/16"]
    serviceDomain: "cluster.local"
  
  # 控制平面引用（传统模式）
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: <control-plane-name>
  
  # 基础设施引用（传统模式）
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: <infrastructure-name>
  
  # 拓扑模式（使用ClusterClass）
  topology:
    class: <cluster-class-name>
    version: v1.28.0
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: default-worker
          name: md-0
          replicas: 3
    variables:
      - name: region
        value: us-west-2

status:
  # 集群阶段
  phase: Provisioned|Provisioning|Failed|Deleting
  
  # 条件状态
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-01T00:00:00Z"
  
  # 基础设施就绪状态
  infrastructureReady: true
  controlPlaneReady: true
```
### 1.3 使用样例
#### 样例1：传统模式 - 完整集群定义
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: default
  labels:
    environment: production
    region: us-west-2
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["10.128.0.0/14"]
    services:
      cidrBlocks: ["172.30.0.0/16"]
    serviceDomain: "cluster.local"
  
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: prod-control-plane
  
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: prod-infra
```
**设计说明：**
- 显式引用ControlPlane和Infrastructure资源
- 适合需要精细控制每个组件的场景
- 每个组件可以独立管理和升级
#### 样例2：拓扑模式 - 使用ClusterClass
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: dev-cluster
  namespace: development
spec:
  topology:
    class: aws-quick-start
    version: v1.28.0
    controlPlane:
      replicas: 3
      metadata:
        labels:
          tier: control-plane
    workers:
      machineDeployments:
        - class: default-worker
          name: md-general
          replicas: 5
          metadata:
            labels:
              node-pool: general
        - class: gpu-worker
          name: md-gpu
          replicas: 2
          metadata:
            labels:
              node-pool: gpu
    variables:
      - name: region
        value: us-west-2
      - name: instanceType
        value: m5.large
      - name: sshKeyName
        value: dev-key
```
**设计说明：**
- 使用ClusterClass作为模板
- 通过variables定制集群配置
- 简化集群创建流程，适合批量管理
## 2. ControlPlane - 管理控制平面节点
### 2.1 设计思路
**类比：StatefulSet**

**核心思想：**
- 管理控制平面节点的生命周期
- 确保控制平面组件的有序部署和升级
- 提供高可用性保证（多副本、滚动升级）

**解决的问题：**
1. **有序管理**：控制平面节点需要有序启动和升级
2. **高可用性**：支持多副本部署，自动故障恢复
3. **版本管理**：支持Kubernetes版本升级
4. **配置一致性**：确保所有控制平面节点配置一致

**关键设计：**
- 使用RollingUpdate策略进行升级
- 支持etcd集群管理
- 集成证书管理
- 支持自定义Kubernetes组件配置
### 2.2 资源规格
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: <name>
  namespace: <namespace>
spec:
  # 副本数
  replicas: 3
  
  # Kubernetes版本
  version: v1.28.0
  
  # 机器模板
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: <template-name>
    nodeDrainTimeout: 5m
  
  # Kubeadm配置
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs: {}
        certSANs: []
      controllerManager:
        extraArgs: {}
      etcd:
        local:
          dataDir: /var/lib/etcd
    initConfiguration:
      nodeRegistration: {}
    joinConfiguration:
      nodeRegistration: {}
  
  # 滚动升级策略
  rolloutStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1

status:
  ready: true
  initialized: true
  apiServerReady: true
  replicas: 3
  updatedReplicas: 3
  readyReplicas: 3
```
### 2.3 使用样例
#### 样例1：AWS高可用控制平面
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: prod-control-plane
  namespace: default
spec:
  replicas: 3
  version: v1.28.0
  
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: prod-control-plane-template
    nodeDrainTimeout: 10m
  
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
          audit-log-path: /var/log/kubernetes/audit.log
          audit-log-maxage: "30"
        certSANs:
          - prod-api.example.com
          - 192.168.1.100
      controllerManager:
        extraArgs:
          cloud-provider: aws
          cluster-signing-duration: "8760h"
      etcd:
        local:
          dataDir: /var/lib/etcd
          extraArgs:
            snapshot-count: "10000"
            heartbeat-interval: "100"
            election-timeout: "1000"
    initConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
        criSocket: /var/run/containerd/containerd.sock
        kubeletExtraArgs:
          cloud-provider: aws
    joinConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
        criSocket: /var/run/containerd/containerd.sock
        kubeletExtraArgs:
          cloud-provider: aws
  
  rolloutStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
```
**设计说明：**
- 3副本保证高可用
- etcd优化配置提升性能
- 审计日志配置增强安全
- 滚动升级策略确保服务不中断
#### 样例2：BKE控制平面
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: BKEControlPlane
metadata:
  name: bke-control-plane
  namespace: cluster-system
spec:
  replicas: 3
  version: v1.28.0
  
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.bke.bocloud.com/v1beta1
      kind: BKEMachineTemplate
      name: bke-control-plane-template
  
  bkeConfigSpec:
    clusterConfig:
      containerRuntime:
        cri: containerd
        runtime: runc
      apiServer:
        extraArgs:
          enable-admission-plugins: NodeRestriction,PodSecurityPolicy
        certSANs:
          - api.bke.example.com
      etcd:
        dataDir: /var/lib/openfuyao/etcd
        extraArgs:
          snapshot-count: "10000"
```
## 3. Machine - 单个节点抽象
### 3.1 设计思路
**类比：Pod**

**核心思想：**
- Machine是对单个节点的抽象，类似Pod对容器的抽象
- 组合Bootstrap和Infrastructure配置
- 管理节点从创建到删除的完整生命周期

**解决的问题：**
1. **节点抽象**：统一不同基础设施的节点表示
2. **生命周期管理**：自动管理节点的创建、更新、删除
3. **配置分离**：将引导配置和基础设施配置分离
4. **状态追踪**：实时反映节点的健康状态

**关键设计：**
- 通过Bootstrap Provider生成节点初始化配置
- 通过Infrastructure Provider创建底层资源
- 自动生成ProviderID用于节点标识
### 3.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: <machine-name>
  namespace: <namespace>
  labels:
    cluster.x-k8s.io/cluster-name: <cluster-name>
spec:
  # 所属集群
  clusterName: <cluster-name>
  
  # Kubernetes版本
  version: v1.28.0
  
  # 引导配置
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: <bootstrap-config-name>
    dataSecretName: <bootstrap-secret-name>
  
  # 基础设施引用
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSMachine
    name: <infrastructure-machine-name>
  
  # 超时配置
  nodeDrainTimeout: 5m
  nodeDeletionTimeout: 10m
  
  # 提供者ID
  providerID: aws:///us-west-2/i-1234567890abcdef0

status:
  # 节点引用
  nodeRef:
    name: ip-192-168-1-100.us-west-2.compute.internal
  
  # 就绪状态
  bootstrapReady: true
  infrastructureReady: true
  
  # 阶段
  phase: Running|Provisioning|Failed|Deleting
  
  # 地址信息
  addresses:
    - type: InternalIP
      address: 192.168.1.100
    - type: ExternalIP
      address: 54.123.45.67
```
### 3.3 使用样例
#### 样例1：手动创建单个工作节点
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: prod-worker-1
  namespace: default
  labels:
    cluster.x-k8s.io/cluster-name: production-cluster
    node-pool: general
spec:
  clusterName: production-cluster
  version: v1.28.0
  
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: prod-worker-1-config
  
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSMachine
    name: prod-worker-1-infra
  
  nodeDrainTimeout: 5m
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfig
metadata:
  name: prod-worker-1-config
  namespace: default
spec:
  joinConfiguration:
    nodeRegistration:
      name: '{{ ds.meta_data.local_hostname }}'
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: aws
        node-labels: "node-pool=general"
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSMachine
metadata:
  name: prod-worker-1-infra
  namespace: default
spec:
  instanceType: m5.large
  ami:
    id: ami-0c55b159cbfafe1f0
  sshKeyName: my-key-pair
  rootVolume:
    size: 100
    type: gp3
```
**设计说明：**
- 手动创建适合特殊场景（如GPU节点、存储节点）
- 需要显式创建Bootstrap和Infrastructure资源
- 适合需要精确控制单个节点的场景
#### 样例2：控制平面节点
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: prod-control-plane-1
  namespace: default
  labels:
    cluster.x-k8s.io/cluster-name: production-cluster
    cluster.x-k8s.io/control-plane: "true"
spec:
  clusterName: production-cluster
  version: v1.28.0
  
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: prod-control-plane-1-config
  
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSMachine
    name: prod-control-plane-1-infra
```
## 4. MachineSet - 一组相同配置的节点
### 4.1 设计思路
**类比：ReplicaSet**

**核心思想：**
- 管理一组相同配置的节点，确保副本数符合期望
- 提供节点池的基础能力
- 支持水平扩展和收缩

**解决的问题：**
1. **批量管理**：一次管理多个相同配置的节点
2. **副本保证**：确保节点数量符合期望
3. **标签选择**：通过标签选择器管理特定节点组
4. **故障恢复**：自动替换失败的节点

**关键设计：**
- 使用Pod模板模式定义节点配置
- 通过Selector匹配管理的Machine
- 支持删除策略
### 4.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineSet
metadata:
  name: <machineset-name>
  namespace: <namespace>
spec:
  # 所属集群
  clusterName: <cluster-name>
  
  # 副本数
  replicas: 3
  
  # 选择器
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: <cluster-name>
      node-pool: general
  
  # 机器模板
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: <cluster-name>
        node-pool: general
    spec:
      clusterName: <cluster-name>
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: <bootstrap-template-name>
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: <infrastructure-template-name>
  
  # 删除策略
  deletePolicy: Random|Newest|Oldest
  
  # 最小就绪时间
  minReadySeconds: 100

status:
  replicas: 3
  readyReplicas: 3
  availableReplicas: 3
  updatedReplicas: 3
  unavailableReplicas: 0
```
### 4.3 使用样例
#### 样例1：通用工作节点池
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineSet
metadata:
  name: prod-worker-general
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 5
  
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
      node-pool: general
  
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: production-cluster
        node-pool: general
    spec:
      clusterName: production-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: prod-worker-general-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: prod-worker-general-infra
  
  deletePolicy: Random
  minReadySeconds: 100
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: prod-worker-general-config
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          name: '{{ ds.meta_data.local_hostname }}'
          criSocket: /var/run/containerd/containerd.sock
          kubeletExtraArgs:
            cloud-provider: aws
            node-labels: "node-pool=general"
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSMachineTemplate
metadata:
  name: prod-worker-general-infra
  namespace: default
spec:
  template:
    spec:
      instanceType: m5.large
      ami:
        id: ami-0c55b159cbfafe1f0
      sshKeyName: my-key-pair
      rootVolume:
        size: 100
        type: gp3
```
**设计说明：**
- 使用模板定义节点配置
- 通过标签选择器管理节点
- 支持水平扩展（修改replicas字段）
## 5. MachineDeployment - 管理工作节点组，支持升级
### 5.1 设计思路
**类比：Deployment**

**核心思想：**
- 在MachineSet之上提供声明式更新能力
- 支持滚动升级、回滚等高级功能
- 管理多个MachineSet版本

**解决的问题：**
1. **声明式升级**：修改模板自动触发滚动升级
2. **版本管理**：维护历史版本，支持回滚
3. **升级策略**：控制升级速率和方式
4. **零停机升级**：通过滚动升级保证服务连续性

**关键设计：**
- 使用ReplicaSet模式管理MachineSet
- 支持RollingUpdate和Recreate策略
- 维护revision历史
- 支持暂停和恢复升级
### 5.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: <deployment-name>
  namespace: <namespace>
spec:
  # 所属集群
  clusterName: <cluster-name>
  
  # 副本数
  replicas: 10
  
  # 选择器
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: <cluster-name>
      node-pool: application
  
  # 升级策略
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
      deletePolicy: Newest
  
  # 机器模板
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: <cluster-name>
        node-pool: application
    spec:
      clusterName: <cluster-name>
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: <bootstrap-template-name>
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: <infrastructure-template-name>
  
  # 最小就绪时间
  minReadySeconds: 300
  
  # 历史版本限制
  revisionHistoryLimit: 10
  
  # 暂停升级
  paused: false

status:
  replicas: 10
  readyReplicas: 10
  availableReplicas: 10
  updatedReplicas: 10
  unavailableReplicas: 0
  observedGeneration: 2
  phase: Running|ScalingUp|ScalingDown|Failed
```
### 5.3 使用样例
#### 样例1：应用节点池（支持滚动升级）
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: prod-worker-app
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 10
  
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
      node-pool: application
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
      deletePolicy: Newest
  
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: production-cluster
        node-pool: application
    spec:
      clusterName: production-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: prod-worker-app-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: prod-worker-app-infra
  
  minReadySeconds: 300
  revisionHistoryLimit: 10
```
**升级流程：**
```bash
# 1. 修改MachineDeployment的版本
kubectl patch machinedeployment prod-worker-app --type merge -p '
{
  "spec": {
    "template": {
      "spec": {
        "version": "v1.29.0"
      }
    }
  }
}'

# 2. 观察升级进度
kubectl get machinedeployment prod-worker-app -w

# 3. 查看历史版本
kubectl get machinedeployment prod-worker-app -o jsonpath='{.status.revision}'

# 4. 回滚到上一个版本
kubectl rollout undo machinedeployment prod-worker-app
```
#### 样例2：GPU节点池
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: prod-worker-gpu
  namespace: default
spec:
  clusterName: production-cluster
  replicas: 3
  
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
      node-pool: gpu
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: production-cluster
        node-pool: gpu
    spec:
      clusterName: production-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: prod-worker-gpu-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: prod-worker-gpu-infra
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSMachineTemplate
metadata:
  name: prod-worker-gpu-infra
  namespace: default
spec:
  template:
    spec:
      instanceType: p3.2xlarge
      ami:
        id: ami-0c55b159cbfafe1f0
      rootVolume:
        size: 200
        type: gp3
```
## 6. Infrastructure Provider - 创建底层机器资源
### 6.1 设计思路
**类比：云厂商驱动**

**核心思想：**
- 抽象不同基础设施提供商的实现细节
- 提供统一的接口管理底层资源
- 实现跨云平台的集群管理

**解决的问题：**
1. **多云支持**：统一接口支持AWS、Azure、GCP、vSphere等
2. **资源抽象**：隐藏底层资源创建的复杂性
3. **生命周期管理**：管理资源的创建、更新、删除
4. **状态同步**：实时同步底层资源状态到Cluster API

**关键设计：**
- 定义InfrastructureCluster和InfrastructureMachine接口
- 每个云厂商实现自己的Provider
- 使用ProviderID标识底层资源
### 6.2 资源规格
#### InfrastructureCluster规格
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: <cluster-name>
  namespace: <namespace>
spec:
  # AWS区域
  region: us-west-2
  
  # SSH密钥
  sshKeyName: my-key-pair
  
  # 网络配置
  network:
    vpc:
      id: vpc-12345678
      cidrBlock: 10.0.0.0/16
    subnets:
      - id: subnet-12345678
        cidrBlock: 10.0.1.0/24
        availabilityZone: us-west-2a
        isPublic: true
  
  # 控制平面端点
  controlPlaneEndpoint:
    host: ""
    port: 6443
  
  # 负载均衡器配置
  controlPlaneLoadBalancer:
    scheme: internet-facing
    crossZoneLoadBalancing: true
  
  # 镜像查找配置
  imageLookupOrg: "123456789012"
  imageLookupBaseOS: "ubuntu-20.04"
  
  # Bastion主机
  bastion:
    enabled: true
    instanceType: t3.micro

status:
  ready: true
  network:
    apiServerElb:
      dnsName: a1b2c3d4e5f6g7h8i9j0.elb.us-west-2.amazonaws.com
    securityGroups:
      controlPlane: sg-12345678
      node: sg-87654321
```
#### InfrastructureMachine规格
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSMachine
metadata:
  name: <machine-name>
  namespace: <namespace>
spec:
  # 实例ID（由Provider填充）
  instanceID: i-1234567890abcdef0
  
  # 实例类型
  instanceType: m5.large
  
  # AMI镜像
  ami:
    id: ami-0c55b159cbfafe1f0
  
  # SSH密钥
  sshKeyName: my-key-pair
  
  # 根卷配置
  rootVolume:
    size: 100
    type: gp3
    encrypted: true
  
  # 安全组
  additionalSecurityGroups:
    - id: sg-12345678
  
  # 标签
  tags:
    - key: Name
      value: prod-worker
    - key: Environment
      value: production
  
  # IAM实例配置
  iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
  
  # Spot实例配置
  spotMarketOptions:
    maxPrice: "0.05"

status:
  ready: true
  addresses:
    - type: InternalIP
      address: 192.168.1.100
    - type: ExternalIP
      address: 54.123.45.67
  instanceState: running
```
### 6.3 使用样例
#### 样例1：AWS完整基础设施
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: prod-infra
  namespace: default
spec:
  region: us-west-2
  sshKeyName: production-key
  
  network:
    vpc:
      cidrBlock: 10.0.0.0/16
    subnets:
      - cidrBlock: 10.0.1.0/24
        availabilityZone: us-west-2a
        isPublic: true
      - cidrBlock: 10.0.2.0/24
        availabilityZone: us-west-2b
        isPublic: true
      - cidrBlock: 10.0.3.0/24
        availabilityZone: us-west-2a
        isPublic: false
      - cidrBlock: 10.0.4.0/24
        availabilityZone: us-west-2b
        isPublic: false
  
  controlPlaneEndpoint:
    host: ""
    port: 6443
  
  controlPlaneLoadBalancer:
    scheme: internet-facing
    crossZoneLoadBalancing: true
  
  bastion:
    enabled: true
    instanceType: t3.micro
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSMachineTemplate
metadata:
  name: prod-control-plane-template
  namespace: default
spec:
  template:
    spec:
      instanceType: m5.xlarge
      ami:
        id: ami-0c55b159cbfafe1f0
      sshKeyName: production-key
      rootVolume:
        size: 100
        type: gp3
        encrypted: true
      iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      tags:
        - key: Role
          value: control-plane
        - key: Environment
          value: production
```
#### 样例2：BKE基础设施
```yaml
apiVersion: infrastructure.bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: bke-infra
  namespace: cluster-system
spec:
  controlPlaneEndpoint:
    host: 192.168.1.100
    port: 6443
  
  nodes:
    - hostname: master-0
      ipAddress: 192.168.1.101
      role: master
    - hostname: master-1
      ipAddress: 192.168.1.102
      role: master
    - hostname: master-2
      ipAddress: 192.168.1.103
      role: master
  
  network:
    podCIDR: "10.244.0.0/16"
    serviceCIDR: "10.96.0.0/12"
    vip: "192.168.1.100"
---
apiVersion: infrastructure.bke.bocloud.com/v1beta1
kind: BKEMachineTemplate
metadata:
  name: bke-control-plane-template
  namespace: cluster-system
spec:
  template:
    spec:
      sshPublicKey: "ssh-rsa AAAAB3..."
      labels:
        node-role.kubernetes.io/control-plane: ""
      taints:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
```
## 7. Bootstrap Provider - 初始化机器上的Kubernetes
### 7.1 设计思路
**类比：安装脚本**

**核心思想：**
- 生成节点初始化配置（如kubeadm配置）
- 支持多种引导方式（kubeadm、ignition等）
- 将配置注入到节点中

**解决的问题：**
1. **配置生成**：自动生成节点加入集群所需的配置
2. **密钥管理**：管理引导密钥和证书
3. **平台适配**：支持不同操作系统的引导方式
4. **安全性**：通过Secret传递敏感信息

**关键设计：**
- 生成bootstrap data（如kubeadm join命令）
- 将data存储在Secret中
- Infrastructure Provider负责将Secret注入节点
### 7.2 资源规格
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfig
metadata:
  name: <config-name>
  namespace: <namespace>
spec:
  # 集群配置（仅用于控制平面初始化）
  clusterConfiguration:
    apiServer:
      extraArgs: {}
      certSANs: []
    controllerManager:
      extraArgs: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    dns:
      type: CoreDNS
    networking:
      dnsDomain: cluster.local
      podSubnet: "10.244.0.0/16"
  
  # 初始化配置（第一个控制平面节点）
  initConfiguration:
    localAPIEndpoint:
      advertiseAddress: ""
      bindPort: 6443
    nodeRegistration:
      name: ""
      criSocket: /var/run/containerd/containerd.sock
      taints: []
  
  # 加入配置（后续节点）
  joinConfiguration:
    controlPlane:
      localAPIEndpoint:
        advertiseAddress: ""
        bindPort: 6443
    discovery:
      bootstrapToken:
        apiServerEndpoint: ""
        token: ""
        caCertHashes: []
    nodeRegistration:
      name: ""
      criSocket: /var/run/containerd/containerd.sock
  
  # 额外文件
  files:
    - path: /etc/kubernetes/audit-policy.yaml
      content: |
        apiVersion: audit.k8s.io/v1
        kind: Policy
  
  # 用户配置
  users:
    - name: core
      sshAuthorizedKeys:
        - ssh-rsa AAAAB3...
      sudo: ALL=(ALL) NOPASSWD:ALL
  
  # NTP配置
  ntp:
    servers:
      - 0.pool.ntp.org
      - 1.pool.ntp.org
  
  # 格式
  format: cloud-config|ignition

status:
  ready: true
  dataSecretName: prod-worker-1-bootstrap
```
### 7.3 使用样例
#### 样例1：控制平面初始化配置
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfig
metadata:
  name: prod-control-plane-1
  namespace: default
spec:
  clusterConfiguration:
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
        enable-admission-plugins: NodeRestriction
        audit-log-path: /var/log/kubernetes/audit.log
      certSANs:
        - prod-api.example.com
        - 192.168.1.100
      extraVolumes:
        - name: audit-log
          hostPath: /var/log/kubernetes
          mountPath: /var/log/kubernetes
    controllerManager:
      extraArgs:
        cluster-signing-duration: "8760h"
    etcd:
      local:
        dataDir: /var/lib/etcd
        extraArgs:
          snapshot-count: "10000"
    networking:
      dnsDomain: cluster.local
      podSubnet: "10.244.0.0/16"
      serviceSubnet: "10.96.0.0/12"
  
  initConfiguration:
    localAPIEndpoint:
      advertiseAddress: 192.168.1.101
      bindPort: 6443
    nodeRegistration:
      name: master-0
      criSocket: /var/run/containerd/containerd.sock
      taints:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
  
  files:
    - path: /etc/kubernetes/audit-policy.yaml
      content: |
        apiVersion: audit.k8s.io/v1
        kind: Policy
        rules:
          - level: Metadata
    - path: /etc/sysctl.d/99-kubernetes.conf
      content: |
        net.bridge.bridge-nf-call-iptables = 1
        net.bridge.bridge-nf-call-ip6tables = 1
        net.ipv4.ip_forward = 1
  
  users:
    - name: core
      sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ...
      sudo: ALL=(ALL) NOPASSWD:ALL
```
#### 样例2：工作节点加入配置
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: prod-worker-config
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        discovery:
          bootstrapToken:
            apiServerEndpoint: 192.168.1.100:6443
            token: abcdef.0123456789abcdef
            caCertHashes:
              - sha256:7da8c6b8f87e8e6c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9c9
        nodeRegistration:
          name: '{{ ds.meta_data.local_hostname }}'
          criSocket: /var/run/containerd/containerd.sock
          kubeletExtraArgs:
            node-labels: "node-pool=general"
            max-pods: "110"
      
      files:
        - path: /etc/containerd/config.toml
          content: |
            version = 2
            [plugins."io.containerd.grpc.v1.cri"]
              [plugins."io.containerd.grpc.v1.cri".containerd]
                snapshotter = "overlayfs"
```
## 8. ClusterClass - 集群模板蓝图
### 8.1 设计思路
**类比：Helm Chart**

**核心思想：**
- 定义可复用的集群模板
- 支持参数化配置
- 简化集群创建流程

**解决的问题：**
1. **模板复用**：一次定义，多次使用
2. **标准化**：确保集群配置一致性
3. **简化操作**：用户只需提供少量参数
4. **版本管理**：管理集群模板的版本

**关键设计：**
- 定义ControlPlane、Workers模板
- 支持Variables和Patches定制配置
- 通过Topology引用ClusterClass
### 8.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: <class-name>
  namespace: <namespace>
spec:
  # 控制平面模板
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: <template-name>
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: <template-name>
  
  # 基础设施模板
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSClusterTemplate
      name: <template-name>
  
  # 工作节点模板
  workers:
    machineDeployments:
      - class: default-worker
        template:
          bootstrap:
            ref:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
              kind: KubeadmConfigTemplate
              name: <template-name>
          infrastructure:
            ref:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
              kind: AWSMachineTemplate
              name: <template-name>
  
  # 变量定义
  variables:
    - name: region
      required: true
      schema:
        openAPIV3Schema:
          type: string
          description: "AWS region"
          enum:
            - us-east-1
            - us-west-2
  
  # 补丁定义
  patches:
    - name: regionPatch
      definitions:
        - selector:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: AWSClusterTemplate
            matchResources:
              infrastructureCluster: true
          jsonPatches:
            - op: replace
              path: /spec/template/spec/region
              valueFrom:
                variable: region
```
### 8.3 使用样例
#### 样例1：AWS ClusterClass
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: aws-quick-start
  namespace: default
spec:
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: aws-control-plane
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: aws-control-plane-machine
  
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSClusterTemplate
      name: aws-cluster
  
  workers:
    machineDeployments:
      - class: default-worker
        template:
          bootstrap:
            ref:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
              kind: KubeadmConfigTemplate
              name: aws-worker-bootstrap
          infrastructure:
            ref:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
              kind: AWSMachineTemplate
              name: aws-worker-machine
  
  variables:
    - name: region
      required: true
      schema:
        openAPIV3Schema:
          type: string
          enum:
            - us-east-1
            - us-west-2
    - name: instanceType
      required: false
      schema:
        openAPIV3Schema:
          type: string
          default: "m5.large"
    - name: sshKeyName
      required: true
      schema:
        openAPIV3Schema:
          type: string
  
  patches:
    - name: regionPatch
      definitions:
        - selector:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: AWSClusterTemplate
            matchResources:
              infrastructureCluster: true
          jsonPatches:
            - op: replace
              path: /spec/template/spec/region
              valueFrom:
                variable: region
    - name: instanceTypePatch
      definitions:
        - selector:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: AWSMachineTemplate
            matchResources:
              machineDeploymentClass:
                names:
                  - default-worker
          jsonPatches:
            - op: replace
              path: /spec/template/spec/instanceType
              valueFrom:
                variable: instanceType
```
#### 样例2：使用ClusterClass创建集群
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production
  namespace: default
spec:
  topology:
    class: aws-quick-start
    version: v1.28.0
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: default-worker
          name: md-0
          replicas: 5
    variables:
      - name: region
        value: us-west-2
      - name: instanceType
        value: m5.xlarge
      - name: sshKeyName
        value: production-key
```
## 9. Topology - 集群整体结构
### 9.1 设计思路
**类比：架构图**

**核心思想：**
- 在Cluster中定义集群的整体结构
- 引用ClusterClass作为模板
- 通过变量定制配置

**解决的问题：**
1. **结构可视化**：清晰展示集群组成
2. **简化配置**：只需填写关键参数
3. **一致性保证**：基于模板确保配置一致
### 9.2 使用样例
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: multi-zone-cluster
  namespace: production
spec:
  topology:
    class: aws-production
    version: v1.28.0
    controlPlane:
      replicas: 3
      metadata:
        labels:
          tier: control-plane
    workers:
      machineDeployments:
        - class: general-worker
          name: general-us-west-2a
          replicas: 3
          failureDomain: us-west-2a
        - class: general-worker
          name: general-us-west-2b
          replicas: 3
          failureDomain: us-west-2b
    variables:
      - name: region
        value: us-west-2
      - name: controlPlaneInstanceType
        value: m5.xlarge
```
## 10. MachineHealthCheck - 节点健康检测与替换
### 10.1 设计思路
**类比：PodDisruptionBudget + 自动修复**

**核心思想：**
- 监控节点健康状态
- 自动替换不健康的节点
- 防止大规模节点故障

**解决的问题：**
1. **自动修复**：无需人工干预自动替换故障节点
2. **健康监控**：持续监控节点状态
3. **保护机制**：防止同时替换过多节点
### 10.2 资源规格
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: <healthcheck-name>
  namespace: <namespace>
spec:
  clusterName: <cluster-name>
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: <cluster-name>
  unhealthyConditions:
    - type: Ready
      status: Unknown
      timeout: 300s
    - type: Ready
      status: "False"
      timeout: 300s
  maxUnhealthy: 40%
  nodeStartupTimeout: 10m
```
### 10.3 使用样例
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: worker-health-check
  namespace: default
spec:
  clusterName: production-cluster
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
      node-pool: general
  unhealthyConditions:
    - type: Ready
      status: Unknown
      timeout: 300s
    - type: Ready
      status: "False"
      timeout: 300s
    - type: MemoryPressure
      status: "True"
      timeout: 300s
  maxUnhealthy: 40%
  nodeStartupTimeout: 10m
```
## 11. Add-ons - 集群额外组件
### 11.1 设计思路
**类比：插件系统**

**核心思想：**
- 自动安装集群组件
- 支持多种资源类型
- 声明式管理
### 11.2 使用样例
```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: production-addons
  namespace: default
spec:
  clusterSelector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-cluster
  resources:
    - kind: ConfigMap
      name: cni-calico
    - kind: ConfigMap
      name: metrics-server
  strategy: Reconcile
```
## 12. Runtime Extensions - 生命周期钩子
### 12.1 设计思路
**类比：Operator Hook**

**核心思想：**
- 在集群生命周期关键点注入自定义逻辑
- 支持Webhook方式扩展
- 实现自定义工作流
### 12.2 使用样例
```yaml
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: custom-lifecycle-hooks
  namespace: cluster-system
spec:
  clientConfig:
    service:
      name: lifecycle-webhook
      namespace: cluster-system
      port: 443
      path: /lifecycle
  settings:
    preKubeadmCommand: "/usr/local/bin/pre-setup.sh"
    postKubeadmCommand: "/usr/local/bin/post-setup.sh"
```
## 总结对比表
| 资源 | 类比 | 核心职责 | 关键设计点 | 典型应用场景 |
|------|------|---------|-----------|-------------|
| **Cluster** | Deployment | 集群蓝图 | 组合模式，引用其他资源 | 创建新集群 |
| **ControlPlane** | StatefulSet | 管理控制平面 | 有序部署，高可用 | 管理Master节点 |
| **Machine** | Pod | 单个节点抽象 | 组合Bootstrap和Infrastructure | 手动创建特殊节点 |
| **MachineSet** | ReplicaSet | 节点副本管理 | 标签选择器，副本保证 | 节点池基础 |
| **MachineDeployment** | Deployment | 节点升级管理 | 滚动升级，版本管理 | 工作节点池 |
| **Infrastructure Provider** | 云厂商驱动 | 底层资源管理 | 抽象接口，多云支持 | AWS/Azure/GCP集成 |
| **Bootstrap Provider** | 安装脚本 | 节点初始化 | 配置生成，密钥管理 | Kubeadm/Ignition |
| **ClusterClass** | Helm Chart | 集群模板 | 参数化，补丁机制 | 标准化集群创建 |
| **Topology** | 架构图 | 集群结构定义 | 引用ClusterClass | 简化集群配置 |
| **MachineHealthCheck** | PDB+修复 | 节点健康监控 | 自动修复，保护机制 | 节点故障恢复 |
| **Add-ons** | 插件系统 | 组件安装 | 声明式，自动应用 | CNI/CSI安装 |
| **Runtime Extensions** | Operator Hook | 生命周期扩展 | Webhook，自定义逻辑 | 自定义工作流 |


        
