# Variables 和 Patches

# 在 **ClusterClass** 中，`workers.machineDeployments` 的 **class** 字段作用是：  
它充当 **模板选择器**，用于指向 `MachineDeploymentClass`，从而定义工作节点的机器规格、配置和生命周期策略。  

## ✨ 关键作用
- **模板绑定**  
  `class` 字段将 `workers.machineDeployments` 与 `MachineDeploymentClass` 绑定，确保工作节点使用预定义的机器模板。

- **一致性保障**  
  所有引用同一个 `class` 的 `MachineDeployment` 会继承相同的配置（如 VM 类型、磁盘大小、网络设置），保证集群中工作节点的一致性。

- **灵活扩展**  
  可以在 `ClusterClass` 中定义多个 `MachineDeploymentClass`，通过不同的 `class` 值来区分不同类型的工作负载节点（例如 GPU 节点、通用计算节点）。

- **升级与滚动替换**  
  修改 `MachineDeploymentClass` 的关键字段会触发滚动替换，确保工作节点逐步升级而不影响整体集群稳定性。

## 📑 对比表：ClusterClass 中的 class 字段
| **字段** | **作用对象** | **影响范围** |
|----------------|----------------|----------------|
| `workers.machineDeployments[].class` | 指向 `MachineDeploymentClass` | 决定工作节点的机器规格与配置 |
| `controlPlane.machineTemplate.class` | 指向 `MachineTemplateClass` | 决定控制平面节点的机器规格 |
| `infrastructure.class` | 指向 `ClusterInfrastructureClass` | 决定集群基础设施（网络、负载均衡等） |

## ⚠️ 注意事项
- **class 名称必须唯一**，避免不同 `MachineDeployment` 引用同一个 class 时产生冲突。  
- **修改 class 会触发替换**，需提前规划兼容性与回滚。  
- **推荐使用多 class 策略**：例如 `general-purpose`、`gpu-accelerated`、`storage-heavy`，以便灵活满足不同工作负载需求。  

## ✅ 总结
在 `ClusterClass` 中，`workers.machineDeployments.class` 的作用是 **将工作节点与预定义的 MachineDeploymentClass 模板绑定**，从而实现 **一致性、灵活性和可控的升级策略**。它是 ClusterClass **声明式管理工作节点类型** 的核心机制。  
