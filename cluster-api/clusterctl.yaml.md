# baremetal provider注册
要让 `clusterctl` 识别自研的 **baremetal provider**，必须在 `$HOME/.cluster-api/clusterctl.yaml` 中显式声明。这个文件就是 clusterctl 的 **provider registry 配置**，它会覆盖或补充内置的默认列表。

## clusterctl.yaml 示例

```yaml
providers:
  - name: baremetal
    url: https://github.com/your-org/cluster-api-provider-baremetal/releases/latest/infrastructure-components.yaml
    type: InfrastructureProvider
```

### 字段说明
- **name**：你在 `clusterctl init --infrastructure <name>` 中使用的名字，必须一致。  
- **url**：指向 provider 的 `infrastructure-components.yaml` 文件，可以是 GitHub release、HTTP 目录或本地文件路径。  
- **type**：这里必须是 `InfrastructureProvider`。  

## 可选增强
如果你希望支持 **版本升级**，还需要提供 `metadata.yaml`：
```yaml
apiVersion: clusterctl.cluster.x-k8s.io/v1alpha3
kind: Metadata
releaseSeries:
  - major: 0
    minor: 7
    contract: v1alpha4
```
把它放在 provider 的 `config/default/` 目录下，clusterctl 就能识别版本契约。

## 使用流程
1. 在 `$HOME/.cluster-api/clusterctl.yaml` 添加上述配置。  
2. 运行：
   ```bash
   clusterctl init --infrastructure baremetal
   ```
   clusterctl 会从你指定的 URL 下载并安装 provider。  
3. 验证：
   ```bash
   clusterctl config repositories
   ```
   确认列表里有 `baremetal`。  

## 总结
- **默认 clusterctl 不认识 baremetal** → 必须在 `clusterctl.yaml` 显式声明。  
- **核心配置**：`name`、`url`、`type`。

# metadata.yaml

在自研 **baremetal provider** 时，`metadata.yaml` 是必须提供的文件，它告诉 `clusterctl` 你的 provider 支持哪些 **版本契约 (contract)**，以及不同版本之间的升级路径。  

## 📌 一个典型的 `metadata.yaml` 示例

```yaml
apiVersion: clusterctl.cluster.x-k8s.io/v1alpha3
kind: Metadata
releaseSeries:
  - major: 0
    minor: 7
    contract: v1alpha4
  - major: 0
    minor: 8
    contract: v1alpha5
```

### 字段说明
- **apiVersion**：固定为 `clusterctl.cluster.x-k8s.io/v1alpha3`。  
- **kind**：固定为 `Metadata`。  
- **releaseSeries**：定义 provider 的版本系列。  
  - `major` / `minor`：对应你 provider 的版本号，例如 `v0.7.x`。  
  - `contract`：对应 Cluster API 的版本契约（如 `v1alpha4`、`v1alpha5`）。  

## 🛠️ 使用场景
- 当你运行：
  ```bash
  clusterctl init --infrastructure baremetal
  ```
  clusterctl 会读取你 provider 的 `metadata.yaml`，确认它支持的 contract。  
- 在升级时：
  ```bash
  clusterctl upgrade plan
  ```
  clusterctl 会根据 `metadata.yaml` 判断是否能从 `v0.7.x` 升级到 `v0.8.x`，以及对应的 contract 是否兼容。

## ✅ 最佳实践
- 每次发布新版本时，更新 `metadata.yaml`，确保 contract 与 Cluster API 主版本保持一致。  
- 如果你们的 provider只支持一个 contract，可以只写一个 `releaseSeries`。  
- 把 `metadata.yaml` 放在 provider 的 `config/default/` 目录下，随 release 一起打包。  

要不要我帮你生成一个 **完整的 clusterctl.yaml 模板**（包含 core、bootstrap、control-plane 以及你们自研 baremetal provider），这样你可以直接复制到 `$HOME/.cluster-api/` 下使用？
