# conversion
**Kubernetes 的 CRD conversion 机制用于在同一个自定义资源定义（CRD）支持多个版本时，保证 API Server 能够在不同版本之间透明地转换对象，从而实现平滑升级与兼容。它通过 `served`/`storage` 标志和可选的 conversion webhook 来完成。**

## 核心机制

### 1. 多版本支持
- **`spec.versions`**：CRD 可以定义多个版本（如 v1alpha1、v1beta1、v1）。  
- **`served: true`**：该版本可通过 API Server 提供给客户端。  
- **`storage: true`**：该版本是唯一的存储版本，所有对象在 etcd 中以此版本保存。
- 
### 2. 转换策略
- **None**：仅修改 `apiVersion` 字段，不做 schema 转换。适用于字段完全兼容的情况。  
- **Webhook**：调用外部 webhook 服务执行自定义转换逻辑，适用于字段结构变化或语义变化。
- 
### 3. Hub-and-Spoke 模型
- **Hub version**：定义一个中心版本作为转换的“枢纽”。  
- **Spoke versions**：其他版本通过转换 webhook与 Hub 版本互相转换。  
- **好处**：避免版本之间两两转换的复杂性，只需维护 Hub ↔ Spoke 的转换逻辑。
- 
## 工作流程
1. **客户端请求**：用户通过 API 请求某个版本的 CRD 对象。  
2. **API Server**：如果请求版本不是存储版本，API Server 会调用 conversion 逻辑。  
3. **转换执行**：  
   - 若策略为 None → 自动转换 `apiVersion`。  
   - 若策略为 Webhook → 调用 webhook 服务进行字段映射与转换。  
4. **返回结果**：API Server 将转换后的对象返回给客户端。
5. 
## 示例场景
- **字段重命名**：`spec.region` → `spec.primaryRegion` + `spec.replicaRegions`。  
- **旧对象存储**：仍以旧版本存储在 etcd。  
- **新客户端请求**：通过 conversion webhook 转换为新版本结构返回。  
- **兼容性保证**：旧客户端仍可使用旧版本 API，不会因 schema 更新而报错。
- 
## 关键价值
- **零停机升级**：支持在生产环境中平滑引入新版本。  
- **兼容性**：允许不同客户端同时使用不同版本。  
- **灵活性**：通过 webhook 实现复杂的字段转换逻辑。
- 
## 总结
Kubernetes CRD conversion 机制通过 **served/storage 标志**与**conversion webhook**，实现了 **多版本共存、透明转换、零停机升级**。在设计 CRD 时，推荐采用 **Hub-and-Spoke 模型**，并在 schema 变更时实现 webhook 转换逻辑，以保证兼容性和稳定性。 
- [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) 
- [Medium](https://medium.com/@rameshavutu/crd-versioning-conversion-webhooks-evolving-your-api-without-breaking-production-42f7b95724ee)
