# 使用Helm SDK进行安装与升级时，Helm的数据存放在哪里了？

在使用 **Helm SDK**（或直接用 Helm CLI）进行安装与升级时，Helm 会把它的“状态数据”存放在 Kubernetes 集群内部，而不是本地磁盘。具体来说：

### Helm 数据存放位置
- **ConfigMap/Secret**  
  Helm 将每一次安装或升级的 Release 信息（包括 Chart 元数据、values 配置、manifest 等）序列化后存储在目标命名空间的 ConfigMap 或 Secret 中。  
  - Helm v2 默认使用 ConfigMap。  
  - Helm v3 默认使用 Secret（更安全，因为包含敏感信息）。  

- **命名空间**  
  Release 数据存放在安装时指定的命名空间中。例如：  
  ```bash
  helm install myapp ./chart --namespace myns
  ```  
  那么 Helm 会在 `myns` 命名空间里创建一个 Secret，名字类似：  
  ```
  sh.helm.release.v1.myapp.v1
  ```

- **Release 历史**  
  每次升级都会生成一个新的 Secret（或 ConfigMap），版本号递增。这样 Helm 可以回滚到之前的版本。  
  例如：  
  ```
  sh.helm.release.v1.myapp.v1
  sh.helm.release.v1.myapp.v2
  sh.helm.release.v1.myapp.v3
  ```

### 本地数据
- **Helm 本地缓存**  
  Helm 在本地也会维护缓存（如 Chart 下载包、插件等），通常位于：  
  ```
  ~/.cache/helm
  ~/.config/helm
  ~/.helm (旧版本)
  ```
  但这些只是缓存和配置，不包含 Release 的运行状态。

📌 总结：  
Helm 的安装与升级数据主要存放在 **Kubernetes 集群的 Secret/ConfigMap** 中，按命名空间和版本号区分；本地只保留缓存和配置，不影响 Release 的实际状态。

