# /etc/containerd/certs.d/{registry}/hosts.toml

在 **containerd** 中，`/etc/containerd/certs.d/{registry}/hosts.toml` 是 **镜像仓库的专用配置文件**，用于定义某个 registry 的访问方式、认证信息、TLS 校验和加速源。它比 `config.toml` 更灵活，可以针对不同仓库单独配置，并且支持热加载，无需重启 containerd。

## 📂 hosts.toml 的作用
- **镜像源配置**：指定仓库的 endpoint，支持多个地址实现容错与负载均衡。  
- **认证与安全**：配置用户名/密码、TLS 证书，保证镜像拉取安全。  
- **加速与优化**：为公共仓库配置国内加速源，提升拉取速度。  
- **能力声明**：定义该仓库支持的操作（pull、push、resolve）。  

## 🔑 示例配置

路径：`/etc/containerd/certs.d/docker.io/hosts.toml`

```toml
server = "https://registry-1.docker.io"

[host."https://<your-id>.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]
  skip_verify = false

[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = false
```

### 字段说明
- **server**：默认仓库地址。  
- **[host."URL"]**：定义一个 endpoint，可以是加速源或私有仓库。  
- **capabilities**：支持的操作类型（`pull` 拉取、`push` 推送、`resolve` 解析）。  
- **skip_verify**：是否跳过 TLS 校验（生产环境必须为 `false`）。  

## 📊 使用场景
| **场景** | **配置方式** | **效果** |
|---|---|---|
| **公共仓库加速** | 添加国内镜像源 endpoint | 提升拉取速度 |
| **私有仓库认证** | 配置 TLS 证书与用户名/密码 | 确保安全访问 |
| **多 endpoint 容错** | 配置多个 host | 一个失败时自动切换 |
| **企业安全仓库** | 强制 TLS 校验 + 签名验证 | 保证镜像可信 |

## ⚠️ 注意事项
- 修改 `hosts.toml` **无需重启 containerd**，支持热加载。  
- 避免使用 `skip_verify=true`，否则可能导致中间人攻击风险。  
- 建议为每个 registry 单独建立目录，例如：  
  - `/etc/containerd/certs.d/docker.io/hosts.toml`  
  - `/etc/containerd/certs.d/harbor.local/hosts.toml`  

# containerd的配置文件

**在 containerd 中，主要的配置文件是 `/etc/containerd/config.toml`，它决定了守护进程的运行方式、插件加载、镜像仓库设置、日志与监控等。除此之外，还有 `hosts.toml`（用于 registry 配置）、systemd 服务文件以及证书目录等辅助配置文件。**  

## 📂 containerd 配置文件分类与作用

### 1. config.toml
- **位置**：`/etc/containerd/config.toml`  
- **作用**：核心配置文件，定义 containerd 守护进程的行为。  
- **关键字段**：
  - `version`：配置版本（最新为 v4）。  
  - `root`：元数据存储目录（默认 `/var/lib/containerd`）。  
  - `state`：运行时状态目录（默认 `/run/containerd`）。  
  - `[grpc]`：gRPC socket 配置（监听地址、TLS 证书）。  
  - `[debug]`：调试日志级别与格式。  
  - `[metrics]`：Prometheus 指标监听端口。  
  - `[plugins]`：插件配置（如 CRI、snapshotter、registry）。  

👉 **这是最重要的文件，决定了 containerd 的整体运行逻辑。**  [containerd](https://containerd.io/docs/main/man/containerd-config.toml.5/)  [Github](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md?plain=1)

### 2. hosts.toml
- **位置**：`/etc/containerd/certs.d/<registry_host>/hosts.toml`  
- **作用**：为特定镜像仓库配置 endpoint、认证方式、TLS 校验。  
- **典型用途**：
  - 配置国内镜像加速源（如阿里云、腾讯云）。  
  - 设置私有仓库的用户名/密码或证书。  
  - 定义多个 endpoint，实现容错与负载均衡。  

👉 **比 config.toml 更灵活，支持动态加载，无需重启 containerd。**

### 3. systemd 服务文件
- **位置**：`/usr/lib/systemd/system/containerd.service`  
- **作用**：定义 containerd 的启动方式、参数、依赖。  
- **常见用途**：
  - 指定 `--config` 参数加载不同配置文件。  
  - 控制启动顺序（依赖网络、存储）。  
  - 设置资源限制（CPU、内存）。  

### 4. 证书与认证文件
- **位置**：`/etc/containerd/certs.d/`  
- **作用**：存放 registry 的 TLS 证书与密钥。  
- **用途**：
  - 确保镜像拉取安全。  
  - 支持私有仓库的双向认证。  

## 📊 配置文件作用对比

| **文件** | **位置** | **作用** | **典型场景** |
|---|---|---|---|
| **config.toml** | `/etc/containerd/config.toml` | 守护进程核心配置 | 插件、日志、监控、CRI |
| **hosts.toml** | `/etc/containerd/certs.d/<registry>/hosts.toml` | 镜像仓库配置 | 加速源、私有仓库认证 |
| **systemd service** | `/usr/lib/systemd/system/containerd.service` | 启动参数与依赖 | 指定配置文件、资源限制 |
| **证书文件** | `/etc/containerd/certs.d/` | TLS 安全认证 | 镜像仓库安全通信 |

## ⚠️ 注意事项
- **版本兼容**：不同版本的 config.toml 字段可能变化，升级时需检查。  
- **安全性**：避免在配置文件中存储明文密码，推荐使用证书或密钥管理。  
- **动态更新**：`hosts.toml` 支持热加载，但 `config.toml` 修改后需重启 containerd。  

# containerd配置最佳实践清单

**最佳实践是通过合理配置 `config.toml` 与 `hosts.toml`，结合安全认证、镜像加速、监控与日志管理，确保 containerd 在生产环境下既高效又安全。** 以下清单总结了常见的生产级配置要点。  [百度开发者中心](https://developer.baidu.com/article/detail.html?id=3850802)  [百度智能云](https://cloud.baidu.com/article/3850645)  [百度智能云](https://cloud.baidu.com/article/3849399)  

## 📂 containerd 配置最佳实践清单

### 1. 核心配置文件
- 使用 `containerd config default > /etc/containerd/config.toml` 生成基础配置。  
- **关键字段**：
  - `root` 与 `state` 路径：确保磁盘空间充足。  
  - `[grpc]`：配置 socket 地址，推荐使用 `/run/containerd/containerd.sock`。  
  - `[metrics]`：启用 Prometheus 指标，便于监控。  
  - `[debug]`：生产环境关闭 debug，避免日志过量。

### 2. 镜像仓库配置
- **公共仓库加速**：配置国内镜像源（阿里云、腾讯云），并保留官方地址作为 fallback。  
  ```toml
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://<your-id>.mirror.aliyuncs.com", "https://registry-1.docker.io"]
  ```
- **私有仓库认证**：使用 `auth` 段配置用户名/密码，推荐 base64 编码。  
- **TLS 安全**：必须配置 CA 证书，避免 `insecure_skip_verify=true` 出现在生产环境。  
- **hosts.toml**：为不同仓库单独配置 endpoint 与证书，支持热加载。

### 3. 安全优化
- 启用 **镜像签名验证**（如 Notary），防止拉取恶意镜像。  
- 使用 **TLS 双向认证**，保护私有仓库通信。  
- 避免在配置文件中存储明文密码，推荐使用密钥管理系统。  

### 4. 性能优化
- **镜像缓存**：确保 `/var/lib/containerd` 有足够空间，提升重复拉取效率。  
- **并行拉取**：合理调整并发数，避免网络瓶颈。  
- **CDN 加速**：为全球分布的仓库配置 CDN 域名。  

### 5. 监控与日志
- 启用 `[metrics]`，结合 Prometheus+Grafana 监控延迟、错误率。  
- 配置 systemd 日志轮转，避免日志撑满磁盘。  
- 定期检查 `ctr images pull` 与 `kubectl describe pod`，验证镜像拉取是否正常。  

## ⚠️ 风险与注意事项
- **跳过 TLS 验证**：仅限测试环境，生产必须启用证书。  
- **认证信息泄露**：避免明文存储，使用安全插件或 Vault。  
- **镜像源切换错误**：可能导致 Pod 无法拉取镜像，需保留官方仓库作为后备。  

# 生产环境 containerd配置模板

下面是一份 **生产环境 containerd 配置模板清单**，涵盖了常见的最佳实践：日志、监控、镜像仓库加速与认证、安全优化等。你可以在 `/etc/containerd/config.toml` 和 `hosts.toml` 中应用这些配置。

## 📂 config.toml 示例（生产环境）

```toml
version = 2

root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0

[debug]
  level = "info"   # 生产环境关闭 debug，避免日志过量

[metrics]
  address = "0.0.0.0:1338"   # Prometheus 指标监听端口

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
  max_concurrent_downloads = 3

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = [
        "https://<your-id>.mirror.aliyuncs.com",
        "https://registry-1.docker.io"
      ]
```

## 📂 hosts.toml 示例（镜像仓库配置）

路径：`/etc/containerd/certs.d/<registry_host>/hosts.toml`
```toml
server = "https://registry-1.docker.io"

[host."https://<your-id>.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]
  skip_verify = false

[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve", "push"]
```

## 🔑 最佳实践要点

- **日志与监控**：启用 `[metrics]`，结合 Prometheus+Grafana 监控延迟、错误率。  
- **镜像加速**：配置国内加速源，并保留官方仓库作为 fallback。  
- **私有仓库认证**：使用 `hosts.toml` 配置用户名/密码或 TLS 证书，避免明文存储。  
- **安全优化**：启用 TLS 校验，避免 `insecure_skip_verify=true` 出现在生产环境。  
- **性能优化**：合理设置 `max_concurrent_downloads`，避免网络瓶颈。  

## ⚠️ 注意事项
- 修改 `config.toml` 后需重启 containerd；`hosts.toml` 支持热加载。  
- 确保 `/var/lib/containerd` 磁盘空间充足，避免镜像缓存撑满。  
- 使用 **密钥管理系统**（如 Vault）存储认证信息，避免泄露。  

# /etc/crictl.yaml

在生产环境中，除了 `config.toml` 和 `hosts.toml`，**containerd 还会用到 `/etc/crictl.yaml`**，它是 **crictl 工具的配置文件**，主要用于指定 CRI 接口的连接方式。  

## 📂 etc/crictl.yaml 的作用
- **定义运行时 socket**：告诉 `crictl` 如何连接到 containerd 的 CRI 插件。  
- **简化命令行**：避免每次执行 `crictl` 都要手动指定 `--runtime-endpoint`。  
- **支持多运行时**：可切换不同的 CRI 实现（如 containerd、CRI-O）。  

## 🔑 常见配置示例

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```

### 字段说明
- **runtime-endpoint**：指定 CRI socket 地址，containerd 默认是 `/run/containerd/containerd.sock`。  
- **image-endpoint**：镜像管理接口，通常与 runtime 相同。  
- **timeout**：命令超时时间，默认 10 秒。  
- **debug**：是否启用调试日志。  

## 📊 使用场景
| **场景** | **配置方式** | **效果** |
|---|---|---|
| **单一运行时（containerd）** | 指定 `/run/containerd/containerd.sock` | 简化命令行操作 |
| **多运行时环境** | 修改 `runtime-endpoint` 指向不同 socket | 快速切换运行时 |
| **调试模式** | `debug: true` | 输出详细日志，便于排查问题 |

## ⚠️ 注意事项
- `etc/crictl.yaml` 仅影响 **crictl 工具**，不会改变 containerd 本身的行为。  
- 如果 socket 路径错误，`crictl` 会报错无法连接。  
- 在 Kubernetes 环境中，通常只需配置一次，后续所有 `crictl` 命令都会自动生效。  
