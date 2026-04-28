# 特殊的标签（labels）和注解（annotations）
**在 Cluster API 中，除了 `cluster.x-k8s.io/paused` 之外，还有一系列特殊的标签（labels）和注解（annotations），用于控制对象的调谐、标识控制平面、部署拓扑、生命周期钩子等。这些标签和注解是 Cluster API 控制器识别和管理资源的关键机制。**  

## 📑 常见特殊标签（Labels）
| 标签 | 作用 | 应用对象 |
|------|------|----------|
| **cluster.x-k8s.io/cluster-name** | 标识机器所属的集群 | Machines、外部对象 |
| **cluster.x-k8s.io/control-plane** | 标识该机器属于控制平面 | Machines |
| **cluster.x-k8s.io/control-plane-name** | 控制平面名称（可能是哈希） | Machines |
| **cluster.x-k8s.io/deployment-name** | 标识机器属于某个 MachineDeployment | Machines |
| **cluster.x-k8s.io/drain=skip** | Pod 在节点 drain 时不被驱逐 | Pods（工作负载集群） |
| **cluster.x-k8s.io/interruptible** | 标识节点运行在可中断实例上（如 Spot） | Nodes |
| **cluster.x-k8s.io/pool-name** | 标识机器属于某个 MachinePool | Machines |
| **cluster.x-k8s.io/provider** | 标识组件属于某个 provider，便于 clusterctl 管理 | Provider 组件 |
| **cluster.x-k8s.io/set-name** | 标识机器属于某个 MachineSet | Machines |
| **cluster.x-k8s.io/watch-filter** | 控制器选择性调谐时使用 | 所有 Cluster API 对象 |
| **machine-template-hash** | MachineDeployment 中模板的哈希值 | Machines |
| **topology.cluster.x-k8s.io/deployment-name** | MachineDeployment 的拓扑来源 | MachineDeployments |
| **topology.cluster.x-k8s.io/owned** | 标识对象由 ClusterTopology 管理 | ClusterTopology 对象 |

## 📑 常见特殊注解（Annotations）
| 注解 | 作用 | 应用对象 |
|------|------|----------|
| **cluster.x-k8s.io/paused** | 暂停对象调谐 | Cluster、Machine、MachineDeployment 等 |
| **before-upgrade.hook.cluster.cluster.x-k8s.io** | 升级前生命周期钩子，阻止版本传播 | Clusters |
| **cluster.x-k8s.io/annotations-from-machine** | 跟踪节点上来源于 Machine 的注解 | Nodes |
| **remediate-machine** | 标识需要修复的机器 | Machines |
| **skip-remediation** | 跳过机器修复逻辑 | Machines |
| **externally-managed** | 标识对象由外部系统管理，Cluster API 不调谐 | 任意对象 |
| **replicas-managed-by-external-autoscaler** | 标识副本数由外部自动扩缩器管理 | MachineDeployment/MachineSet |

## ⚠️ 使用注意事项
- **调试/维护时**：`paused` 注解可防止控制器覆盖手动修改，但必须记得移除，否则资源不会再被调谐。  
- **升级场景**：`before-upgrade.hook` 注解可阻止拓扑版本传播，常用于分阶段升级。  
- **自动扩缩容**：`replicas-managed-by-external-autoscaler` 注解避免与 Cluster API 的副本数控制冲突。  
- **外部管理**：`externally-managed` 注解用于声明某些资源由外部系统负责，避免 Cluster API 干预。  

✅ 总结：Cluster API 提供了丰富的 **特殊标签与注解**，用于标识资源归属、控制调谐行为、支持拓扑管理和生命周期钩子。除了常见的 `cluster.x-k8s.io/paused`，还包括 `control-plane`、`deployment-name`、`watch-filter` 等标签，以及 `skip-remediation`、`externally-managed` 等注解。正确使用这些机制，可以灵活控制集群的自动化管理行为。  

# *Cluster API 标签与注解速查表
按用途分类，方便快速查阅。  

## 🛠 调谐控制（Reconciliation Control）
| 名称 | 类型 | 作用 |
|------|------|------|
| `cluster.x-k8s.io/paused` | 注解 | 暂停对象调谐（Cluster、Machine、MachineDeployment 等）。 |
| `cluster.x-k8s.io/watch-filter` | 标签 | 控制器选择性调谐时使用，过滤对象。 |
| `externally-managed` | 注解 | 声明对象由外部系统管理，Cluster API 不调谐。 |
| `skip-remediation` | 注解 | 跳过机器修复逻辑。 |
| `remediate-machine` | 注解 | 标识需要修复的机器。 |

## 🏗️ 拓扑管理（Topology Management）
| 名称 | 类型 | 作用 |
|------|------|------|
| `topology.cluster.x-k8s.io/deployment-name` | 标签 | MachineDeployment 的拓扑来源。 |
| `topology.cluster.x-k8s.io/owned` | 标签 | 标识对象由 ClusterTopology 管理。 |
| `machine-template-hash` | 标签 | MachineDeployment 中模板的哈希值，用于滚动更新。 |

## 🔄 生命周期钩子（Lifecycle Hooks）
| 名称 | 类型 | 作用 |
|------|------|------|
| `before-upgrade.hook.cluster.cluster.x-k8s.io` | 注解 | 升级前生命周期钩子，阻止版本传播。 |
| `cluster.x-k8s.io/annotations-from-machine` | 注解 | 跟踪节点上来源于 Machine 的注解。 |

## 🌐 外部管理与扩缩容（External Management & Autoscaling）
| 名称 | 类型 | 作用 |
|------|------|------|
| `replicas-managed-by-external-autoscaler` | 注解 | 标识副本数由外部自动扩缩器管理。 |
| `cluster.x-k8s.io/provider` | 标签 | 标识组件属于某个 provider，便于 clusterctl 管理。 |
| `cluster.x-k8s.io/interruptible` | 标签 | 标识节点运行在可中断实例上（如 Spot）。 |

## 📊 集群与控制平面标识（Cluster & Control Plane Identification）
| 名称 | 类型 | 作用 |
|------|------|------|
| `cluster.x-k8s.io/cluster-name` | 标签 | 标识机器所属的集群。 |
| `cluster.x-k8s.io/control-plane` | 标签 | 标识该机器属于控制平面。 |
| `cluster.x-k8s.io/control-plane-name` | 标签 | 控制平面名称（可能是哈希）。 |
| `cluster.x-k8s.io/deployment-name` | 标签 | 标识机器属于某个 MachineDeployment。 |
| `cluster.x-k8s.io/set-name` | 标签 | 标识机器属于某个 MachineSet。 |
| `cluster.x-k8s.io/pool-name` | 标签 | 标识机器属于某个 MachinePool。 |

## ✅ 总结
- **调谐控制类**：`paused`、`watch-filter`、`externally-managed` 等。  
- **拓扑管理类**：`topology.*`、`machine-template-hash`。  
- **生命周期钩子类**：`before-upgrade.hook.*`、`annotations-from-machine`。  
- **外部管理类**：`replicas-managed-by-external-autoscaler`、`provider`、`interruptible`。  
- **集群标识类**：`cluster-name`、`control-plane`、`deployment-name` 等。  

# `cluster.x-k8s.io/paused`
在 Cluster API 中，`cluster.x-k8s.io/paused` 是一个 **注解（annotation）**，它的作用是让控制器暂时停止对某个对象（如 Cluster、Machine、MachineDeployment）的自动调谐和变更。  

## 📑 作用机制
- **注解位置**：可以加在 Cluster、Machine、MachineDeployment 等资源对象上。  
- **效果**：当该注解存在且值为 `"true"` 时，Cluster API 的控制器不会对该对象进行 reconcile（调谐）。  
- **范围**：  
  - 如果加在 Cluster 上 → 整个集群的所有子资源（Machine、MachineDeployment 等）都会暂停。  
  - 如果只加在某个 MachineDeployment 上 → 仅该对象暂停，其他资源仍正常调谐。  

## ⚙️ 使用场景
1. **手动维护或调试**  
   - 暂停自动调谐，避免控制器覆盖手动修改。  
2. **升级或迁移**  
   - 在进行集群升级或迁移时，先暂停，确保控制器不会自动触发不必要的变更。  
3. **问题排查**  
   - 当怀疑控制器逻辑导致异常时，可以暂停对象，进行独立排查。  

## 📊 示例
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  annotations:
    cluster.x-k8s.io/paused: "true"
```

此时，Cluster API 控制器不会再对 `my-cluster` 及其子资源进行调谐，直到移除该注解。  

## ✅ 总结
- `cluster.x-k8s.io/paused` 是 Cluster API 的 **暂停开关**。  
- 加上该注解后，控制器停止调谐，避免自动修改资源。  
- 常用于 **手动维护、升级迁移、问题排查** 等场景。  
