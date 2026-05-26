# Cluster-API 中与 K8s 版本相关的字段

## 一、核心资源版本字段

### 1.1 Cluster
| 字段 | 路径 | 说明 |
|------|------|------|
| **Topology Version** | `spec.topology.version` | 集群目标 K8s 版本 (使用 ClusterClass 时) |

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
spec:
  topology:
    version: v1.31.0   # ← 集群 K8s 版本
```

### 1.2 Machine
| 字段 | 路径 | 说明 |
|------|------|------|
| **Version** | `spec.version` | 该 Machine 应运行的 K8s 版本 |

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: Machine
spec:
  version: v1.31.0   # ← Machine K8s 版本
```

### 1.3 MachineDeployment
| 字段 | 路径 | 说明 |
|------|------|------|
| **Template Version** | `spec.template.spec.version` | Worker 节点目标 K8s 版本 |

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
spec:
  template:
    spec:
      version: v1.31.0   # ← Worker K8s 版本
```

### 1.4 MachineSet
| 字段 | 路径 | 说明 |
|------|------|------|
| **Template Version** | `spec.template.spec.version` | MachineSet 管理的节点 K8s 版本 |

### 1.5 MachinePool
| 字段 | 路径 | 说明 |
|------|------|------|
| **Template Version** | `spec.template.spec.version` | MachinePool 管理的节点 K8s 版本 |

## 二、控制面资源版本字段

### 2.1 KubeadmControlPlane
| 字段 | 路径 | 说明 |
|------|------|------|
| **Version** | `spec.version` | 控制面目标 K8s 版本 |
| **Status Version** | `status.version` | 控制面当前最低 K8s 版本 |

```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  version: v1.31.0   # ← 控制面 K8s 版本
status:
  version: v1.31.0   # ← 当前最低版本
```

## 三、ClusterClass 相关版本字段

### 3.1 ClusterClass
| 字段 | 路径 | 说明 |
|------|------|------|
| **KubernetesVersions** | `spec.kubernetesVersions` | 允许的 K8s 版本列表 (用于链式升级) |

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
spec:
  kubernetesVersions:
    - v1.29.0
    - v1.30.0
    - v1.31.0
    - v1.32.0
```

## 四、Bootstrap 配置版本字段

### 4.1 KubeadmConfig / KubeadmConfigTemplate
| 字段 | 路径 | 说明 |
|------|------|------|
| **KubernetesVersion** | `spec.clusterConfiguration.kubernetesVersion` | kubeadm 目标 K8s 版本 |

```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
kind: KubeadmConfig
spec:
  clusterConfiguration:
    kubernetesVersion: v1.31.0   # ← kubeadm K8s 版本
```

## 五、Runtime Hooks 版本字段

### 5.1 Lifecycle Hooks
| Hook | 字段 | 说明 |
|------|------|------|
| **BeforeClusterUpgrade** | `request.fromKubernetesVersion` | 升级前 K8s 版本 |
| | `request.toKubernetesVersion` | 升级目标 K8s 版本 |
| **AfterControlPlaneUpgrade** | `request.kubernetesVersion` | 控制面升级后 K8s 版本 |
| **AfterWorkersUpgrade** | `request.kubernetesVersion` | Worker 升级后 K8s 版本 |
| **AfterClusterUpgrade** | `request.kubernetesVersion` | 集群升级后 K8s 版本 |

### 5.2 Topology Mutation Hooks
| Hook | 字段 | 说明 |
|------|------|------|
| **GeneratePatches** | `variables[].name: "version"` | 集群/ControlPlane/MD/MachinePool 的 K8s 版本变量 |

### 5.3 Upgrade Plan Hook
| 字段 | 说明 |
|------|------|
| `request.fromControlPlaneKubernetesVersion` | 控制面当前 K8s 版本 |
| `request.fromWorkersKubernetesVersion` | Worker 当前最低 K8s 版本 |
| `request.toKubernetesVersion` | 升级目标 K8s 版本 |
| `response.controlPlaneUpgrades[].version` | 控制面升级路径中的中间版本 |
| `response.workersUpgrades[].version` | Worker 升级路径中的中间版本 |

## 六、版本字段传播路径
```
┌─────────────────────────────────────────────────────────────────────┐
│                    K8s 版本传播路径                                  │
│                                                                      │
│  Cluster.spec.topology.version                                      │
│      │                                                               │
│      ▼                                                               │
│  KubeadmControlPlane.spec.version                                   │
│      │                                                               │
│      ▼                                                               │
│  Machine.spec.version                                               │
│      │                                                               │
│      ▼                                                               │
│  KubeadmConfig.spec.clusterConfiguration.kubernetesVersion          │
│      │                                                               │
│      ▼                                                               │
│  目标节点上实际运行的 kubelet/kubeadm 版本                            │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────   │
│                                                                      │
│  Cluster.spec.topology.version                                      │
│      │                                                               │
│      ▼                                                               │
│  MachineDeployment.spec.template.spec.version                       │
│      │                                                               │
│      ▼                                                               │
│  Machine.spec.version                                               │
│      │                                                               │
│      ▼                                                               │
│  KubeadmConfig.spec.joinConfiguration (继承控制面版本)                │
│      │                                                               │
│      ▼                                                               │
│  目标节点上实际运行的 kubelet/kubeadm 版本                            │
└─────────────────────────────────────────────────────────────────────┘
```

## 七、版本相关打印列
| 资源 | kubectl 列 | JSONPath |
|------|-----------|----------|
| Cluster | Version | `.spec.topology.version` |
| Machine | Version | `.spec.version` |
| MachineDeployment | Version | `.spec.template.spec.version` |
| MachineSet | Version | `.spec.template.spec.version` |
| MachinePool | Version | `.spec.template.spec.version` |
| KubeadmControlPlane | Version | `.spec.version` |

## 八、注意事项
| 注意事项 | 说明 |
|----------|------|
| **升级期间版本差异** | 升级过程中，Cluster/ControlPlane 的 version 可能与实际节点版本不同 |

| **版本格式** | 必须为语义化版本，如 `v1.31.0`，不能省略 `v` 前缀 |
| **版本倾斜** | 需遵循 [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) |
| **Managed Topology** | 使用 ClusterClass 时，所有 Machine 必须使用相同 K8s 版本 |
