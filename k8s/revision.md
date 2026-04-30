# revision lastAppliedRevision
在 Kubernetes 的对象元数据里，**`revision`** 和 **`lastAppliedRevision`** 这两个字段常出现在 **Deployment** 或 **StatefulSet** 等控制器的注解中，用来追踪版本和配置的应用情况。它们的作用不同：
## 🔧 `revision`
- **来源**：通常由控制器（如 Deployment、StatefulSet）自动维护  
- **作用**：表示当前控制器所处的版本号（类似迭代次数）  
- **场景**：  
  - 每次你更新 Deployment 的 Pod 模板，控制器会生成新的 ReplicaSet，并递增 `revision`  
  - 方便回滚时知道当前是第几次更新  

👉 举例：  
```yaml
annotations:
  deployment.kubernetes.io/revision: "3"
```
表示这是第三次更新后的版本。
## 🔧 `lastAppliedRevision`
- **来源**：通常由 `kubectl rollout` 或控制器在回滚时使用  
- **作用**：记录上一次成功应用的版本号  
- **场景**：  
  - 当你执行 `kubectl rollout undo` 时，系统会参考 `lastAppliedRevision` 来决定回滚到哪个版本  
  - 用于确保回滚的目标是最近一次稳定版本，而不是随便一个旧版本  

👉 举例：  
```yaml
annotations:
  controller.kubernetes.io/last-applied-revision: "2"
```
表示上一次成功应用的版本是第 2 次更新。
## 📊 对比总结
| 字段                | 谁维护 | 含义 | 使用场景 |
|---------------------|--------|------|----------|
| `revision`          | 控制器 | 当前版本号 | 标识当前 Deployment/StatefulSet 的迭代次数 |
| `lastAppliedRevision` | 控制器/rollout工具 | 上一次成功应用的版本号 | 回滚时参考，确保回到稳定版本 |

换句话说：  
- **`revision`** 是“我现在是第几版”  
- **`lastAppliedRevision`** 是“上一次成功的版本是第几版”  

---

要不要我帮你画一个 **Deployment 更新流程图**（包含 revision 递增和 lastAppliedRevision 的变化），让你更直观地看到它们的作用？
