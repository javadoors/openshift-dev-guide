# 🎯 一、Kubernetes 中 Image 的格式规范（官方标准）

下面我给你一个 **完整、专业、可直接用于生产环境的答案**，涵盖：
- **Kubernetes 中 Pod 的 Image 格式规范**
- **如何在 Pod / Deployment / 全局层面指定镜像仓库（registry）**
- **企业常用镜像仓库策略**

K8s 中容器镜像的格式遵循 OCI / Docker 标准，完整格式如下：
```
[registry/]repository[:tag][@digest]
```

逐项解释：

| 字段 | 示例 | 说明 |
|------|------|------|
| **registry** | `registry.example.com` | 镜像仓库域名，可带端口 |
| **repository** | `nginx`, `myteam/app` | 镜像名称，可包含多级路径 |
| **tag** | `:v1.2.3`, `:latest` | 可选，不写默认 `latest` |
| **digest** | `@sha256:xxxx` | 可选，强一致性版本 |

## ✔ 常见合法格式示例

| 示例 | 说明 |
|------|------|
| `nginx` | 默认从 Docker Hub 拉取 |
| `nginx:1.25` | 指定 tag |
| `myregistry.com/app/web:1.0` | 指定 registry + tag |
| `myregistry.com:5000/app/web@sha256:xxxx` | 指定 registry + digest |

## 🎯 二、如何在 Pod 中指定镜像仓库（registry）

K8s 中指定镜像仓库有 **三种层级**：

### 🥇 方式 1：在 Pod/Deployment 中直接写完整镜像地址（最常用）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - name: web
    image: registry.example.com/team/app:v1.0
```
这是最明确、最推荐的方式。

### 🥈 方式 2：使用 imagePullSecrets（私有仓库认证）

如果你的 registry 需要登录：

#### 1）创建 secret

```
kubectl create secret docker-registry myregistrykey \
  --docker-server=registry.example.com \
  --docker-username=paul \
  --docker-password=yourpassword \
  --docker-email=paul@example.com
```

#### 2）在 Pod 中引用

```yaml
spec:
  imagePullSecrets:
  - name: myregistrykey
  containers:
  - name: web
    image: registry.example.com/team/app:v1.0
```

### 🥉 方式 3：通过 kubelet 配置全局默认 registry（企业常用）

适用于：
- 你想让所有镜像都从某个 registry 拉取
- 或者你想替换 Docker Hub 为国内镜像源

#### ✔ kubelet 配置（/var/lib/kubelet/config.yaml）

添加：
```yaml
imagePullPolicy: IfNotPresent
```

或配置 registry mirror：
```yaml
registryMirrors:
  - https://mirror.example.com
```

> 注意：不同 CRI（containerd / docker / CRI-O）配置方式不同。

## 🎯 三、containerd / Docker 配置默认镜像仓库（企业常用）

### 🟦 containerd（K8s 默认运行时）
编辑：
```
/etc/containerd/config.toml
```
添加 registry：
```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.example.com"]
  endpoint = ["https://registry.example.com"]
```

重启：
```
systemctl restart containerd
```

## 🟧 Docker（旧版 K8s 或开发环境）
编辑：
```
/etc/docker/daemon.json
```

添加：
```json
{
  "registry-mirrors": [
    "https://registry.example.com"
  ]
}
```

重启：
```
systemctl restart docker
```

## 🎯 四、企业常用镜像仓库策略（最佳实践）

| 场景 | 推荐策略 |
|------|----------|
| 生产环境 | 所有镜像必须写完整 registry 地址 |
| 多团队协作 | 使用多级路径：`registry/team/app` |
| 安全要求高 | 使用 digest：`image@sha256:xxx` |
| 私有仓库 | 使用 imagePullSecrets |
| 大规模集群 | 配置 containerd registry mirror |

## 🎁 五、给你一个最标准的 Deployment 示例（含私有仓库）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      imagePullSecrets:
      - name: myregistrykey
      containers:
      - name: web
        image: registry.example.com/team/app:v1.0
        imagePullPolicy: IfNotPresent
```

# 🧩 kubelet 与 containerd 在镜像拉取中的角色区别
🎯 最重要的结论（先说答案）
**绝大多数情况下：你应该配置 *containerd* 的 registry，而不是 kubelet。**

原因很简单：
- **真正拉镜像的是 containerd（CRI runtime）**
- kubelet 只是告诉 containerd “去拉这个镜像”
- kubelet 的 registry 配置能力非常有限（几乎不推荐使用）
- containerd 的 registry 配置能力强、稳定、可控、可全局生效

## 🟦 containerd：真正执行镜像拉取的组件

containerd 负责：
- 解析 registry 地址  
- 处理 TLS / insecure registry  
- 处理 registry mirror  
- 处理认证（auth）  
- 拉取镜像  
- 存储镜像  
- 解压镜像  
- 提供镜像给 kubelet  

**所以 registry 配置应该放在 containerd。**
## 🟧 kubelet：只是调用 containerd 的“调度者”
kubelet 负责：
- 读取 PodSpec 中的 image 字段  
- 调用 CRI（containerd）执行拉取  
- 处理 imagePullPolicy  
- 处理 imagePullSecrets（传给 containerd）  

但 kubelet **不负责镜像拉取逻辑本身**。

## 🥇 为什么应该配置 containerd 的 registry（强烈推荐）

| 能力 | kubelet | containerd |
|------|---------|------------|
| 配置 registry mirror | ❌ 不支持 | ✔ 完整支持 |
| 配置 insecure registry | ❌ 不支持 | ✔ 支持 |
| 配置 per-registry TLS | ❌ 不支持 | ✔ 支持（certs.d） |
| 配置 registry 认证 | ❌ 不支持 | ✔ 支持 |
| 控制镜像拉取行为 | ❌ 弱 | ✔ 强 |
| 企业生产环境可用性 | ❌ 不推荐 | ✔ 标准做法 |

**所以企业级集群必须配置 containerd，而不是 kubelet。**

## 🧭 containerd 配置 registry（标准方式）

### 1）编辑 containerd 配置
```
/etc/containerd/config.toml
```

#### 添加 registry mirror
```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.example.com"]
  endpoint = ["https://registry.example.com"]
```

#### 配置 insecure registry
```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com".tls]
  insecure_skip_verify = true
```

#### 配置 per-registry TLS（certs.d）
```
/etc/containerd/certs.d/registry.example.com/hosts.toml
```

## 🧭 kubelet 的 registry 配置（为什么不推荐）

kubelet 的 registry 配置能力非常有限：
- 不能配置 mirror  
- 不能配置 TLS  
- 不能配置 insecure registry  
- 不能配置 per-registry 证书  
- 不能配置认证  

它能做的只有：
- imagePullPolicy  
- imagePullSecrets（传给 containerd）  

所以 kubelet **不是 registry 配置的正确位置**。

## 🧨 如果 kubelet 和 containerd 都配置了 registry，会怎样？

**containerd 的配置优先级更高。**

流程是：
```
kubelet → CRI → containerd → registry
```
所以最终生效的是 containerd 的 registry 配置。

## 🥇 最终最佳实践（企业级）

| 需求 | 应该配置在哪 |
|------|--------------|
| registry mirror | containerd |
| insecure registry | containerd |
| registry TLS | containerd |
| registry 认证 | containerd |
| imagePullSecrets | kubelet（PodSpec） |
| imagePullPolicy | kubelet（PodSpec） |

## 🎁 最终总结（你可以直接带到团队里）

> **镜像拉取是 containerd 的职责，registry 配置必须放在 containerd。  
> kubelet 只负责调度，不负责镜像拉取逻辑。**
