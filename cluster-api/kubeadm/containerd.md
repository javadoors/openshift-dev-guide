
# Kubeadm 对 Containerd 的支持
## Kubeadm 对 Containerd 的支持
### 一、直接回答
**kubeadm 不直接安装 containerd，但完全支持配置和使用 containerd 作为容器运行时。**
### 二、Kubeadm 与 Containerd 的关系
```
┌─────────────────────────────────────────────────────────────────┐
│                    组件职责划分                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Kubeadm 职责:                                                 │
│   ├── 初始化 Kubernetes 控制平面                                │
│   ├── 生成证书和 kubeconfig                                    │
│   ├── 配置 kubelet 启动参数                                    │
│   ├── 配置容器运行时接口 (CRI)                                  │
│   └── ⚠️ 不负责安装容器运行时                                   │
│                                                                 │
│   Containerd 职责:                                              │
│   ├── 管理容器镜像                                              │
│   ├── 执行容器运行                                              │
│   ├── 提供 CRI 接口                                             │
│   └── ⚠️ 需要单独安装                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 三、Kubeadm 对 Containerd 的支持方式
#### 1. **配置文件生成**
```bash
# 生成默认的 containerd 配置
containerd config default > /etc/containerd/config.toml

# 或使用 kubeadm 配置
kubeadm config print init-defaults --component-configs KubeletConfiguration
```
#### 2. **Kubeadm 配置中指定 CRI**
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
```
#### 3. **Kubeadm Init 时指定配置**
```bash
kubeadm init --config=kubeadm-config.yaml
```
### 四、标准安装流程
```bash
# 1. 安装 containerd（kubeadm 不负责此步骤）
apt-get update
apt-get install -y containerd

# 2. 配置 containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 3. 重要：配置 systemd cgroup driver
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 4. 重启 containerd
systemctl restart containerd

# 5. 使用 kubeadm 初始化集群
kubeadm init
```
### 五、在 BKE 项目中的处理方式
查看 BKE 项目中是如何处理的：
```go
// 从之前的代码分析中，BKE 项目通过以下方式处理：

// 1. EnsureNodesEnv Phase - 准备节点环境
//    - 安装 containerd
//    - 配置 containerd

// 2. EnsureContainerdUpgrade Phase - Containerd 升级
//    - 升级 containerd 版本

// 3. 通过 Agent 命令执行安装
```
**BKE 的处理方式**：
- containerd 安装由 **Agent** 执行，不是 kubeadm
- kubeadm 只负责配置 kubelet 连接 containerd
### 六、版本兼容性
| Kubernetes 版本 | Containerd 版本要求 | 说明 |
|----------------|--------------------|------|
| v1.24+ | containerd v1.6+ | dockershim 已移除 |
| v1.26+ | containerd v1.6.9+ | 推荐版本 |
| v1.28+ | containerd v1.7+ | 推荐版本 |
### 七、总结
| 问题 | 答案 |
|------|------|
| kubeadm 能安装 containerd 吗？ | ❌ 不能，需要单独安装 |
| kubeadm 能配置 containerd 吗？ | ✅ 能，通过 CRI 配置 |
| kubeadm 能管理 containerd 吗？ | ❌ 不能，containerd 独立运行 |
| kubeadm 依赖 containerd 吗？ | ✅ 是，作为容器运行时 |

**结论**：kubeadm 与 containerd 是协作关系，kubeadm 负责配置 Kubernetes 组件使用 containerd，但 containerd 的安装和配置需要通过其他方式（如包管理器、Ansible、BKE Agent 等）完成。

