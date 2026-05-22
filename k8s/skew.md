# version skew
**Kubernetes 的 *version skew* 指的是不同组件之间允许存在的版本差异。核心规则是：API Server 必须处于最高版本，其他组件只能比它低一个或几个 minor 版本，不能更高。**这保证了集群在升级过程中保持兼容性和稳定性。  [Kubernetes](https://kubernetes.io/releases/version-skew-policy/)  [oneuptime.com](https://oneuptime.com/blog/post/2026-02-09-cluster-upgrades-version-skew/view)  [deepwiki.com](https://deepwiki.com/kubernetes/sig-release/4.2-version-skew-policy)  
## 🧩 关键规则概览
| 组件 | 允许的版本差异 | 说明 |
|------|----------------|------|
| **kube-apiserver** | HA 集群中最新与最旧实例必须在 **1 个 minor 版本内** | API Server 是最敏感组件，必须保持紧密一致 |
| **控制面组件** (controller-manager, scheduler, cloud-controller-manager) | 不得高于 API Server，可比它低 **1 个 minor 版本** | 保证控制面逻辑与 API Server 同步 |
| **kubelet** | 不得高于 API Server，可比它低 **最多 3 个 minor 版本**（<1.25 时最多 2 个） | 允许节点逐步升级，避免一次性全量升级 |
| **kube-proxy** | 不得高于 API Server，可比它低 **最多 3 个 minor 版本**；与 kubelet 可差 **±3 个 minor 版本** | 保证网络代理与节点兼容 |
| **kubectl** | 可比 API Server **高或低 1 个 minor 版本** | 提供客户端升级灵活性 |
## 🔄 升级场景中的应用
- **渐进式升级**：先升级 API Server，再逐步升级控制面和节点。  
- **HA 集群限制**：如果 API Server 有版本差异，会缩小 kubelet/kube-proxy 的可兼容范围。  
- **客户端灵活性**：kubectl 可提前或延后升级，方便运维。  
## ⚠️ 风险与挑战
- **违反 skew 规则** → 可能导致 API 不兼容、Pod 部署失败、集群不稳定。  
- **多云/混合环境** → Provider 必须遵循相同契约，否则跨集群治理困难。  
- **安全补丁引入** → 升级路径必须考虑安全修复的及时性。  
## 🔑 总结
Kubernetes 的 **version skew policy** 是升级和运维的核心规则：  
- **API Server 定义上限**，其他组件只能落后，不能超前。  
- **kubelet/kube-proxy** 有更宽松的落后范围，支持渐进升级。  
- **kubectl** 提供灵活性，但仍需保持在 ±1 minor 版本范围。

# *K8s version skew 检查清单
帮助在升级或运维过程中逐项确认各组件是否符合兼容规则，避免因版本差异导致集群不稳定。  
## 🧩 K8s Version Skew 检查清单
- **API Server**  
  - HA 集群中最新与最旧的 API Server 实例必须在 **1 个 minor 版本内**。  
  - API Server 必须是集群中版本最高的组件。  
- **控制面组件**  
  - kube-controller-manager、kube-scheduler、cloud-controller-manager 不得高于 API Server。  
  - 可比 API Server 低 **1 个 minor 版本**。  
- **kubelet**  
  - 不得高于 API Server。  
  - 可比 API Server 低 **最多 3 个 minor 版本**（在 Kubernetes <1.25 时为 2 个）。  
  - 建议逐步升级，避免一次性全量升级。  
- **kube-proxy**  
  - 不得高于 API Server。  
  - 可比 API Server 低 **最多 3 个 minor 版本**。  
  - 与 kubelet 可差 **±3 个 minor 版本**。  
- **kubectl**  
  - 可比 API Server **高或低 1 个 minor 版本**。  
  - 提供客户端升级灵活性，但仍需保持在 ±1 范围。  
## ⚠️ 风险提示
- **违反 skew 规则** → 可能导致 API 不兼容、Pod 部署失败、集群不稳定。  
- **多云/混合环境** → Provider 必须遵循相同契约，否则跨集群治理困难。  
- **安全补丁引入** → 升级路径必须考虑安全修复的及时性。  
## 🔑 总结
这份检查清单确保在升级前逐项确认：  
1. **API Server 定义上限**，其他组件只能落后，不能超前。  
2. **控制面组件** 与 API Server 保持紧密一致。  
3. **kubelet/kube-proxy** 有更宽松的落后范围，支持渐进升级。  
4. **kubectl** 提供灵活性，但必须在 ±1 minor 范围。  
