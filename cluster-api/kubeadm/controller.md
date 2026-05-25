# cluster-api控制器
**Cluster API 包含一系列核心控制器，它们分别负责集群、控制平面、节点池以及健康检查等不同维度的资源管理。最常见的控制器包括 Cluster、Machine、MachineSet、MachineDeployment、KubeadmControlPlane、Bootstrap、Infrastructure Provider、MachineHealthCheck、ClusterClass 与 Topology 控制器。**  [release-1-1.cluster-api.sigs.k8s.io](https://release-1-1.cluster-api.sigs.k8s.io/developer/architecture/controllers)  
## 📑 Cluster API 控制器分类
### 1. 核心控制器
- **Cluster Controller**：管理整个集群对象，协调控制平面与工作节点的生命周期。  
- **Machine Controller**：负责单个节点（VM/容器）的创建、删除和状态维护。  
- **MachineSet Controller**：类似于 Kubernetes 的 ReplicaSet，保证一组 Machine 的副本数。  
- **MachineDeployment Controller**：类似于 Deployment，负责滚动升级和扩缩容 MachineSet。  
### 2. 控制平面相关
- **KubeadmControlPlane Controller**：管理基于 kubeadm 的控制平面节点，负责证书、版本升级、扩缩容。  
- **其他 ControlPlane Provider Controller**：例如 MicroK8sControlPlane，用于不同的控制平面实现。  
### 3. 引导与基础设施
- **Bootstrap Controller**：负责节点的引导配置（如 kubeadm join/init 脚本）。  
- **Infrastructure Provider Controllers**：如 CAPA (AWS)、CAPZ (Azure)、CAPD (Docker)，根据 `infrastructureRef` 创建实际的 VM/容器资源。  
### 4. 健康与策略
- **MachineHealthCheck Controller**：监控节点健康，自动触发替换不健康的 Machine。  
- **ClusterClass Controller**：管理集群的模板化定义，支持多集群一致性。  
- **Topology Controller**：根据 ClusterClass 自动生成控制平面与 MachineDeployment，统一管理集群拓扑。  
### 5. 扩展与实验性
- **MachinePool Controller**：管理大规模节点池（实验性功能）。  
- **MachineSetPreflightChecks Controller**：在创建 MachineSet 前进行预检查。  
## ⚖️ 总结
- **核心控制器**：Cluster、Machine、MachineSet、MachineDeployment。  
- **控制平面控制器**：KubeadmControlPlane 等。  
- **基础设施控制器**：各 Provider（AWS、Azure、Docker 等）。  
- **健康与拓扑控制器**：MachineHealthCheck、ClusterClass、Topology。  

这些控制器协同工作，保证集群从声明式配置到实际运行环境的自动化管理，类似 Kubernetes 自身的控制器模式，但对象是集群和节点而不是 Pod。  
## 架构图 
```mermaid
flowchart TD

    A["Cluster：Cluster Controller)"]
      --> B["ControlPlane：KubeadmControlPlane Controller"]

    A --> C["MachineDeployment：MachineDeployment Controller)]

    B --> D["MachineSet：MachineSet Controller"]
    C --> D

    D --> E["Machine：Machine Controller"]

    E --> F["InfrastructureRef：Infra Provider Controller"]
    F --> G["VM/实例/容器"]

    E --> H["Bootstrap：Bootstrap Controller)"]
    H --> I["kubeadm init/join配置"]

    %% 新增健康检查链路
    J["MachineHealthCheck Controller"]
      --> E

    J --> K["检测不健康 Machine"]
    K --> L["触发替换：删除旧 Machine/创建新Machine)"]
```
### 图解说明
- **Cluster Controller**：顶层对象，协调控制平面与工作节点。  
- **KubeadmControlPlane Controller**：负责控制平面节点的声明与管理。  
- **MachineDeployment Controller**：负责工作节点池的声明与滚动升级。  
- **MachineSet Controller**：保证副本数，承接 MachineDeployment 或 ControlPlane 的输出。  
- **Machine Controller**：管理单个节点对象。  
- **Infra Provider Controller**：根据 `infrastructureRef` 创建实际 VM/容器。  
- **Bootstrap Controller**：负责节点的 kubeadm 引导配置。  
- **MachineHealthCheck Controller**：持续监控 `Machine` 的健康状态（如节点不可达、未就绪）。  
- **检测不健康 Machine**：当发现某个节点不健康时，标记为 `unhealthy`。  
- **触发替换**：删除不健康的 `Machine`，由上游的 `MachineSet` 或 `MachineDeployment` 创建新的 `Machine` 来替换。  
- **整体链路**：Cluster → ControlPlane/Deployment → MachineSet → Machine → Infra Provider/Bootstrap → VM → kubeadm；同时 MachineHealthCheck 在 Machine 层介入，保证节点池的健康性。  

这样你可以直观地看到：**MachineHealthCheck Controller 是在 Machine 层插入的守护机制，负责自动替换不健康节点，保证集群的高可用性。**  

要不要我再帮你扩展这个图，加上 **ClusterClass/Topology Controller**，让你看到它如何在最顶层统一生成 ControlPlane 和 MachineDeployment？
