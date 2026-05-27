# 预检工具
在 **OpenShift 安装集群之前**，确实有预检工具和机制，用来确保环境满足安装要求，避免后续部署失败。它们主要分为以下几类：  

## 📌 OpenShift 安装前的预检工具与机制
- **openshift-install 内置检查**  
  - 当你运行 `openshift-install create cluster` 时，安装器会自动检查部分环境条件，例如：DNS 解析、云平台 API 权限、Pull Secret 是否有效。  
  - 如果条件不满足，会直接报错并阻止安装。  
- **oc adm diagnostics**  
  - 在 OpenShift 3.x 中提供的诊断工具，可以检查节点配置、网络、权限等。  
  - 在 OpenShift 4.x 中部分功能被替代，但仍有类似的健康检查命令。  
- **Preflight Checks**  
  - 在 OpenShift 4.x 安装文档中，官方推荐在安装前进行一系列手动或自动检查：  
    - 确认操作系统版本（RHEL CoreOS 或兼容版本）。  
    - 确认 CPU、内存、磁盘空间满足最低要求。  
    - 确认网络连通性、DNS、NTP 时钟同步。  
    - 确认防火墙和端口开放情况。  
- **Red Hat OpenShift Preflight CLI**  
  - 主要用于 **镜像和 Operator 的认证检查**，不是集群安装必需，但在 ISV/企业场景下常用。  

## ⚖️ 典型安装前检查流程
1. **环境检查**：操作系统、硬件资源、NTP、SELinux/AppArmor。  
2. **网络检查**：节点间连通性、DNS、端口、防火墙。  
3. **依赖检查**：Pull Secret、云平台 API 权限、镜像仓库可访问性。  
4. **安装器预检**：运行 `openshift-install`，自动验证配置文件和环境。  

## 🚀 总结
- OpenShift 在安装集群前有 **内置预检机制**（`openshift-install`）和 **诊断工具**（如 `oc adm diagnostics`）。  
- 官方文档也提供了 **Preflight Checks 列表**，涵盖环境、网络、依赖、安全等方面。  
- 在企业场景下，还可以结合 **Preflight CLI** 做镜像和 Operator 的合规检查。  


# OpenShift 安装前检查清单
下面给你整理一个 **OpenShift 安装前检查清单**，涵盖环境、网络、依赖、安全等方面，方便在部署前逐项确认。  

## 📌 OpenShift 安装前检查清单
| **检查类别** | **检查项** | **说明** |
|--------------|------------|----------|
| **操作系统与硬件** | OS 版本 | 确认使用 RHEL CoreOS 或兼容版本，版本需满足官方支持矩阵 |
| | CPU/内存 | 每个节点至少 4 vCPU、16GB RAM（生产环境更高） |
| | 磁盘空间 | 至少 120GB，可用空间满足 etcd 和容器存储需求 |
| | 时钟同步 | NTP 服务正常，避免 etcd 数据不一致 |
| **网络** | 节点连通性 | 节点间网络可达，ping/端口探测正常 |
| | DNS | 节点主机名解析正确，集群域名可解析 |
| | 防火墙/端口 | 必要端口开放（API Server、etcd、Ingress、NodePort 等） |
| | 负载均衡 | 控制面节点前需配置负载均衡器 |
| **依赖与配置** | Pull Secret | 有效的 Red Hat Pull Secret，用于镜像拉取 |
| | 云平台权限 | 在 AWS/Azure/GCP 等平台上，确保安装器有足够 API 权限 |
| | 镜像仓库 | 能访问 registry.redhat.io 和 quay.io |
| | kubeconfig | 安装器能正确生成并使用 kubeconfig |
| **安全与合规** | SELinux | 建议启用并配置为 enforcing 模式 |
| | TLS 证书 | 确认证书有效，避免 API Server 启动失败 |
| | 用户权限 | 安装用户需具备 root 或 sudo 权限 |
| **安装器预检** | 配置文件 | `install-config.yaml` 参数正确（域名、网络、平台） |
| | 安装器检查 | `openshift-install create cluster` 会自动验证配置和环境 |

## 🚀 总结
- **环境**：操作系统、硬件资源、时钟同步。  
- **网络**：节点连通性、DNS、防火墙、负载均衡。  
- **依赖**：Pull Secret、云平台权限、镜像仓库访问。  
- **安全**：SELinux、TLS、用户权限。  
- **安装器**：配置文件和自动预检。  

这个清单可以作为 **部署前的核对表**，逐项确认后再运行 `openshift-install`，能大幅减少安装失败的风险。  

# OpenShift/Kubernetes 安装前检查操作手册
面给你一个 **OpenShift/Kubernetes 安装前检查操作手册**，把每个检查项的具体命令列出来，方便直接执行验证。  

## 📌 安装前检查操作手册

### 1. **环境检查**
- 操作系统版本  
  ```bash
  cat /etc/os-release
  uname -r
  ```
- CPU/内存/磁盘  
  ```bash
  lscpu
  free -h
  df -h
  ```
- 时钟同步  
  ```bash
  ntpstat        # NTP 状态
  timedatectl    # 确认时区和同步状态
  ```

### 2. **网络检查**
- 节点连通性  
  ```bash
  ping <node-ip>
  nc -zv <node-ip> 6443   # 检查 API Server 端口
  ```
- DNS 解析  
  ```bash
  dig <cluster-domain>
  nslookup <cluster-domain>
  ```
- 防火墙/端口  
  ```bash
  firewall-cmd --list-all
  netstat -tulnp | grep LISTEN
  ```

### 3. **依赖检查**
- 容器运行时  
  ```bash
  systemctl status docker
  systemctl status containerd
  ```
- kubelet/kubeadm 版本  
  ```bash
  kubelet --version
  kubeadm version
  ```
- 镜像仓库访问  
  ```bash
  curl -I https://registry.redhat.io
  curl -I https://quay.io
  ```

### 4. **安全检查**
- SELinux 状态  
  ```bash
  getenforce
  sestatus
  ```
- TLS 证书有效性  
  ```bash
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
  ```
- 用户权限  
  ```bash
  id
  sudo -l
  ```

### 5. **安装器预检**
- 配置文件检查  
  ```bash
  cat install-config.yaml
  ```
- 安装器自动检查  
  ```bash
  ./openshift-install create cluster --log-level=debug
  ```

## 🚀 总结
这份操作手册提供了 **逐项检查命令**，涵盖环境、网络、依赖、安全和安装器预检。  
- **执行顺序**：先环境 → 网络 → 依赖 → 安全 → 安装器。  
- **结果分级**：✅ 正常、⚠️ 警告（可继续但需优化）、❌ 错误（必须修复）。  
