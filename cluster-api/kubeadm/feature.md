# Kubeadm 完整功能清单

## 📋 核心功能概述

kubeadm 是 Kubernetes 官方提供的集群引导工具，用于快速部署符合最佳实践的 Kubernetes 集群。

## 🚀 主要命令分类

### 1️⃣ **集群引导命令**

#### `kubeadm init`
**功能**：初始化 Kubernetes 控制平面节点

**主要功能**：
- ✅ 预检查系统环境（preflight）
- ✅ 生成必要的证书（CA、API Server、etcd 等）
- ✅ 生成 kubeconfig 文件
- ✅ 设置控制平面静态 Pod（kube-apiserver、kube-controller-manager、kube-scheduler）
- ✅ 初始化 etcd 集群或本地 etcd
- ✅ 上传配置到 ConfigMap
- ✅ 标记节点为 control-plane
- ✅ 安装 CNI 网络插件（可选）
- ✅ 生成加入集群的 token

**常用参数**：
```bash
--config                        # 使用配置文件
--apiserver-advertise-address   # API Server 广播地址
--image-repository              # 镜像仓库地址
--kubernetes-version            # Kubernetes 版本
--pod-network-cidr              # Pod 网络 CIDR
--service-cidr                  # Service 网络 CIDR
--control-plane-endpoint        # 控制平面端点（负载均衡地址）
--upload-certs                  # 上传证书到集群
--skip-certificate-key-print    # 跳过打印证书密钥
```

**执行阶段**：
1. `preflight` - 预检查
2. `certs` - 生成证书
3. `kubeconfig` - 生成 kubeconfig
4. `kubelet-start` - 启动 kubelet
5. `control-plane` - 部署控制平面组件
6. `etcd` - 初始化 etcd
7. `upload-config` - 上传配置
8. `upload-certs` - 上传证书
9. `mark-control-plane` - 标记控制平面节点
10. `bootstrap-token` - 生成引导 token
11. `kubelet-finalize` - 完成 kubelet 设置
12. `addon` - 安装插件

#### `kubeadm join`
**功能**：将节点加入 Kubernetes 集群

**主要功能**：
- ✅ 预检查系统环境
- ✅ 下载集群配置
- ✅ 下载必要的证书
- ✅ 加入控制平面（如果是 control-plane 节点）
- ✅ 加入工作节点
- ✅ 配置 kubelet

**常用参数**：
```bash
--token                         # 加入集群的 token
--discovery-token-ca-cert-hash  # CA 证书哈希验证
--control-plane                 # 加入为控制平面节点
--experimental-control-plane    # 实验性控制平面加入
--apiserver-advertise-address   # API Server 广播地址
--ignore-preflight-errors       # 忽略预检查错误
```

**执行阶段**：
1. `preflight` - 预检查
2. `control-plane-prepare` - 控制平面准备（仅 control-plane）
3. `kubelet-start` - 启动 kubelet
4. `control-plane-join` - 控制平面加入（仅 control-plane）

### 2️⃣ **集群管理命令**

#### `kubeadm reset`
**功能**：撤销 kubeadm init 或 kubeadm join 的操作

**主要功能**：
- ✅ 停止 kubelet 服务
- ✅ 删除静态 Pod 配置
- ✅ 清理 etcd 数据
- ✅ 删除证书和配置文件
- ✅ 清理 iptables/IPVS 规则
- ✅ 删除 CNI 配置

**常用参数**：
```bash
--force               # 强制重置
--cri-socket          # CRI socket 路径
--ignore-preflight-errors  # 忽略预检查错误
```

#### `kubeadm upgrade`
**功能**：升级 Kubernetes 集群到新版本

**子命令**：

##### `kubeadm upgrade plan`
- ✅ 检查可升级的版本
- ✅ 显示升级计划
- ✅ 验证当前集群状态

##### `kubeadm upgrade apply`
- ✅ 升级控制平面组件
- ✅ 更新 ConfigMap 配置
- ✅ 更新证书（如需要）
- ✅ 更新 kubelet 配置

**常用参数**：
```bash
--config              # 使用配置文件
--version             # 目标版本
--allow-unstable-upgrade  # 允许不稳定版本升级
--certificate-renewal     # 证书续期策略
--etcd-upgrade            # 升级 etcd
--patches                 # 补丁目录
```

### 3️⃣ **证书管理命令**

#### `kubeadm certs`
**功能**：管理 Kubernetes 集群证书

**子命令**：

##### `kubeadm certs list`
- ✅ 列出所有证书及其过期时间

##### `kubeadm certs check-expiration`
- ✅ 检查证书过期状态

##### `kubeadm certs renew`
- ✅ 续期证书
- ✅ 支持续期单个或所有证书

**可管理的证书**：
- `ca.crt` - CA 证书
- `apiserver.crt` - API Server 证书
- `apiserver-kubelet-client.crt` - API Server kubelet 客户端证书
- `front-proxy-ca.crt` - 前端代理 CA
- `front-proxy-client.crt` - 前端代理客户端证书
- `etcd-ca.crt` - etcd CA
- `etcd-server.crt` - etcd 服务端证书
- `etcd-peer.crt` - etcd 对等证书
- `etcd-healthcheck-client.crt` - etcd 健康检查客户端证书
- `apiserver-etcd-client.crt` - API Server etcd 客户端证书

**常用参数**：
```bash
--cert-dir            # 证书目录
--config              # 使用配置文件
```

### 4️⃣ **Token 管理命令**

#### `kubeadm token`
**功能**：管理用于加入集群的引导 token

**子命令**：

##### `kubeadm token create`
- ✅ 创建新的引导 token
- ✅ 设置过期时间
- ✅ 设置用途

##### `kubeadm token delete`
- ✅ 删除指定的 token

##### `kubeadm token list`
- ✅ 列出所有 token

##### `kubeadm token generate`
- ✅ 生成随机 token 值

**常用参数**：
```bash
--ttl                 # Token 生存时间
--usages              # Token 用途
--description         # Token 描述
--groups              # Token 关联的组
```

### 5️⃣ **配置管理命令**

#### `kubeadm config`
**功能**：管理 kubeadm 配置

**子命令**：

##### `kubeadm config print init-defaults`
- ✅ 打印默认初始化配置

##### `kubeadm config print join-defaults`
- ✅ 打印默认加入配置

##### `kubeadm config migrate`
- ✅ 迁移旧版本配置到新版本

##### `kubeadm config images list`
- ✅ 列出所需镜像列表

##### `kubeadm config images pull`
- ✅ 拉取所需镜像

**常用参数**：
```bash
--image-repository    # 镜像仓库
--kubernetes-version  # Kubernetes 版本
--cri-socket          # CRI socket
```

### 6️⃣ **版本信息命令**

#### `kubeadm version`
**功能**：查看 kubeadm 版本信息

**主要功能**：
- ✅ 显示 kubeadm 版本
- ✅ 显示 Git commit
- ✅ 显示 Go 版本
- ✅ 显示构建平台

**常用参数**：
```bash
-o short             # 短格式输出
-o yaml              # YAML 格式输出
-o json              # JSON 格式输出
```

### 7️⃣ **kubeconfig 管理命令**

#### `kubeadm kubeconfig`
**功能**：管理 kubeconfig 文件

**子命令**：

##### `kubeadm kubeconfig user`
- ✅ 为指定用户生成 kubeconfig
- ✅ 支持自定义证书和 API Server 地址

**常用参数**：
```bash
--client-certificate  # 客户端证书路径
--client-key          # 客户端私钥路径
--apiserver-advertise-address  # API Server 地址
--cluster-name        # 集群名称
```

### 8️⃣ **初始化阶段命令**

#### `kubeadm init phase`
**功能**：单独执行初始化的某个阶段（高级用法）

**可用阶段**：
- `preflight` - 预检查
- `certs` - 证书生成
- `kubeconfig` - kubeconfig 生成
- `kubelet-start` - kubelet 启动
- `control-plane` - 控制平面部署
- `etcd` - etcd 初始化
- `upload-config` - 配置上传
- `upload-certs` - 证书上传
- `mark-control-plane` - 标记控制平面
- `bootstrap-token` - 引导 token 生成
- `addon` - 插件安装

**用途**：
- ✅ 分步调试初始化过程
- ✅ 自定义初始化流程
- ✅ 故障排查

#### `kubeadm join phase`
**功能**：单独执行加入集群的某个阶段（高级用法）

**可用阶段**：
- `preflight` - 预检查
- `control-plane-prepare` - 控制平面准备
- `kubelet-start` - kubelet 启动
- `control-plane-join` - 控制平面加入

## 🔧 高级功能

### 1. **镜像管理**
```bash
# 列出所需镜像
kubeadm config images list

# 拉取镜像
kubeadm config images pull

# 指定镜像仓库
kubeadm config images pull --image-repository registry.example.com
```

### 2. **配置文件支持**
- ✅ 支持 YAML 配置文件
- ✅ 支持自定义所有参数
- ✅ 支持配置文件继承和覆盖

### 3. **插件支持**
- ✅ 自动安装 kube-dns/CoreDNS
- ✅ 支持自定义 CNI 插件

### 4. **高可用支持**
- ✅ 支持多控制平面节点
- ✅ 支持外部 etcd 集群
- ✅ 支持堆叠 etcd 模式
- ✅ 支持负载均衡器

### 5. **安全特性**
- ✅ 自动生成 CA 和证书
- ✅ 证书自动轮换
- ✅ Token 认证
- ✅ RBAC 默认启用
- ✅ 支持 TLS bootstrap

### 6. **预检查功能**
- ✅ 系统版本检查
- ✅ 内核参数检查
- ✅ 端口占用检查
- ✅ 依赖工具检查
- ✅ 内存和 CPU 检查
- ✅ 容器运行时检查

## 📊 功能对比表

| 功能 | kubeadm init | kubeadm join | kubeadm upgrade | kubeadm reset |
|------|--------------|--------------|-----------------|---------------|
| 预检查 | ✅ | ✅ | ✅ | ✅ |
| 证书管理 | ✅ | ✅ | ✅ | ✅ |
| 配置生成 | ✅ | ✅ | ✅ | ❌ |
| 镜像拉取 | ✅ | ❌ | ✅ | ❌ |
| 服务启动 | ✅ | ✅ | ✅ | ✅ |
| 清理功能 | ❌ | ❌ | ❌ | ✅ |

## 🎯 典型使用场景

### 场景 1：单节点集群
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

### 场景 2：高可用集群
```bash
# 第一个控制平面节点
kubeadm init --control-plane-endpoint="lb.example.com:6443" \
             --upload-certs \
             --pod-network-cidr=10.244.0.0/16

# 其他控制平面节点
kubeadm join lb.example.com:6443 --token <token> \
     --discovery-token-ca-cert-hash sha256:<hash> \
     --control-plane \
     --certificate-key <cert-key>

# 工作节点
kubeadm join lb.example.com:6443 --token <token> \
     --discovery-token-ca-cert-hash sha256:<hash>
```

### 场景 3：离线部署
```bash
# 提前拉取镜像
kubeadm config images pull --image-repository registry.local

# 使用本地镜像初始化
kubeadm init --image-repository registry.local
```

### 场景 4：证书续期
```bash
# 检查证书过期
kubeadm certs check-expiration

# 续期所有证书
kubeadm certs renew all

# 重启控制平面组件
kubectl rollout restart deployment -n kube-system
```

## 📝 总结

kubeadm 提供了完整的 Kubernetes 集群生命周期管理功能：

1. **集群引导**：init、join
2. **集群管理**：reset、upgrade
3. **证书管理**：certs
4. **Token 管理**：token
5. **配置管理**：config
6. **版本管理**：version
7. **kubeconfig 管理**：kubeconfig
8. **高级调试**：phase

它是生产环境部署 Kubernetes 的推荐工具，遵循 Kubernetes 官方最佳实践。
        
