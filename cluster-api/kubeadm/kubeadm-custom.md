# kubeadm 实现控制面组件分离功能的方案分析

## 一、核心结论

**kubeadm 原生不支持控制面组件分离部署**。这是 kubeadm 的架构限制，不是配置问题。

## 二、kubeadm 架构限制分析

### 2.1 kubeadm 的控制面部署模型
```
┌─────────────────────────────────────────────────────────────┐
│              kubeadm 控制面节点 (每个节点)                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ kube-        │  │ kube-        │  │ kube-            │  │
│  │ apiserver    │  │ scheduler    │  │ controller-      │  │
│  │ (Static Pod) │  │ (Static Pod) │  │ manager          │  │
│  │              │  │              │  │ (Static Pod)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────┐                                          │
│  │ etcd         │  (stacked 模式)                          │
│  │ (Static Pod) │                                          │
│  └──────────────┘                                          │
│                                                              │
│  所有组件通过 static pod manifest 部署在 /etc/kubernetes/    │
│  manifests/ 目录下，kubeadm 自动生成和管理这些文件            │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 为什么无法分离
| 原因 | 说明 |
|------|------|
| **Static Pod 绑定** | kubeadm 将 API Server/Scheduler/Controller Manager 都作为 static pod 部署在同一节点 |
| **证书共享** | 控制面组件共享 `/etc/kubernetes/pki` 目录下的证书 |
| **kubeadm join --control-plane** | 加入控制面节点时，会部署所有三个组件 |
| **无配置项** | 没有任何 kubeadm 配置可以禁止某个节点运行特定组件 |

## 三、替代方案
### 方案 A: 接受同节点部署 (强烈推荐)
**适用场景**: 绝大多数生产环境
```
┌─────────────────────────────────────────────────────────────┐
│                    9 节点控制面架构                           │
│                                                              │
│  Node-1    Node-2    Node-3    Node-4    Node-5              │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │ APIS │  │ APIS │  │ APIS │  │ APIS │  │ APIS │          │
│  │ Sched│  │ Sched│  │ Sched│  │ Sched│  │ Sched│          │
│  │ CM   │  │ CM   │  │ CM   │  │ CM   │  │ CM   │          │
│  │ etcd │  │ etcd │  │ etcd │  │      │  │      │          │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘          │
│                                                              │
│  Node-6    Node-7    Node-8    Node-9                        │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                    │
│  │ APIS │  │ APIS │  │ APIS │  │ APIS │                    │
│  │ Sched│  │ Sched│  │ Sched│  │ Sched│                    │
│  │ CM   │  │ CM   │  │ CM   │  │ CM   │                    │
│  │      │  │      │  │      │  │ etcd │                    │
│  └──────┘  └──────┘  └──────┘  └──────┘                    │
│                                                              │
│  说明:                                                        │
│  - 每个节点运行全部控制面组件                                  │
│  - API Server 无状态，通过 LB 水平扩展                        │
│  - Scheduler/CM 通过 leader election 保证单 active            │
│  - 实际资源消耗: Scheduler ~50MB, CM ~100MB                  │
│  - etcd 可以是非均匀分布 (3/5/7 个成员)                       │
└─────────────────────────────────────────────────────────────┘
```
**为什么这是最佳方案**:

1. **API Server 无状态**: 通过 LB 分发请求，增加节点即增加吞吐量
2. **Scheduler/CM 资源占用极小**: 即使 9 个节点都运行，额外消耗可忽略
3. **运维简单**: 升级、扩缩容路径清晰
4. **高可用**: 任意节点故障不影响集群

### 方案 B: 自定义 ControlPlane Provider
**适用场景**: 极端需求，必须物理分离

**架构设计**:
```
┌─────────────────────────────────────────────────────────────┐
│                    自定义 ControlPlane 架构                   │
│                                                              │
│  API Server 节点池          Scheduler 节点池                 │
│  ┌──────────┐ ┌──────────┐  ┌──────────┐ ┌──────────┐      │
│  │ APIS-1   │ │ APIS-2   │  │ Sched-1  │ │ Sched-2  │      │
│  │ APIS-3   │ │          │  │          │ │          │      │
│  └──────────┘ └──────────┘  └──────────┘ └──────────┘      │
│                                                              │
│  Controller Manager 节点池   etcd 节点池                     │
│  ┌──────────┐ ┌──────────┐  ┌──────────┐ ┌──────────┐      │
│  │ CM-1     │ │ CM-2     │  │ etcd-1   │ │ etcd-2   │      │
│  │          │ │          │  │ etcd-3   │ │          │      │
│  └──────────┘ └──────────┘  └──────────┘ └──────────┘      │
│                                                              │
│  实现方式:                                                    │
│  1. 开发自定义 ControlPlane Provider                         │
│  2. 不使用 kubeadm 管理控制面组件                            │
│  3. 自行管理证书、配置、健康检查                              │
│  4. 实现完整的升级/回滚逻辑                                   │
└─────────────────────────────────────────────────────────────┘
```

**实现要点**:
```go
// 自定义 ControlPlane CRD
type CustomControlPlaneSpec struct {
    APIServer     ComponentSpec `json:"apiServer"`
    Scheduler     ComponentSpec `json:"scheduler"`
    ControllerMgr ComponentSpec `json:"controllerManager"`
    Etcd          EtcdSpec      `json:"etcd"`
}

type ComponentSpec struct {
    Replicas         int32              `json:"replicas"`
    MachineTemplate  MachineTemplateRef `json:"machineTemplate"`
    Image            string             `json:"image"`
    ExtraArgs        []Arg              `json:"extraArgs,omitempty"`
    ConfigSecretRef  SecretReference    `json:"configSecretRef,omitempty"`
}
```
**工作量评估**:

| 模块 | 工作量 | 说明 |
|------|--------|------|
| CRD 定义 | 1 周 | 定义 ControlPlane/Component 等 CRD |
| 证书管理 | 2-3 周 | CA 签发、轮转、分发 |
| 配置管理 | 2 周 | 生成各组件配置文件 |
| 健康检查 | 2 周 | 组件健康探测、状态聚合 |
| 升级逻辑 | 3-4 周 | 滚动升级、版本兼容 |
| 扩缩容 | 2 周 | 动态增减节点 |
| 测试 | 4 周 | 端到端测试、故障注入 |
| **总计** | **16-18 周** | 约 4 个月 |

### 方案 C: kubeadm + 手动干预 (不推荐)
**思路**: 使用 kubeadm 初始化后，手动删除不需要的 static pod manifest
```bash
# 1. 在节点 A 上执行 kubeadm init
kubeadm init --config kubeadm-config.yaml

# 2. 删除不需要的 static pod manifests
rm /etc/kubernetes/manifests/kube-scheduler.yaml
rm /etc/kubernetes/manifests/kube-controller-manager.yaml

# 3. 在节点 B 上手动部署 scheduler
# 复制证书
scp node-a:/etc/kubernetes/pki/* /etc/kubernetes/pki/

# 手动创建 scheduler static pod manifest
cat > /etc/kubernetes/manifests/kube-scheduler.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.31.0
    command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    volumeMounts:
    - name: kubeconfig
      mountPath: /etc/kubernetes
      readOnly: true
    - name: pki
      mountPath: /etc/kubernetes/pki
      readOnly: true
  volumes:
  - name: kubeconfig
    hostPath:
      path: /etc/kubernetes
  - name: pki
    hostPath:
      path: /etc/kubernetes/pki
EOF
```
**为什么强烈不推荐**:

| 问题 | 说明 |
|------|------|
| **升级破坏** | `kubeadm upgrade` 会重新生成所有 manifests |
| **证书同步** | 需要手动同步证书到所有节点 |
| **配置漂移** | 手动修改的配置无法被 kubeadm 管理 |
| **无自动化** | 扩缩容、故障修复都需要手动操作 |
| **不受 CAPI 支持** | KCP 无法管理这种架构 |

## 四、方案对比
| 维度 | 方案 A (同节点) | 方案 B (自定义 Provider) | 方案 C (手动干预) |
|------|-----------------|--------------------------|-------------------|
| **可行性** | ✅ 原生支持 | ✅ 需开发 | ⚠️ 可行但脆弱 |
| **开发成本** | 无 | 高 (4 个月+) | 低但维护成本高 |
| **升级支持** | ✅ 自动 | ✅ 需自行实现 | ❌ 升级会破坏 |
| **扩缩容** | ✅ 自动 | ✅ 需自行实现 | ❌ 手动 |
| **故障修复** | ✅ 自动 | ✅ 需自行实现 | ❌ 手动 |
| **与 CAPI 集成** | ✅ 完美 | ✅ 需开发 Provider | ❌ 不兼容 |
| **运维复杂度** | 低 | 中 | 高 |
| **推荐度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |

## 五、推荐方案
**对于 99% 的场景，推荐使用方案 A (同节点部署)**:
```yaml
# KCP 配置 - 9 节点控制面
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 9
  version: v1.31.0
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          # 大规模优化
          - name: max-requests-inflight
            value: "2000"
          - name: max-mutating-requests-inflight
            value: "1000"
          - name: watch-cache-sizes
            value: "pods#1000,secrets#1000,configmaps#1000"
      controllerManager:
        extraArgs:
          - name: leader-elect
            value: "true"
          - name: concurrent-gc-syncs
            value: "30"
          - name: node-cidr-mask-size
            value: "24"
      scheduler:
        extraArgs:
          - name: leader-elect
            value: "true"
```
**资源消耗估算** (9 节点):

| 组件 | 单节点内存 | 9 节点总内存 | 说明 |
|------|-----------|-------------|------|
| API Server | ~300MB | ~2.7GB | 无状态，可水平扩展 |
| Scheduler | ~50MB | ~450MB | 仅 1 个 active |
| Controller Manager | ~100MB | ~900MB | 仅 1 个 active |
| etcd | ~200MB | ~1.8GB (5 节点) | 仅 etcd 成员占用 |

| **总计** | | **~5.85GB** | 分散到 9 节点 |

**结论**: 即使 9 节点全部运行所有组件，额外内存消耗约 4GB (Scheduler + CM 的 standby 副本)，对于大规模集群来说这是完全可以接受的代价。
