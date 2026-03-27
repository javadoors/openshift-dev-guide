# kubeadm

# Kubeadm 核心概念及其作用
## 一、Kubeadm 概述
Kubeadm 是 Kubernetes 官方提供的集群引导工具，用于快速部署符合最佳实践的 Kubernetes 集群。它遵循"声明式"和"不可变基础设施"原则。
```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubeadm 架构概览                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────┐        ┌─────────────────┐              │
│   │   kubeadm init  │        │ kubeadm join    │              │
│   │   (初始化Master) │        │ (加入节点)       │              │
│   └────────┬────────┘        └────────┬────────┘              │
│            │                          │                        │
│            ▼                          ▼                        │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              Control Plane (控制平面)                 │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│   │  │ kube-    │  │ kube-    │  │ kube-    │           │    │
│   │  │ apiserver│  │ scheduler│  │ controller│           │    │
│   │  │          │  │          │  │ -manager │           │    │
│   │  └──────────┘  └──────────┘  └──────────┘           │    │
│   │  ┌──────────┐  ┌──────────┐                         │    │
│   │  │   etcd   │  │ kube-    │                         │    │
│   │  │          │  │ proxy    │                         │    │
│   │  └──────────┘  └──────────┘                         │    │
│   └──────────────────────────────────────────────────────┘    │
│                          │                                     │
│                          ▼                                     │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              Worker Nodes (工作节点)                  │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│   │  │ kubelet  │  │ kube-    │  │ container│           │    │
│   │  │          │  │ proxy    │  │ runtime  │           │    │
│   │  └──────────┘  └──────────┘  └──────────┘           │    │
│   └──────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 二、核心概念详解
### 2.1 Bootstrap Token（引导令牌）
**作用**：用于新节点安全加入集群的临时认证凭证。
```go
// Token 格式: [a-z0-9]{6}.[a-z0-9]{16}
// 示例: abcdef.0123456789abcdef

type BootstrapToken struct {
    // Token ID (前6位)
    ID string `json:"id"`
    
    // Token Secret (后16位)
    Secret string `json:"secret"`
    
    // 过期时间
    Expires *metav1.Time `json:"expires,omitempty"`
    
    // 用途: signing, authentication
    Usages []string `json:"usages,omitempty"`
    
    // 关联的组
    Groups []string `json:"groups,omitempty"`
}

// Token 存储在 Secret 中
// Secret 名称: bootstrap-token-<token-id>
// Secret 命名空间: kube-system
```
**工作流程**：
```
┌──────────────┐     1. 生成Token      ┌──────────────┐
│   Master     │ ───────────────────▶  │   Secret     │
│              │                       │ (kube-system)│
└──────────────┘                       └──────────────┘
       │
       │ 2. 分发Token
       ▼
┌──────────────┐     3. 使用Token      ┌──────────────┐
│   Worker     │ ───────────────────▶  │ kube-apiserver│
│              │     认证请求          │              │
└──────────────┘                       └──────────────┘
       │                                      │
       │ 4. 获取CSR                           │
       ▼                                      ▼
┌──────────────┐     5. 自动批准CSR         ┌──────────────┐
│   kubelet    │ ◀───────────────────────  │  CSR Approver│
│              │     签发证书               │              │
└──────────────┘                           └──────────────┘
```
### 2.2 Certificate Authority (CA) 证书体系
**作用**：构建集群的信任链，确保组件间通信安全。
```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes 证书体系                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────────┐                          │
│                    │   Root CA       │                          │
│                    │  (ca.crt)       │                          │
│                    └────────┬────────┘                          │
│                             │                                   │
│         ┌───────────────────┼───────────────────┐              │
│         │                   │                   │              │
│         ▼                   ▼                   ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │ apiserver   │    │   etcd      │    │  front-proxy│        │
│  │ CA          │    │   CA        │    │  CA         │        │
│  │ (ca.crt)    │    │ (etcd-ca)   │    │(front-proxy-│        │
│  │             │    │             │    │ ca.crt)     │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                  │                │
│         ▼                  ▼                  ▼                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │• apiserver  │    │• server     │    │• client     │        │
│  │  cert       │    │  cert       │    │  cert       │        │
│  │• kubelet    │    │• peer       │    │             │        │
│  │  client     │    │  cert       │    │             │        │
│  │• admin      │    │• healthcheck│    │             │        │
│  │  cert       │    │  client     │    │             │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**证书清单**：
```bash
/etc/kubernetes/pki/
├── apiserver-etcd-client.crt      # apiserver访问etcd的客户端证书
├── apiserver-etcd-client.key
├── apiserver-kubelet-client.crt   # apiserver访问kubelet的客户端证书
├── apiserver-kubelet-client.key
├── apiserver.crt                  # apiserver服务端证书
├── apiserver.key
├── ca.crt                         # 集群根CA证书
├── ca.key
├── controller-manager.conf        # controller-manager kubeconfig
├── etcd/
│   ├── ca.crt                     # etcd CA证书
│   ├── ca.key
│   ├── healthcheck-client.crt     # etcd健康检查客户端证书
│   ├── healthcheck-client.key
│   ├── peer.crt                   # etcd对等通信证书
│   ├── peer.key
│   ├── server.crt                 # etcd服务端证书
│   └── server.key
├── front-proxy-ca.crt             # 前端代理CA证书
├── front-proxy-ca.key
├── front-proxy-client.crt         # 前端代理客户端证书
├── front-proxy-client.key
├── scheduler.conf                 # scheduler kubeconfig
└── sa.key                         # ServiceAccount签名密钥
    sa.pub                         # ServiceAccount验证公钥
```
### 2.3 Kubeadm Configuration (配置文件)
**作用**：声明式定义集群配置，支持高度定制化。
```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
metadata:
  name: config
spec:
  # 控制平面端点
  controlPlaneEndpoint: "lb.example.com:6443"
  
  # 网络配置
  networking:
    serviceSubnet: "10.96.0.0/12"
    podSubnet: "10.244.0.0/16"
    dnsDomain: "cluster.local"
  
  # Kubernetes 版本
  kubernetesVersion: "v1.28.0"
  
  # 镜像仓库
  imageRepository: "registry.k8s.io"
  
  # API Server 配置
  apiServer:
    extraArgs:
      authorization-mode: "Node,RBAC"
      enable-admission-plugins: "NodeRestriction"
    certSANs:
      - "kubernetes"
      - "kubernetes.default"
      - "kubernetes.default.svc"
      - "10.96.0.1"
      - "192.168.1.100"
    timeoutForControlPlane: 4m0s
  
  # Controller Manager 配置
  controllerManager:
    extraArgs:
      cluster-signing-cert-file: "/etc/kubernetes/pki/ca.crt"
      cluster-signing-key-file: "/etc/kubernetes/pki/ca.key"
  
  # Scheduler 配置
  scheduler:
    extraArgs:
      bind-address: "0.0.0.0"
  
  # Etcd 配置
  etcd:
    local:
      dataDir: "/var/lib/etcd"
      extraArgs:
        listen-client-urls: "https://127.0.0.1:2379"
        advertise-client-urls: "https://127.0.0.1:2379"

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
metadata:
  name: config
spec:
  # 引导Token
  bootstrapTokens:
    - token: "abcdef.0123456789abcdef"
      description: "kubeadm bootstrap token"
      ttl: "24h"
      usages:
        - signing
        - authentication
      groups:
        - system:bootstrappers:kubeadm:default-node-token
  
  # 节点注册配置
  nodeRegistration:
    name: "master-1"
    criSocket: "unix:///var/run/containerd/containerd.sock"
    taints:
      - key: "node-role.kubernetes.io/control-plane"
        effect: "NoSchedule"
    kubeletExtraArgs:
      cgroup-driver: "systemd"
  
  # 本地API Server端点
  localAPIEndpoint:
    advertiseAddress: "192.168.1.100"
    bindPort: 6443

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
metadata:
  name: config
spec:
  # 控制平面配置 (加入Master时需要)
  controlPlane:
    localAPIEndpoint:
      advertiseAddress: "192.168.1.101"
      bindPort: 6443
  
  # 发现配置
  discovery:
    bootstrapToken:
      apiServerEndpoint: "lb.example.com:6443"
      token: "abcdef.0123456789abcdef"
      unsafeSkipCAVerification: false
      caCertHashes:
        - "sha256:abc123..."
    timeout: 5m0s
  
  # 节点注册配置
  nodeRegistration:
    name: "worker-1"
    criSocket: "unix:///var/run/containerd/containerd.sock"
    taints: []
    kubeletExtraArgs:
      cgroup-driver: "systemd"
```
### 2.4 Control Plane（控制平面）
**作用**：管理集群状态，调度工作负载。
```
┌─────────────────────────────────────────────────────────────────┐
│                    Control Plane 组件                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    kube-apiserver                          │ │
│  │  • 集群统一入口                                           │ │
│  │  • RESTful API                                           │ │
│  │  • 认证、授权、准入控制                                   │ │
│  │  • 唯一直接访问etcd的组件                                 │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌────────────────────┐  ┌────────────────────┐               │
│  │  kube-scheduler    │  │ kube-controller-   │               │
│  │                    │  │ manager            │               │
│  │  • Pod调度         │  │                    │               │
│  │  • 资源分配        │  │  • 节点控制器       │               │
│  │  • 亲和性/反亲和性 │  │  • 副本控制器       │               │
│  │  • 污点容忍        │  │  • 端点控制器       │               │
│  │                    │  │  • 服务账号控制器   │               │
│  └────────────────────┘  └────────────────────┘               │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                         etcd                               │ │
│  │  • 分布式键值存储                                         │ │
│  │  • 集群状态存储                                           │ │
│  │  • 配置数据                                               │ │
│  │  • Raft一致性协议                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.5 Kubeconfig
**作用**：客户端访问集群的认证配置文件。
```go
// Kubeconfig 结构
type Config struct {
    // 当前上下文
    CurrentContext string `json:"current-context"`
    
    // 集群列表
    Clusters []NamedCluster `json:"clusters"`
    
    // 用户列表
    AuthInfos []NamedAuthInfo `json:"users"`
    
    // 上下文列表
    Contexts []NamedContext `json:"contexts"`
}

type Cluster struct {
    // API Server 地址
    Server string `json:"server"`
    
    // CA 证书
    CertificateAuthorityData []byte `json:"certificate-authority-data"`
    
    // 跳过TLS验证
    InsecureSkipTLSVerify bool `json:"insecure-skip-tls-verify"`
}

type AuthInfo struct {
    // 客户端证书
    ClientCertificateData []byte `json:"client-certificate-data"`
    ClientKeyData []byte `json:"client-key-data"`
    
    // 或使用Token
    Token string `json:"token"`
}

type Context struct {
    Cluster  string `json:"cluster"`
    AuthInfo string `json:"user"`
    Namespace string `json:"namespace,omitempty"`
}
```
**Kubeconfig 示例**：
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...
    server: https://192.168.1.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTi...
    client-key-data: LS0tLS1CRUdJTi...
```
### 2.6 CSR (Certificate Signing Request)
**作用**：节点自动申请证书的机制。
```
┌─────────────────────────────────────────────────────────────────┐
│                    CSR 自动批准流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                              │
│  │   kubelet    │  1. 生成私钥                                 │
│  │   (启动)     │                                              │
│  └──────┬───────┘                                              │
│         │                                                       │
│         │ 2. 创建CSR                                            │
│         ▼                                                       │
│  ┌──────────────┐      ┌──────────────┐                        │
│  │     CSR      │ ───▶ │ kube-apiserver│                       │
│  │  (Pending)   │      │              │                        │
│  └──────────────┘      └──────────────┘                        │
│         │                       │                               │
│         │                       ▼                               │
│         │              ┌──────────────┐                        │
│         │              │  csrapprover │  3. 自动批准           │
│         │              │  controller  │     (验证Token)        │
│         │              └──────────────┘                        │
│         │                       │                               │
│         │ 4. 获取签名证书       │                               │
│         ▼                       ▼                               │
│  ┌──────────────┐      ┌──────────────┐                        │
│  │   kubelet    │ ◀─── │  Signed      │                        │
│  │   (Ready)    │      │  Certificate │                        │
│  └──────────────┘      └──────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
```go
// CSR 对象示例
type CertificateSigningRequest struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec CertificateSigningRequestSpec `json:"spec,omitempty"`
    Status CertificateSigningRequestStatus `json:"status,omitempty"`
}

type CertificateSigningRequestSpec struct {
    // CSR 数据 (PEM格式)
    Request []byte `json:"request"`
    
    // 签名者
    SignerName string `json:"signerName"`
    
    // 用途
    Usages []KeyUsage `json:"usages,omitempty"`
    
    // 过期时间
    ExpirationSeconds *int32 `json:"expirationSeconds,omitempty"`
}
```
## 三、Kubeadm 工作流程
### 3.1 kubeadm init 流程
```
┌─────────────────────────────────────────────────────────────────┐
│                    kubeadm init 执行流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Preflight Checks (预检查)                                  │
│     ├─ 检查系统要求 (内存、CPU、内核版本)                       │
│     ├─ 检查端口占用 (6443, 2379-2380, 10250-10252)             │
│     ├─ 检查容器运行时                                 │
│     ├─ 检查已有配置/证书                                       │
│     └─ 检查网络连通性                                          │
│                                                                 │
│  2. Generate Certificates (生成证书)                           │
│     ├─ 生成CA证书                               │
│     ├─ 生成API Server证书                                      │
│     ├─ 生成etcd证书                                            │
│     ├─ 生成ServiceAccount密钥对                                │
│     └─ 生成前端代理证书                                        │
│                                                                 │
│  3. Generate Kubeconfig Files (生成配置文件)                   │
│     ├─ admin.conf (管理员)                                     │
│     ├─ controller-manager.conf                                 │
│     ├─ scheduler.conf                                          │
│     └─ kubelet.conf                                            │
│                                                                 │
│  4. Generate Static Pod Manifests (生成静态Pod清单)            │
│     ├─ /etc/kubernetes/manifests/kube-apiserver.yaml           │
│     ├─ /etc/kubernetes/manifests/kube-controller-manager.yaml  │
│     ├─ /etc/kubernetes/manifests/kube-scheduler.yaml           │
│     └─ /etc/kubernetes/manifests/etcd.yaml                     │
│                                                                 │
│  5. Wait for Control Plane (等待控制平面就绪)                  │
│     └─ 轮询检查API Server健康状态                              │
│                                                                 │
│  6. Upload Configs (上传配置到ConfigMap)                       │
│     ├─ kubeadm-config ConfigMap                                │
│     └─ kube-root-ca.crt ConfigMap                              │
│                                                                 │
│  7. Generate Bootstrap Token (生成引导Token)                   │
│     └─ 创建bootstrap-token Secret                              │
│                                                                 │
│  8. Install Addons (安装插件)                                  │
│     ├─ kube-proxy (DaemonSet)                                  │
│     └─ CoreDNS (Deployment)                                    │
│                                                                 │
│  9. Output Join Command (输出加入命令)                         │
│     └─ kubeadm join <endpoint> --token <token> --discovery...  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 3.2 kubeadm join 流程
```
┌─────────────────────────────────────────────────────────────────┐
│                    kubeadm join 执行流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Preflight Checks (预检查)                                  │
│     ├─ 检查系统要求                                            │
│     ├─ 检查容器运行时                                          │
│     └─ 检查端口占用                                            │
│                                                                 │
│  2. Discovery (发现集群)                                       │
│     ├─ 使用Token连接API Server                                 │
│     ├─ 验证CA证书哈希                                          │
│     └─ 获取集群配置                                            │
│                                                                 │
│  3. TLS Bootstrap (TLS引导)                                    │
│     ├─ 生成临时私钥                                            │
│     ├─ 创建CSR请求                                             │
│     ├─ 等待CSR批准                                             │
│     └─ 获取签名证书                                            │
│                                                                 │
│  4. Generate Kubeconfig (生成kubelet配置)                      │
│     └─ /etc/kubernetes/kubelet.conf                            │
│                                                                 │
│  5. Node Registration (节点注册)                               │
│     ├─ 创建Node对象                                            │
│     └─ 设置标签和污点                                          │
│                                                                 │
│  6. Start Kubelet (启动kubelet)                                │
│     └─ kubelet连接API Server并开始工作                         │
│                                                                 │
│  [如果是Control Plane节点]                                     │
│  7. Download Certificates (下载证书)                           │
│     └─ 从kubeadm-certs Secret获取                              │
│                                                                 │
│  8. Generate Static Pod Manifests                              │
│     └─ 生成控制平面组件的静态Pod                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 四、Kubeadm 与 Cluster API 的关系
```
┌─────────────────────────────────────────────────────────────────┐
│              Kubeadm 在 Cluster API 中的角色                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   Cluster API                              │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │ │
│  │  │  Cluster    │  │   Machine   │  │  MachineSet │       │ │
│  │  │    CRD      │  │    CRD      │  │     CRD     │       │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │ │
│  └───────────────────────────┬───────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Kubeadm Bootstrap Provider                    │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  KubeadmConfig                                       │ │ │
│  │  │  • 定义kubeadm配置                                   │ │ │
│  │  │  • 生成join/init脚本                                 │ │ │
│  │  │  • 管理证书分发                                       │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────┬───────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Infrastructure Provider                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │ │
│  │  │   AWS       │  │   vSphere   │  │  BareMetal  │       │ │
│  │  │  Provider   │  │  Provider   │  │  Provider   │       │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │ │
│  └───────────────────────────┬───────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Kubeadm                                 │ │
│  │  • 实际执行init/join命令                                  │ │
│  │  • 管理证书生命周期                                       │ │
│  │  • 配置控制平面组件                                       │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 五、Kubeadm 核心配置项
### 5.1 关键配置参数
```yaml
# InitConfiguration 关键参数
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
spec:
  # API Server 广播地址
  localAPIEndpoint:
    advertiseAddress: "192.168.1.100"
    bindPort: 6443
  
  # 节点注册
  nodeRegistration:
    # 容器运行时
    criSocket: "unix:///var/run/containerd/containerd.sock"
    # 节点名称
    name: "node-1"
    # 污点
    taints:
      - key: "node-role.kubernetes.io/control-plane"
        effect: "NoSchedule"
    # 忽略Preflight错误
    ignorePreflightErrors:
      - "SystemVerification"
      - "Mem"

---
# ClusterConfiguration 关键参数
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
spec:
  # 控制平面端点 (HA场景)
  controlPlaneEndpoint: "lb.example.com:6443"
  
  # 集群网络
  networking:
    # Service CIDR
    serviceSubnet: "10.96.0.0/12"
    # Pod CIDR
    podSubnet: "10.244.0.0/16"
    # DNS域名
    dnsDomain: "cluster.local"
  
  # Kubernetes版本
  kubernetesVersion: "v1.28.0"
  
  # 镜像仓库
  imageRepository: "registry.k8s.io"
  
  # 证书有效期
  certificatesDir: "/etc/kubernetes/pki"
  
  # etcd配置
  etcd:
    local:
      dataDir: "/var/lib/etcd"
      extraArgs:
        listen-client-urls: "https://127.0.0.1:2379"
```
### 5.2 常用命令
```bash
# 初始化集群
kubeadm init --config=kubeadm-config.yaml

# 初始化 (简单模式)
kubeadm init \
  --apiserver-advertise-address=192.168.1.100 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0

# 加入工作节点
kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# 加入控制平面节点
kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>

# 生成新的Token
kubeadm token create --print-join-command

# 上传证书 (用于多Master)
kubeadm init phase upload-certs --upload-certs

# 重置节点
kubeadm reset -f

# 查看镜像列表
kubeadm config images list

# 预拉取镜像
kubeadm config images pull

# 查看默认配置
kubeadm config print init-defaults
kubeadm config print join-defaults
```
## 六、总结
| 概念 | 作用 | 关键点 |
|------|------|--------|
| **Bootstrap Token** | 节点安全加入集群 | 临时认证、自动过期 |
| **CA证书体系** | 构建信任链 | 多层CA、双向认证 |
| **Kubeadm Config** | 声明式集群配置 | 可定制、可复用 |
| **Control Plane** | 集群大脑 | 高可用、静态Pod |
| **Kubeconfig** | 客户端认证 | 证书/Token双模式 |
| **CSR** | 自动证书管理 | 自动批准、安全引导 |

Kubeadm 是 Kubernetes 集群引导的事实标准，理解其核心概念对于构建和管理生产级 Kubernetes 集群至关重要。在 Cluster API 架构中，Kubeadm 作为 Bootstrap Provider 的核心实现，负责节点的初始化和集群的引导工作。

# Kubeadm 概要设计
## 一、设计目标与原则
### 1.1 设计目标
```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubeadm 设计目标                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 简单易用                                             │   │
│  │     • 最小化配置即可启动集群                             │   │
│  │     • 提供合理的默认值                                   │   │
│  │     • 一条命令完成初始化                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  2. 安全最佳实践                                         │   │
│  │     • 自动生成证书                                       │   │
│  │     • TLS 加密通信                                       │   │
│  │     • RBAC 默认启用                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  3. 可扩展性                                             │   │
│  │     • 支持自定义配置                                     │   │
│  │     • 插件化组件安装                                     │   │
│  │     • 支持多种基础设施                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  4. 生产就绪                                             │   │
│  │     • 高可用支持                                         │   │
│  │     • 版本升级路径                                       │   │
│  │     • 声明式配置管理                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 1.2 设计原则
| 原则 | 说明 |
|------|------|
| **声明式配置** | 使用 YAML 配置文件定义集群状态 |
| **不可变基础设施** | 已创建的资源不修改，重建代替修改 |
| **最小权限** | 组件仅拥有必要的权限 |
| **关注点分离** | kubeadm 只负责引导，不管理运行时 |
## 二、整体架构
### 2.1 架构概览
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubeadm 整体架构                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                        CLI Layer (命令行层)                        │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │ │
│  │  │  init   │  │  join   │  │  reset  │  │ upgrade │  │ config │ │ │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └───┬────┘ │ │
│  └───────┼────────────┼────────────┼────────────┼───────────┼───────┘ │
│          │            │            │            │           │         │
│          ▼            ▼            ▼            ▼           ▼         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                     Command Layer (命令层)                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                    Command Registry                          │ │ │
│  │  │  • 注册所有子命令                                            │ │ │
│  │  │  • 解析命令行参数                                            │ │ │
│  │  │  • 验证输入配置                                              │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                      Phase Layer (阶段层)                         │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │ │
│  │  │ Preflight  │─▶│ Certs      │─▶│ Kubeconfig │─▶│ Kubelet    │ │ │
│  │  │ (预检查)   │  │ (证书生成) │  │ (配置生成) │  │ Start      │ │ │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │ │
│  │         │                                                   │     │ │
│  │         ▼                                                   ▼     │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │ │
│  │  │ Control    │─▶│ Etcd       │─▶│ Upload     │─▶│ Addons     │ │ │
│  │  │ Plane      │  │ (本地/外部)│  │ Config     │  │ (插件)     │ │ │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Component Layer (组件层)                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │ │
│  │  │ CertManager │  │ Kubeconfig  │  │ StaticPod   │              │ │
│  │  │ (证书管理)  │  │ Generator   │  │ Manager     │              │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │ │
│  │  │ Token       │  │ Discovery   │  │ Bootstrap   │              │ │
│  │  │ Manager     │  │ Manager     │  │ Signer      │              │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Runtime Layer (运行时层)                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │ │
│  │  │ Container   │  │ Systemd     │  │ File        │              │ │
│  │  │ Runtime     │  │ Manager     │  │ Manager     │              │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 2.2 核心模块设计
```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubeadm 核心模块                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              1. Preflight Module (预检模块)              │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • 检查系统要求 (内核版本、内存、CPU)                ││   │
│  │  │ • 检查端口可用性                                    ││   │
│  │  │ • 检查容器运行时                                    ││   │
│  │  │ • 检查已有配置/证书                                 ││   │
│  │  │ • 检查网络连通性                                    ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              2. Certs Module (证书模块)                  │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • 生成CA证书                                        ││   │
│  │  │ • 生成组件证书                                      ││   │
│  │  │ • 生成ServiceAccount密钥对                          ││   │
│  │  │ • 管理证书有效期                                    ││   │
│  │  │ • 支持外部CA                                        ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           3. Kubeconfig Module (配置模块)               │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • 生成admin.conf                                   ││   │
│  │  │ • 生成controller-manager.conf                      ││   │
│  │  │ • 生成scheduler.conf                               ││   │
│  │  │ • 生成kubelet.conf                                 ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │          4. ControlPlane Module (控制平面模块)          │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • 生成静态Pod清单                                   ││   │
│  │  │ • 配置API Server                                   ││   │
│  │  │ • 配置Controller Manager                           ││   │
│  │  │ • 配置Scheduler                                    ││   │
│  │  │ • 等待控制平面就绪                                  ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              5. Etcd Module (Etcd模块)                   │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • 本地etcd静态Pod                                   ││   │
│  │  │ • 外部etcd配置                                      ││   │
│  │  │ • etcd证书管理                                      ││   │
│  │  │ • etcd健康检查                                      ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            6. Bootstrap Module (引导模块)               │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ 职责:                                               ││   │
│  │  │ • Token生成和管理                                   ││   │
│  │  │ • CSR自动批准                                       ││   │
│  │  │ • 节点发现机制                                      ││   │
│  │  │ • 证书分发                                          ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 三、核心流程设计
### 3.1 kubeadm init 流程设计
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    kubeadm init 详细流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 1: Preflight (预检查)                                      │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [preflight]                                                  │ │  │
│  │ │ ├─ RunInitMasterChecks()                                    │ │  │
│  │ │ │  ├─ CheckSystemRequirements()                             │ │  │
│  │ │ │  │  ├─ IsSupportedOS()                                    │ │  │
│  │ │ │  │  ├─ CheckKernelVersion()                               │ │  │
│  │ │ │  │  └─ CheckMemory/CPU()                                  │ │  │
│  │ │ │  ├─ CheckPorts()                                          │ │  │
│  │ │ │  │  ├─ 6443 (API Server)                                  │ │  │
│  │ │ │  │  ├─ 2379-2380 (etcd)                                   │ │  │
│  │ │ │  │  ├─ 10250-10252 (kubelet/controller/scheduler)         │ │  │
│  │ │ │  │  └─ 30000-32767 (NodePort)                             │ │  │
│  │ │ │  ├─ CheckContainerRuntime()                               │ │  │
│  │ │ │  │  ├─ IsDocker()                                         │ │  │
│  │ │ │  │  └─ IsContainerd()                                     │ │  │
│  │ │ │  └─ CheckExistingFiles()                                  │ │  │
│  │ │ │     ├─ /etc/kubernetes/pki                                │ │  │
│  │ │ │     └─ /etc/kubernetes/admin.conf                         │ │  │
│  │ │ └─ RunPullImages()                                          │ │  │
│  │ │    └─ PullControlPlaneImages()                              │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 2: Certs (证书生成)                                        │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [certs]                                                      │ │  │
│  │ │ ├─ CreatePKIDirs()                                          │ │  │
│  │ │ │  └─ /etc/kubernetes/pki                                   │ │  │
│  │ │ ├─ CreateCACertsAndKeyPairs()                               │ │  │
│  │ │ │  ├─ ca.crt/ca.key (根CA)                                  │ │  │
│  │ │ │  ├─ etcd/ca.crt/ca.key (etcd CA)                          │ │  │
│  │ │ │  └─ front-proxy-ca.crt/ca.key (前端代理CA)                │ │  │
│  │ │ ├─ CreateAPIServerCerts()                                   │ │  │
│  │ │ │  ├─ apiserver.crt/apiserver.key                           │ │  │
│  │ │ │  ├─ apiserver-kubelet-client.crt/key                      │ │  │
│  │ │ │  └─ apiserver-etcd-client.crt/key                         │ │  │
│  │ │ ├─ CreateEtcdCerts()                                        │ │  │
│  │ │ │  ├─ etcd/server.crt/key                                   │ │  │
│  │ │ │  ├─ etcd/peer.crt/key                                     │ │  │
│  │ │ │  └─ etcd/healthcheck-client.crt/key                       │ │  │
│  │ │ ├─ CreateServiceAccountKeyPair()                            │ │  │
│  │ │ │  ├─ sa.key (签名私钥)                                     │ │  │
│  │ │ │  └─ sa.pub (验证公钥)                                     │ │  │
│  │ │ └─ CreateFrontProxyCerts()                                  │ │  │
│  │ │    └─ front-proxy-client.crt/key                            │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 3: Kubeconfig (配置文件生成)                               │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [kubeconfig]                                                 │ │  │
│  │ │ ├─ CreateInitKubeconfigs()                                  │ │  │
│  │ │ │  ├─ admin.conf                                            │ │  │
│  │ │ │  │  ├─ Cluster: kubernetes                                │ │  │
│  │ │ │  │  ├─ User: kubernetes-admin                             │ │  │
│  │ │ │  │  └─ Context: kubernetes-admin@kubernetes               │ │  │
│  │ │ │  ├─ controller-manager.conf                               │ │  │
│  │ │ │  │  └─ User: system:kube-controller-manager               │ │  │
│  │ │ │  ├─ scheduler.conf                                        │ │  │
│  │ │ │  │  └─ User: system:kube-scheduler                        │ │  │
│  │ │ │  └─ kubelet.conf                                          │ │  │
│  │ │ │     └─ User: system:node:<nodename>                       │ │  │
│  │ │ └─ WriteKubeconfigFiles()                                   │ │  │
│  │ │    └─ /etc/kubernetes/*.conf                                │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 4: Kubelet-start (Kubelet启动)                             │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [kubelet-start]                                              │ │  │
│  │ │ ├─ WriteKubeletServiceFile()                                │ │  │
│  │ │ │  └─ /etc/systemd/system/kubelet.service.d/10-kubeadm.conf │ │  │
│  │ │ ├─ WriteKubeletConfigFile()                                 │ │  │
│  │ │ │  └─ /var/lib/kubelet/config.yaml                          │ │  │
│  │ │ └─ StartKubelet()                                           │ │  │
│  │ │    └─ systemctl enable --now kubelet                        │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 5: Control-plane (控制平面启动)                            │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [control-plane]                                              │ │  │
│  │ │ ├─ CreateStaticPodManifests()                               │ │  │
│  │ │ │  ├─ kube-apiserver.yaml                                   │ │  │
│  │ │ │  │  ├─ image: registry.k8s.io/kube-apiserver:v1.28.0      │ │  │
│  │ │ │  │  ├─ command: kube-apiserver                            │ │  │
│  │ │ │  │  │   ├─ --advertise-address=<ip>                       │ │  │
│  │ │ │  │  │   ├─ --secure-port=6443                             │ │  │
│  │ │ │  │  │   ├─ --etcd-servers=https://127.0.0.1:2379          │ │  │
│  │ │ │  │  │   ├─ --service-cluster-ip-range=10.96.0.0/12        │ │  │
│  │ │ │  │  │   └─ --authorization-mode=Node,RBAC                 │ │  │
│  │ │ │  │  └─ volumeMounts:                                      │ │  │
│  │ │ │  │     ├─ /etc/kubernetes/pki                             │ │  │
│  │ │ │  │     └─ /etc/kubernetes/controller-manager.conf         │ │  │
│  │ │ │  ├─ kube-controller-manager.yaml                          │ │  │
│  │ │ │  │  ├─ command: kube-controller-manager                   │ │  │
│  │ │ │  │  │   ├─ --cluster-signing-cert-file                    │ │  │
│  │ │ │  │  │   ├─ --cluster-signing-key-file                     │ │  │
│  │ │ │  │  │   └─ --use-service-account-credentials=true         │ │  │
│  │ │ │  │  └─ volumeMounts:                                      │ │  │
│  │ │ │  │     ├─ /etc/kubernetes/pki                             │ │  │
│  │ │ │  │     └─ /etc/kubernetes/controller-manager.conf         │ │  │
│  │ │ │  └─ kube-scheduler.yaml                                   │ │  │
│  │ │ │     ├─ command: kube-scheduler                            │ │  │
│  │ │ │     │  └─ --bind-address=0.0.0.0                          │ │  │
│  │ │ │     └─ volumeMounts:                                      │ │  │
│  │ │ │        └─ /etc/kubernetes/scheduler.conf                  │ │  │
│  │ │ └─ WaitForControlPlane()                                    │ │  │
│  │ │    └─ Wait for API Server /healthz                          │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 6: Etcd (Etcd启动)                                         │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [etcd]                                                       │ │  │
│  │ │ ├─ CreateLocalEtcdStaticPodManifest()                       │ │  │
│  │ │ │  └─ etcd.yaml                                              │ │  │
│  │ │ │     ├─ image: registry.k8s.io/etcd:3.5.9-0                │ │  │
│  │ │ │     ├─ command: etcd                                       │ │  │
│  │ │ │     │  ├─ --name=<hostname>                                │ │  │
│  │ │ │     │  ├─ --listen-client-urls=https://127.0.0.1:2379     │ │  │
│  │ │ │     │  ├─ --advertise-client-urls=https://<ip>:2379       │ │  │
│  │ │ │     │  └─ --peer-urls=https://<ip>:2380                    │ │  │
│  │ │ │     └─ volumeMounts:                                       │ │  │
│  │ │ │        ├─ /var/lib/etcd                                    │ │  │
│  │ │ │        └─ /etc/kubernetes/pki/etcd                         │ │  │
│  │ │ └─ WaitForEtcd()                                             │ │  │
│  │ │    └─ etcdctl endpoint health                                │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 7: Upload-config (上传配置)                                │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [upload-config]                                              │ │  │
│  │ │ ├─ CreateKubeadmConfigMap()                                 │ │  │
│  │ │ │  ├─ Name: kubeadm-config                                  │ │  │
│  │ │ │  └─ Namespace: kube-system                                │ │  │
│  │ │ │     ├─ ClusterConfiguration (YAML)                        │ │  │
│  │ │ │     └─ InitConfiguration (YAML)                           │ │  │
│  │ │ └─ CreateKubeRootCAConfigMap()                              │ │  │
│  │ │    └─ Name: kube-root-ca.crt                                │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 8: Bootstrap-token (引导Token)                             │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [bootstrap-token]                                            │ │  │
│  │ │ ├─ CreateBootstrapToken()                                   │ │  │
│  │ │ │  ├─ GenerateTokenID()                                     │ │  │
│  │ │ │  │  └─ <6字符>.<16字符>                                   │ │  │
│  │ │ │  └─ CreateBootstrapTokenSecret()                          │ │  │
│  │ │ │     ├─ Name: bootstrap-token-<token-id>                   │ │  │
│  │ │ │     ├─ Namespace: kube-system                             │ │  │
│  │ │ │     └─ Data:                                              │ │  │
│  │ │ │        ├─ token-id: <6字符>                               │ │  │
│  │ │ │        ├─ token-secret: <16字符>                          │ │  │
│  │ │ │        ├─ usage-bootstrap-authentication: "true"          │ │  │
│  │ │ │        ├─ usage-bootstrap-signing: "true"                 │ │  │
│  │ │ │        └─ auth-extra-groups: system:bootstrappers:...     │ │  │
│  │ │ └─ AllowBootstrapTokensPostCSRs()                           │ │  │
│  │ │    └─ Create RBAC for auto-approval                         │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 9: Kubelet-finalize (Kubelet完成)                          │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [kubelet-finalize]                                           │ │  │
│  │ │ └─ UpdateKubeletConfig()                                    │ │  │
│  │ │    └─ Switch from bootstrap to final kubeconfig             │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 10: Addons (插件安装)                                      │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [addons]                                                     │ │  │
│  │ │ ├─ CreateCoreDNS()                                          │ │  │
│  │ │ │  ├─ ServiceAccount: coredns                               │ │  │
│  │ │ │  ├─ ClusterRole: system:coredns                           │ │  │
│  │ │ │  ├─ ClusterRoleBinding: system:coredns                    │ │  │
│  │ │ │  ├─ ConfigMap: coredns                                    │ │  │
│  │ │ │  ├─ Deployment: coredns                                   │ │  │
│  │ │ │  └─ Service: kube-dns                                     │ │  │
│  │ │ └─ CreateKubeProxy()                                        │ │  │
│  │ │    ├─ ServiceAccount: kube-proxy                            │ │  │
│  │ │    ├─ ClusterRole: system:node-proxier                      │ │  │
│  │ │    ├─ ClusterRoleBinding: kube-proxy                        │ │  │
│  │ │    ├─ ConfigMap: kube-proxy                                 │ │  │
│  │ │    └─ DaemonSet: kube-proxy                                 │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 11: Show-join-command (显示加入命令)                       │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [show-join-command]                                          │ │  │
│  │ │ └─ PrintJoinCommand()                                       │ │  │
│  │ │    └─ kubeadm join <endpoint>:6443 --token <token> \        │ │  │
│  │ │         --discovery-token-ca-cert-hash sha256:<hash>         │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 3.2 kubeadm join 流程设计
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    kubeadm join 详细流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 1: Preflight (预检查)                                      │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [preflight]                                                  │ │  │
│  │ │ ├─ RunJoinNodeChecks()                                      │ │  │
│  │ │ │  ├─ CheckSystemRequirements()                             │ │  │
│  │ │ │  ├─ CheckPorts()                                          │ │  │
│  │ │ │  │  ├─ 10250 (kubelet)                                    │ │  │
│  │ │ │  │  └─ 30000-32767 (NodePort)                             │ │  │
│  │ │ │  ├─ CheckContainerRuntime()                               │ │  │
│  │ │ │  └─ CheckExistingFiles()                                  │ │  │
│  │ │ └─ RunPullImages()                                          │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 2: Control-plane-prepare (控制平面准备 - 仅Master节点)      │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [control-plane-prepare] (仅 --control-plane 时执行)          │ │  │
│  │ │ ├─ DownloadCerts()                                          │ │  │
│  │ │ │  └─ 从 kubeadm-certs Secret 下载证书                       │ │  │
│  │ │ ├─ CreatePKIDirs()                                          │ │  │
│  │ │ ├─ CreateCACertsAndKeyPairs()                               │ │  │
│  │ │ ├─ CreateAPIServerCerts()                                   │ │  │
│  │ │ ├─ CreateEtcdCerts()                                        │ │  │
│  │ │ └─ CreateKubeconfigs()                                      │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 3: Kubelet-start (Kubelet启动)                             │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [kubelet-start]                                              │ │  │
│  │ │ ├─ WriteKubeletServiceFile()                                │ │  │
│  │ │ ├─ WriteKubeletConfigFile()                                 │ │  │
│  │ │ └─ StartKubelet()                                           │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 4: Bootstrap (引导)                                        │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [bootstrap]                                                  │ │  │
│  │ │ ├─ Discovery()                                              │ │  │
│  │ │ │  ├─ 方式1: Token Discovery                                │ │  │
│  │ │ │  │  ├─ 连接 API Server                                    │ │  │
│  │ │ │  │  ├─ 验证 CA 证书哈希                                   │ │  │
│  │ │ │  │  └─ 获取集群信息                                       │ │  │
│  │ │ │  └─ 方式2: File Discovery                                 │ │  │
│  │ │ │     └─ 从本地文件读取集群信息                              │ │  │
│  │ │ ├─ TLSBootstrap()                                           │ │  │
│  │ │ │  ├─ GenerateKeyPair()                                     │ │  │
│  │ │ │  │  └─ 生成临时私钥                                       │ │  │
│  │ │ │  ├─ CreateCSR()                                           │ │  │
│  │ │ │  │  ├─ Subject: system:node:<hostname>                    │ │  │
│  │ │ │  │  ├─ SignerName: kubernetes.io/kube-apiserver-client    │ │  │
│  │ │ │  │  └─ Usages: client auth                                │ │  │
│  │ │ │  ├─ WaitForCSRApproval()                                  │ │  │
│  │ │ │  │  └─ 等待 CSR 状态变为 Approved                         │ │  │
│  │ │ │  └─ GetSignedCertificate()                                │ │  │
│  │ │ │     └─ 从 CSR.status.certificate 获取签名证书             │ │  │
│  │ │ └─ CreateKubeletKubeconfig()                                │ │  │
│  │ │    └─ 使用签名证书生成 kubelet.conf                          │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Phase 5: Control-plane-join (控制平面加入 - 仅Master节点)        │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ [control-plane-join] (仅 --control-plane 时执行)             │ │  │
│  │ │ ├─ UpdateEtcdLocalEndpoints()                               │ │  │
│  │ │ │  └─ 更新 etcd 成员列表                                    │ │  │
│  │ │ ├─ CreateStaticPodManifests()                               │ │  │
│  │ │ │  ├─ kube-apiserver.yaml                                   │ │  │
│  │ │ │  ├─ kube-controller-manager.yaml                          │ │  │
│  │ │ │  ├─ kube-scheduler.yaml                                   │ │  │
│  │ │ │  └─ etcd.yaml                                             │ │  │
│  │ │ └─ WaitForControlPlane()                                    │ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 四、核心数据结构设计
### 4.1 配置结构
```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubeadm 配置结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              InitConfiguration (初始化配置)              │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ type InitConfiguration struct {                     ││   │
│  │  │     // 引导Token配置                                ││   │
│  │  │     BootstrapTokens []BootstrapToken                ││   │
│  │  │                                                     ││   │
│  │  │     // 节点注册配置                                 ││   │
│  │  │     NodeRegistration NodeRegistrationOptions        ││   │
│  │  │                                                     ││   │
│  │  │     // 本地API端点                                  ││   │
│  │  │     LocalAPIEndpoint APIEndpoint                    ││   │
│  │  │                                                     ││   │
│  │  │     // 证书密钥 (多Master场景)                      ││   │
│  │  │     CertificateKey string                           ││   │
│  │  │ }                                                   ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           ClusterConfiguration (集群配置)               │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ type ClusterConfiguration struct {                  ││   │
│  │  │     // 控制平面端点                                 ││   │
│  │  │     ControlPlaneEndpoint string                     ││   │
│  │  │                                                     ││   │
│  │  │     // 网络配置                                     ││   │
│  │  │     Networking Networking                           ││   │
│  │  │      ├─ ServiceSubnet string                        ││   │
│  │  │      ├─ PodSubnet string                            ││   │
│  │  │      └─ DNSDomain string                            ││   │
│  │  │                                                     ││   │
│  │  │     // Kubernetes版本                               ││   │
│  │  │     KubernetesVersion string                        ││   │
│  │  │                                                     ││   │
│  │  │     // 镜像仓库                                     ││   │
│  │  │     ImageRepository string                          ││   │
│  │  │                                                     ││   │
│  │  │     // 证书目录                                     ││   │
│  │  │     CertificatesDir string                          ││   │
│  │  │                                                     ││   │
│  │  │     // 组件配置                                     ││   │
│  │  │     APIServer APIServer                             ││   │
│  │  │     ControllerManager ControlPlaneComponent         ││   │
│  │  │     Scheduler ControlPlaneComponent                 ││   │
│  │  │     Etcd Etcd                                       ││   │
│  │  │ }                                                   ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              JoinConfiguration (加入配置)               │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │ type JoinConfiguration struct {                     ││   │
│  │  │     // 控制平面配置 (Master节点)                    ││   │
│  │  │     ControlPlane JoinControlPlane                   ││   │
│  │  │                                                     ││   │
│  │  │     // 发现配置                                     ││   │
│  │  │     Discovery Discovery                             ││   │
│  │  │      ├─ BootstrapToken *BootstrapTokenDiscovery     ││   │
│  │  │      │   ├─ Token string                            ││   │
│  │  │      │   ├─ APIServerEndpoint string                ││   │
│  │  │      │   ├─ CACertHashes []string                   ││   │
│  │  │      │   └─ UnsafeSkipCAVerification bool           ││   │
│  │  │      └─ File *FileDiscovery                         ││   │
│  │  │                                                     ││   │
│  │  │     // 节点注册配置                                 ││   │
│  │  │     NodeRegistration NodeRegistrationOptions        ││   │
│  │  │ }                                                   ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 4.2 证书结构
```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubeadm 证书结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  /etc/kubernetes/pki/                                          │
│  ├── ca.crt                    # 集群根CA证书                   │
│  ├── ca.key                    # 集群根CA私钥                   │
│  │                                                             │
│  ├── apiserver.crt             # API Server服务端证书           │
│  ├── apiserver.key             # API Server服务端私钥           │
│  │   SANs:                                                     │
│  │   ├─ kubernetes                                             │
│  │   ├─ kubernetes.default                                     │
│  │   ├─ kubernetes.default.svc                                 │
│  │   ├─ kubernetes.default.svc.cluster.local                   │
│  │   ├─ 10.96.0.1 (Service IP)                                │
│  │   └─ <node-ip>                                              │
│  │                                                             │
│  ├── apiserver-kubelet-client.crt  # API Server访问Kubelet     │
│  ├── apiserver-kubelet-client.key  # 的客户端证书               │
│  │                                                             │
│  ├── apiserver-etcd-client.crt     # API Server访问etcd        │
│  ├── apiserver-etcd-client.key     # 的客户端证书               │
│  │                                                             │
│  ├── front-proxy-ca.crt        # 前端代理CA证书                 │
│  ├── front-proxy-ca.key        # 前端代理CA私钥                 │
│  ├── front-proxy-client.crt    # 前端代理客户端证书             │
│  ├── front-proxy-client.key    # 前端代理客户端私钥             │
│  │                                                             │
│  ├── sa.key                    # ServiceAccount签名私钥         │
│  ├── sa.pub                    # ServiceAccount验证公钥         │
│  │                                                             │
│  └── etcd/                     # etcd证书目录                   │
│      ├── ca.crt                # etcd CA证书                    │
│      ├── ca.key                # etcd CA私钥                    │
│      ├── server.crt            # etcd服务端证书                 │
│      ├── server.key            # etcd服务端私钥                 │
│      ├── peer.crt              # etcd对等通信证书               │
│      ├── peer.key              # etcd对等通信私钥               │
│      ├── healthcheck-client.crt  # etcd健康检查客户端证书       │
│      └── healthcheck-client.key  # etcd健康检查客户端私钥       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 五、关键设计模式
### 5.1 Phase 模式
```go
// Phase 接口定义
type Phase interface {
    // Name 返回阶段名称
    Name() string
    
    // Run 执行阶段逻辑
    Run(ctx context.Context) error
    
    // Dependencies 返回依赖的阶段
    Dependencies() []string
}

// PhaseRunner 阶段运行器
type PhaseRunner struct {
    phases []Phase
}

func (r *PhaseRunner) Run(ctx context.Context) error {
    for _, phase := range r.phases {
        fmt.Printf("[%s] Running...\n", phase.Name())
        
        if err := phase.Run(ctx); err != nil {
            return fmt.Errorf("phase %s failed: %w", phase.Name(), err)
        }
        
        fmt.Printf("[%s] Completed successfully\n", phase.Name())
    }
    return nil
}

// 使用示例
func NewInitRunner() *PhaseRunner {
    return &PhaseRunner{
        phases: []Phase{
            NewPreflightPhase(),
            NewCertsPhase(),
            NewKubeconfigPhase(),
            NewKubeletStartPhase(),
            NewControlPlanePhase(),
            NewEtcdPhase(),
            NewUploadConfigPhase(),
            NewBootstrapTokenPhase(),
            NewAddonsPhase(),
        },
    }
}
```
### 5.2 Builder 模式
```go
// StaticPodBuilder 静态Pod构建器
type StaticPodBuilder struct {
    manifest *v1.Pod
}

func NewStaticPodBuilder(name, image string) *StaticPodBuilder {
    return &StaticPodBuilder{
        manifest: &v1.Pod{
            TypeMeta: metav1.TypeMeta{
                APIVersion: "v1",
                Kind:       "Pod",
            },
            ObjectMeta: metav1.ObjectMeta{
                Name:      name,
                Namespace: metav1.NamespaceSystem,
                Labels: map[string]string{
                    "component": name,
                    "tier":      "control-plane",
                },
            },
            Spec: v1.PodSpec{
                Containers: []v1.Container{
                    {
                        Name:    name,
                        Image:   image,
                        Command: []string{},
                        VolumeMounts: []v1.VolumeMount{},
                    },
                },
                Volumes: []v1.Volume{},
            },
        },
    }
}

func (b *StaticPodBuilder) WithCommand(cmd []string) *StaticPodBuilder {
    b.manifest.Spec.Containers[0].Command = cmd
    return b
}

func (b *StaticPodBuilder) WithVolumeMount(name, path string, readOnly bool) *StaticPodBuilder {
    b.manifest.Spec.Containers[0].VolumeMounts = append(
        b.manifest.Spec.Containers[0].VolumeMounts,
        v1.VolumeMount{
            Name:      name,
            MountPath: path,
            ReadOnly:  readOnly,
        },
    )
    return b
}

func (b *StaticPodBuilder) WithVolume(name, hostPath string) *StaticPodBuilder {
    b.manifest.Spec.Volumes = append(
        b.manifest.Spec.Volumes,
        v1.Volume{
            Name: name,
            VolumeSource: v1.VolumeSource{
                HostPath: &v1.HostPathVolumeSource{
                    Path: hostPath,
                },
            },
        },
    )
    return b
}

func (b *StaticPodBuilder) Build() *v1.Pod {
    return b.manifest
}

// 使用示例
func BuildAPIServerPod(image string, cfg *ClusterConfiguration) *v1.Pod {
    return NewStaticPodBuilder("kube-apiserver", image).
        WithCommand([]string{
            "kube-apiserver",
            fmt.Sprintf("--advertise-address=%s", cfg.LocalAPIEndpoint.AdvertiseAddress),
            fmt.Sprintf("--secure-port=%d", cfg.LocalAPIEndpoint.BindPort),
            "--etcd-servers=https://127.0.0.1:2379",
            fmt.Sprintf("--service-cluster-ip-range=%s", cfg.Networking.ServiceSubnet),
            "--authorization-mode=Node,RBAC",
        }).
        WithVolume("pki", "/etc/kubernetes/pki", true).
        WithVolumeMount("pki", "/etc/kubernetes/pki", true).
        Build()
}
```
### 5.3 Factory 模式
```go
// CertFactory 证书工厂
type CertFactory interface {
    CreateCACert() (*x509.Certificate, *rsa.PrivateKey, error)
    CreateSignedCert(caCert *x509.Certificate, caKey *rsa.PrivateKey, 
        cfg *CertConfig) (*x509.Certificate, *rsa.PrivateKey, error)
}

type KubeadmCertFactory struct {
    certificatesDir string
}

func (f *KubeadmCertFactory) CreateCACert() (*x509.Certificate, *rsa.PrivateKey, error) {
    key, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        return nil, nil, err
    }
    
    cert := &x509.Certificate{
        SerialNumber: big.NewInt(1),
        Subject: pkix.Name{
            CommonName: "kubernetes",
        },
        NotBefore:             time.Now(),
        NotAfter:              time.Now().AddDate(10, 0, 0),
        IsCA:                  true,
        KeyUsage:              x509.KeyUsageCertSign | x509.KeyUsageCRLSign,
        BasicConstraintsValid: true,
    }
    
    certDER, err := x509.CreateCertificate(rand.Reader, cert, cert, &key.PublicKey, key)
    if err != nil {
        return nil, nil, err
    }
    
    cert, err = x509.ParseCertificate(certDER)
    if err != nil {
        return nil, nil, err
    }
    
    return cert, key, nil
}

type CertConfig struct {
    CommonName   string
    Organization []string
    DNSNames     []string
    IPAddresses  []net.IP
    KeyUsage     x509.KeyUsage
    ExtKeyUsage  []x509.ExtKeyUsage
}
```
## 六、高可用设计
### 6.1 高可用架构
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Kubeadm 高可用架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                        ┌─────────────┐                                 │
│                        │  负载均衡器  │                                 │
│                        │  (HAProxy/  │                                 │
│                        │   Keepalived)│                                │
│                        │ VIP: 10.0.0.100│                              │
│                        └──────┬──────┘                                 │
│                               │                                        │
│           ┌───────────────────┼───────────────────┐                   │
│           │                   │                   │                   │
│           ▼                   ▼                   ▼                   │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│  │   Master-1      │ │   Master-2      │ │   Master-3      │         │
│  │  10.0.0.101     │ │  10.0.0.102     │ │  10.0.0.103     │         │
│  │                 │ │                 │ │                 │         │
│  │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │         │
│  │  │API Server │  │ │  │API Server │  │ │  │API Server │  │         │
│  │  │  :6443    │  │ │  │  :6443    │  │ │  │  :6443    │  │         │
│  │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │         │
│  │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │         │
│  │  │   etcd    │◀─┼─┼─▶│   etcd    │◀─┼─┼─▶│   etcd    │  │         │
│  │  │  :2379    │  │ │  │  :2379    │  │ │  │  :2379    │  │         │
│  │  │  :2380    │  │ │  │  :2380    │  │ │  │  :2380    │  │         │
│  │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │         │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 6.2 高可用初始化流程
```bash
# 1. 第一个Master节点初始化
kubeadm init \
  --control-plane-endpoint "lb.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# 输出:
# You can now join any number of control-plane nodes by running:
# kubeadm join lb.example.com:6443 --token <token> \
#   --discovery-token-ca-cert-hash sha256:<hash> \
#   --control-plane --certificate-key <cert-key>

# 2. 其他Master节点加入
kubeadm join lb.example.com:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <cert-key>

# 3. Worker节点加入
kubeadm join lb.example.com:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```
## 七、升级设计
### 7.1 升级流程
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Kubeadm 升级流程                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 1. 升级 kubeadm                                                   │  │
│  │    └─ yum/apt install kubeadm-1.29.0                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 2. 验证升级计划                                                   │  │
│  │    └─ kubeadm upgrade plan                                       │  │
│  │       ├─ 检查当前版本                                            │  │
│  │       ├─ 检查可升级版本                                          │  │
│  │       └─ 显示升级计划                                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 3. 升级控制平面 (逐个节点)                                        │  │
│  │    └─ kubeadm upgrade apply v1.29.0                              │  │
│  │       ├─ 拉取新版本镜像                                          │  │
│  │       ├─ 更新证书 (如需要)                                       │  │
│  │       ├─ 更新静态Pod清单                                         │  │
│  │       ├─ 重启控制平面组件                                        │  │
│  │       └─ 等待组件就绪                                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 4. 升级 kubelet (逐个节点)                                        │  │
│  │    ├─ kubectl drain <node> --ignore-daemonsets                   │  │
│  │    ├─ yum/apt install kubelet-1.29.0                             │  │
│  │    ├─ systemctl restart kubelet                                  │  │
│  │    └─ kubectl uncordon <node>                                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 5. 验证升级结果                                                   │  │
│  │    └─ kubectl get nodes                                          │  │
│  │       └─ 确认所有节点版本正确                                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 八、总结
### 8.1 设计亮点
| 设计点 | 说明 |
|--------|------|
| **Phase 模式** | 将复杂流程分解为独立阶段，支持单独执行和断点续传 |
| **声明式配置** | 使用 YAML 配置文件，支持版本控制和复用 |
| **自动化证书** | 自动生成和管理证书，降低安全配置复杂度 |
| **静态 Pod** | 控制平面组件以静态 Pod 运行，无需额外管理 |
| **高可用支持** | 原生支持多 Master 高可用部署 |
| **平滑升级** | 支持滚动升级，最小化服务中断 |
### 8.2 与 Cluster API 的集成
```
┌─────────────────────────────────────────────────────────────────┐
│           Kubeadm 在 Cluster API 中的定位                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cluster API                                                   │
│  ├─ Cluster CRD (集群定义)                                     │
│  ├─ Machine CRD (机器定义)                                     │
│  └─ MachineSet CRD (机器集合)                                  │
│                                                                 │
│  Kubeadm Bootstrap Provider                                    │
│  ├─ KubeadmConfig CRD (Kubeadm配置)                           │
│  │   ├─ InitConfiguration                                     │
│  │   ├─ ClusterConfiguration                                  │
│  │   └─ JoinConfiguration                                     │
│  └─ KubeadmConfigTemplate (配置模板)                          │
│                                                                 │
│  Infrastructure Provider                                       │
│  ├─ 创建/销毁机器                                              │
│  └─ 提供机器信息 (IP、SSH等)                                   │
│                                                                 │
│  Kubeadm                                                       │
│  ├─ 执行 init/join 命令                                       │
│  └─ 配置节点加入集群                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
Kubeadm 作为 Kubernetes 集群引导的标准工具，其设计充分体现了"简单、安全、可扩展"的原则。通过 Phase 模式、声明式配置和自动化证书管理，极大地简化了集群部署的复杂度，同时为 Cluster API 等高级编排工具提供了可靠的底层支持。

# Kubeadm 详细设计
## 一、整体架构设计
### 1.1 设计理念
Kubeadm 的设计遵循以下核心理念：
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    Kubeadm 设计理念                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 关注点分离                             │   │
│  │     • Kubeadm 只负责引导集群，不管理运行时               │   │
│  │     • 节点运维由 kubelet 负责                            │   │
│  │     • 网络插件由 CNI 提供商负责                          │   │
│  │     • 存储插件由 CSI 提供商负责                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  2. 声明式配置                           │   │
│  │     • 使用 YAML 配置文件定义期望状态                     │   │
│  │     • 配置可版本控制、可复用                             │   │
│  │     • 支持配置合并和覆盖                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  3. 阶段化执行                           │   │
│  │     • 将复杂流程分解为独立阶段                           │   │
│  │     • 支持单独执行某个阶段                               │   │
│  │     • 支持断点续传和故障恢复                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  4. 安全默认值                           │   │
│  │     • 自动生成证书和密钥                                 │   │
│  │     • 启用 RBAC 授权                                    │   │
│  │     • 使用 TLS 加密通信                                  │   │
│  │     • 最小权限原则                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  5. 可扩展性                               │   │
│  │     • 支持自定义组件参数                                 │   │
│  │     • 支持外部 CA 和证书                                 │   │
│  │     • 支持外部 etcd 集群                                 │   │
│  │     • 支持自定义镜像仓库                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 1.2 分层架构
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubeadm 分层架构                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  第5层: 用户交互层                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  • 命令行界面                                          │ │
│  │  • 子命令分发                                      │ │
│  │  • 参数解析与验证                                        │ │
│  │  • 输出格式化与进度显示                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ▼                                    │
│  第4层: 工作流编排层                                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  • 阶段注册表                                         │ │
│  │  • 阶段依赖管理                                          │ │
│  │  • 执行顺序控制                                          │ │
│  │  • 状态持久化                                            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ▼                                    │
│  第3层: 业务逻辑层                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │ │
│  │  │ 证书管理器  │ │ 配置生成器  │ │ 清单生成器  │ │ Token管理器│ │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └────────────┘ │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │ │
│  │  │ 发现管理器  │ │ 引导签名器  │ │ 升级协调器  │ │ 重置清理器│ │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ▼                                    │
│  第2层: 资源访问层                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  • Kubernetes API 客户端                                          │ │
│  │  • 文件系统操作                                                    │ │
│  │  • Systemd 服务管理                                               │ │
│  │  • 容器运行时接口                                                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    ▼                                    │
│  第1层: 基础设施层                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  • 日志框架                                                        │ │
│  │  • 配置加载                                                        │ │
│  │  • 错误处理                                                        │ │
│  │  • 工具函数                                                        │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 二、核心模块详细设计
### 2.1 预检模块
职责说明：在执行任何操作前，验证系统环境是否满足要求，提前发现潜在问题。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    预检模块详细设计                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  检查类别                                                       │
│  ├─ 系统要求检查                                               │
│  │  ├─ 操作系统类型                                            │
│  │  │  • 支持: Ubuntu, CentOS, RHEL, Debian, Fedora           │
│  │  │  • 检测方式: 读取 /etc/os-release                        │
│  │  │                                                          │
│  │  ├─ 内核版本                                                │
│  │  │  • 最低要求: 3.10+                                       │
│  │  │  • 推荐版本: 4.x+                                        │
│  │  │  • 检测方式: uname -r                                    │
│  │  │                                                          │
│  │  ├─ 内存要求                                                │
│  │  │  • Master节点: 最低 2GB，推荐 4GB+                       │
│  │  │  • Worker节点: 最低 1GB，推荐 2GB+                       │
│  │  │  • 检测方式: 读取 /proc/meminfo                          │
│  │  │                                                          │
│  │  ├─ CPU要求                                                 │
│  │  │  • Master节点: 最低 2核                                  │
│  │  │  • Worker节点: 最低 1核                                  │
│  │  │  • 检测方式: 读取 /proc/cpuinfo                          │
│  │  │                                                          │
│  │  └─ 磁盘空间                                                │
│  │     • 最低要求: 10GB 可用空间                               │
│  │     • 检测方式: df -h /var, /etc, /opt                      │
│  │                                                             │
│  ├─ 网络检查                                                   │
│  │  ├─ 端口可用性                                              │
│  │  │  • API Server: 6443                                      │
│  │  │  • etcd: 2379 (客户端), 2380 (对等)                      │
│  │  │  • kubelet: 10250                                        │
│  │  │  • kube-scheduler: 10259                                 │
│  │  │  • kube-controller-manager: 10257                        │
│  │  │  • NodePort范围: 30000-32767                             │
│  │  │  • 检测方式: netstat/ss 命令                             │
│  │  │                                                          │
│  │  ├─ 网络连通性                                              │
│  │  │  • 检查默认网关可达                                      │
│  │  │  • 检查DNS解析能力                                       │
│  │  │  • 检查时间同步 (NTP)                                    │
│  │  │                                                          │
│  │  └─ IP地址唯一性                                            │
│  │     • 检查节点IP在网络中唯一                                │
│  │     • 避免IP冲突                                            │
│  │                                                             │
│  ├─ 容器运行时检查                                             │
│  │  ├─ 运行时检测                                              │
│  │  │  • Docker: 检查 /var/run/docker.sock                     │
│  │  │  • Containerd: 检查 /var/run/containerd/containerd.sock  │
│  │  │  • CRI-O: 检查 /var/run/crio/crio.sock                   │
│  │  │                                                          │
│  │  ├─ 运行时版本                                              │
│  │  │  • Docker: 18.09+ (已弃用)                               │
│  │  │  • Containerd: 1.4+                                      │
│  │  │  • CRI-O: 1.20+                                          │
│  │  │                                                          │
│  │  ├─ 运行时配置                                              │
│  │  │  • systemd cgroup driver                                 │
│  │  │  • CRI 配置正确性                                        │
│  │  │                                                          │
│  │  └─ 运行时状态                                              │
│  │     • 服务运行中                                            │
│  │     • 可以拉取镜像                                          │
│  │                                                             │
│  ├─ 已有配置检查                                               │
│  │  ├─ 证书目录                                                │
│  │  │  • /etc/kubernetes/pki                                   │
│  │  │  • 检查是否已存在证书文件                                │
│  │  │                                                          │
│  │  ├─ 配置文件                                                │
│  │  │  • /etc/kubernetes/admin.conf                            │
│  │  │  • /etc/kubernetes/kubelet.conf                          │
│  │  │                                                          │
│  │  └─ 清单目录                                                │
│  │     • /etc/kubernetes/manifests                             │
│  │     • 检查是否已有静态Pod                                   │
│  │                                                             │
│  └─ 特殊检查                                                   │
│     ├─ swap分区                                                │
│     │  • 必须禁用swap                                          │
│     │  • 检测方式: free -m 或 swapon --show                    │
│     │                                                          │
│     ├─ SELinux                                                 │
│     │  • 建议设置为 Permissive 或 Disabled                     │
│     │  • 检测方式: getenforce                                  │
│     │                                                          │
│     ├─ 防火墙                                                  │
│     │  • 建议开放必要端口或禁用                                │
│     │  • 检测方式: iptables/firewall-cmd                       │
│     │                                                          │
│     └─ 内核模块                                                │
│        • br_netfilter (网桥过滤)                               │
│        • overlay (OverlayFS)                                   │
│        • 检测方式: lsmod                                       │
│                                                                 │
│  检查结果处理                                                   │
│  ├─ 错误: 阻止继续执行，必须修复                               │
│  ├─ 警告: 提示用户，可忽略继续                                 │
│  └─ 忽略: 通过 --ignore-preflight-errors 指定                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.2 证书管理模块
职责说明：生成和管理 Kubernetes 集群所需的所有证书，构建安全的信任链。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    证书管理模块详细设计                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  证书层次结构                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      根CA层                              │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │           Kubernetes CA (ca.crt/ca.key)         │   │   │
│  │  │  • 作用: 签发所有Kubernetes组件证书             │   │   │
│  │  │  • 有效期: 10年                                 │   │   │
│  │  │  • 密钥长度: 2048位 RSA                         │   │   │
│  │  │  • 存储位置: /etc/kubernetes/pki/              │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                         │                               │   │
│  │         ┌───────────────┼───────────────┐              │   │
│  │         ▼               ▼               ▼              │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │   │
│  │  │  etcd CA    │ │ Front Proxy │ │ API Server  │      │   │
│  │  │             │ │     CA      │ │   Client    │      │   │
│  │  │ 签发etcd    │ │ 签发前端    │ │   证书      │      │   │
│  │  │ 相关证书    │ │ 代理证书    │ │             │      │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  证书类型详解                                                   │
│  ├─ 服务端证书                                                 │
│  │  ├─ kube-apiserver                                          │
│  │  │  • 文件: apiserver.crt/apiserver.key                     │
│  │  │  • 用途: API Server TLS服务端                             │
│  │  │  • SAN扩展:                                              │
│  │  │    - DNS: kubernetes, kubernetes.default                 │
│  │  │    - DNS: kubernetes.default.svc                         │
│  │  │    - DNS: kubernetes.default.svc.cluster.local           │
│  │  │    - IP: Service ClusterIP (10.96.0.1)                   │
│  │  │    - IP: 节点IP                                          │
│  │  │    - IP: 负载均衡器IP (HA场景)                           │
│  │  │                                                          │
│  │  ├─ etcd server                                             │
│  │  │  • 文件: etcd/server.crt/server.key                      │
│  │  │  • 用途: etcd服务端TLS                                    │
│  │  │  • SAN扩展: localhost, 节点IP                            │
│  │  │                                                          │
│  │  └─ etcd peer                                               │
│  │     • 文件: etcd/peer.crt/peer.key                          │
│  │     • 用途: etcd集群内部通信                                 │
│  │     • SAN扩展: localhost, 节点IP                            │
│  │                                                             │
│  ├─ 客户端证书                                                 │
│  │  ├─ apiserver-kubelet-client                                │
│  │  │  • 用途: API Server访问kubelet                           │
│  │  │  • 组织: system:masters                                  │
│  │  │  • 权限: 节点代理权限                                    │
│  │  │                                                          │
│  │  ├─ apiserver-etcd-client                                   │
│  │  │  • 用途: API Server访问etcd                              │
│  │  │  • 组织: 无                                              │
│  │  │  • 权限: etcd读写权限                                    │
│  │  │                                                          │
│  │  ├─ controller-manager                                      │
│  │  │  • 用途: Controller Manager访问API Server                │
│  │  │  • 组织: system:kube-controller-manager                  │
│  │  │                                                          │
│  │  ├─ scheduler                                               │
│  │  │  • 用途: Scheduler访问API Server                         │
│  │  │  • 组织: system:kube-scheduler                           │
│  │  │                                                          │
│  │  └─ admin                                                   │
│  │     • 用途: 管理员访问API Server                            │
│  │     • 组织: system:masters                                  │
│  │     • 权限: 集群管理员                                      │
│  │                                                             │
│  └─ 特殊密钥                                                   │
│     ├─ ServiceAccount密钥对                                    │
│     │  • sa.key: 签名私钥                                      │
│     │  • sa.pub: 验证公钥                                      │
│     │  • 用途: 签名ServiceAccount Token                        │
│     │  • 算法: RSA 2048位                                      │
│     │                                                          │
│     └─ front-proxy-client                                      │
│        • 用途: 前端代理客户端认证                              │
│        • 场景: kubectl proxy等代理访问                         │
│                                                                 │
│  证书生成流程                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 创建PKI目录                                          │   │
│  │     └─ mkdir -p /etc/kubernetes/pki/etcd                 │   │
│  │                                                          │   │
│  │  2. 生成根CA证书                                         │   │
│  │     ├─ 生成CA私钥 (2048位RSA)                            │   │
│  │     ├─ 创建CA证书模板                                    │   │
│  │     │  - CommonName: kubernetes                          │   │
│  │     │  - IsCA: true                                      │   │
│  │     │  - KeyUsage: CertSign, CRLSign                     │   │
│  │     └─ 自签名生成CA证书                                  │   │
│  │                                                          │   │
│  │  3. 生成组件证书                                         │   │
│  │     ├─ 生成组件私钥                                      │   │
│  │     ├─ 创建CSR (证书签名请求)                            │   │
│  │     │  - 设置Subject信息                                 │   │
│  │     │  - 设置SAN扩展                                     │   │
│  │     │  - 设置KeyUsage                                    │   │
│  │     └─ 使用CA签名生成证书                                │   │
│  │                                                          │   │
│  │  4. 验证证书                                             │   │
│  │     ├─ 检查证书链完整性                                  │   │
│  │     ├─ 检查证书有效期                                    │   │
│  │     └─ 检查密钥匹配                                      │   │
│  │                                                          │   │
│  │  5. 设置文件权限                                         │   │
│  │     ├─ 证书文件: 0644                                    │   │
│  │     └─ 私钥文件: 0600                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  外部CA支持                                                     │
│  ├─ 使用场景                                                   │
│  │  • 企业内部已有PKI基础设施                                 │
│  │  • 需要集中管理证书                                        │
│  │  • 合规性要求                                              │
│  │                                                             │
│  ├─ 配置方式                                                   │
│  │  • 提供外部CA证书: /etc/kubernetes/pki/ca.crt              │
│  │  • 不提供CA私钥: ca.key 不存在                             │
│  │  • kubeadm检测到外部CA后，跳过证书生成                     │
│  │  • 需要手动生成组件证书并放置到正确位置                    │
│  │                                                             │
│  └─ 证书签名请求 (CSR)                                         │
│     • kubeadm可生成CSR文件                                     │
│     • 使用外部CA签名后放回                                     │
│                                                                 │
│  证书轮换                                                       │
│  ├─ 自动轮换                                                   │
│  │  • kubelet客户端证书自动轮换                               │
│  │  • 到期前7天开始轮换                                       │
│  │  • 通过CSR机制申请新证书                                   │
│  │                                                             │
│  └─ 手动轮换                                                   │
│     • kubeadm certs renew 命令                                 │
│     • 支持指定证书类型或全部轮换                               │
│     • 轮换后需重启相关组件                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.3 配置生成模块
职责说明：生成 Kubernetes 组件所需的配置文件和清单文件。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    配置生成模块详细设计                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Kubeconfig文件生成                                             │
│  ├─ 文件类型                                                   │
│  │  ├─ admin.conf                                              │
│  │  │  • 用途: 集群管理员访问凭证                              │
│  │  │  • 用户: kubernetes-admin                                │
│  │  │  • 组织: system:masters                                  │
│  │  │  • 权限: 集群超级管理员                                  │
│  │  │  • 存放: /etc/kubernetes/admin.conf                      │
│  │  │                                                          │
│  │  ├─ controller-manager.conf                                 │
│  │  │  • 用途: Controller Manager访问API Server                │
│  │  │  • 用户: system:kube-controller-manager                  │
│  │  │  • 权限: 控制器所需权限                                  │
│  │  │  • 存放: /etc/kubernetes/controller-manager.conf         │
│  │  │                                                          │
│  │  ├─ scheduler.conf                                          │
│  │  │  • 用途: Scheduler访问API Server                         │
│  │  │  • 用户: system:kube-scheduler                           │
│  │  │  • 权限: 调度器所需权限                                  │
│  │  │  • 存放: /etc/kubernetes/scheduler.conf                  │
│  │  │                                                          │
│  │  └─ kubelet.conf                                            │
│  │     • 用途: Kubelet访问API Server                           │
│  │     • 用户: system:node:<hostname>                          │
│  │     • 权限: 节点所需权限                                    │
│  │     • 存放: /etc/kubernetes/kubelet.conf                    │
│  │                                                             │
│  ├─ 生成流程                                                   │
│  │  ├─ 1. 确定集群信息                                        │
│  │  │    • API Server地址 (单节点或负载均衡器)                 │
│  │  │    • 集群名称                                           │
│  │  │    • CA证书数据                                         │
│  │  │                                                          │
│  │  ├─ 2. 创建用户凭证                                        │
│  │  │    • 生成客户端证书                                     │
│  │  │    • 设置用户和组织信息                                 │
│  │  │                                                          │
│  │  ├─ 3. 构建kubeconfig结构                                  │
│  │  │    • Clusters部分: 集群信息                             │
│  │  │    • Users部分: 用户凭证                                │
│  │  │    • Contexts部分: 上下文绑定                           │
│  │  │                                                          │
│  │  └─ 4. 写入文件                                            │
│  │       • 设置文件权限: 0600                                 │
│  │       • 设置文件所有者: root:root                          │
│  │                                                             │
│  └─ 配置内容示例                                               │
│     • Cluster配置                                              │
│       - server: https://<endpoint>:6443                        │
│       - certificate-authority-data: <base64编码的CA证书>       │
│     • User配置                                                 │
│       - client-certificate-data: <base64编码的客户端证书>      │
│       - client-key-data: <base64编码的客户端私钥>              │
│     • Context配置                                              │
│       - cluster: kubernetes                                    │
│       - user: <username>                                       │
│                                                                 │
│  静态Pod清单生成                                                │
│  ├─ kube-apiserver.yaml                                        │
│  │  ├─ 基本信息                                                │
│  │  │  • 名称: kube-apiserver                                  │
│  │  │  • 命名空间: kube-system                                 │
│  │  │  • 标签: component=kube-apiserver, tier=control-plane    │
│  │  │                                                          │
│  │  ├─ 容器配置                                                │
│  │  │  • 镜像: registry.k8s.io/kube-apiserver:v1.28.0          │
│  │  │  • 命令参数:                                             │
│  │  │    --advertise-address=<ip>                              │
│  │  │    --secure-port=6443                                    │
│  │  │    --etcd-servers=https://127.0.0.1:2379                 │
│  │  │    --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt         │
│  │  │    --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd... │
│  │  │    --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd...  │
│  │  │    --service-cluster-ip-range=10.96.0.0/12               │
│  │  │    --service-account-key-file=/etc/kubernetes/pki/sa.pub │
│  │  │    --service-account-signing-key-file=.../sa.key         │
│  │  │    --service-account-issuer=https://kubernetes.default...│
│  │  │    --authorization-mode=Node,RBAC                        │
│  │  │    --client-ca-file=/etc/kubernetes/pki/ca.crt           │
│  │  │    --tls-cert-file=/etc/kubernetes/pki/apiserver.crt     │
│  │  │    --tls-private-key-file=/etc/kubernetes/pki/apiserver..│
│  │  │    --kubelet-client-certificate=.../apiserver-kubelet... │
│  │  │    --kubelet-client-key=.../apiserver-kubelet-client.key │
│  │  │    --enable-admission-plugins=NodeRestriction            │
│  │  │                                                          │
│  │  └─ 卷挂载                                                  │
│  │     • /etc/kubernetes/pki (只读)                            │
│  │     • /etc/kubernetes/admin.conf (只读)                     │
│  │                                                             │
│  ├─ kube-controller-manager.yaml                               │
│  │  ├─ 容器配置                                                │
│  │  │  • 镜像: registry.k8s.io/kube-controller-manager:v1.28.0 │
│  │  │  • 命令参数:                                             │
│  │  │    --bind-address=0.0.0.0                                │
│  │  │    --cluster-signing-cert-file=.../ca.crt                │
│  │  │    --cluster-signing-key-file=.../ca.key                 │
│  │  │    --use-service-account-credentials=true                │
│  │  │    --root-ca-file=/etc/kubernetes/pki/ca.crt             │
│  │  │    --service-account-private-key-file=.../sa.key         │
│  │  │    --leader-elect=true                                   │
│  │  │                                                          │
│  │  └─ 卷挂载                                                  │
│  │     • /etc/kubernetes/pki (只读)                            │
│  │     • /etc/kubernetes/controller-manager.conf (只读)        │
│  │                                                             │
│  ├─ kube-scheduler.yaml                                        │
│  │  ├─ 容器配置                                                │
│  │  │  • 镜像: registry.k8s.io/kube-scheduler:v1.28.0          │
│  │  │  • 命令参数:                                             │
│  │  │    --bind-address=0.0.0.0                                │
│  │  │    --leader-elect=true                                   │
│  │  │                                                          │
│  │  └─ 卷挂载                                                  │
│  │     • /etc/kubernetes/scheduler.conf (只读)                 │
│  │                                                             │
│  └─ etcd.yaml                                                  │
│     ├─ 容器配置                                                │
│     │  • 镜像: registry.k8s.io/etcd:3.5.9-0                    │
│     │  • 命令参数:                                             │
│     │    --name=<hostname>                                     │
│     │    --data-dir=/var/lib/etcd                              │
│     │    --listen-client-urls=https://127.0.0.1:2379           │
│     │    --advertise-client-urls=https://<ip>:2379             │
│     │    --listen-peer-urls=https://<ip>:2380                  │
│     │    --initial-advertise-peer-urls=https://<ip>:2380       │
│     │    --initial-cluster=<hostname>=https://<ip>:2380        │
│     │    --client-cert-auth=true                               │
│     │    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt     │
│     │    --cert-file=/etc/kubernetes/pki/etcd/server.crt       │
│     │    --key-file=/etc/kubernetes/pki/etcd/server.key        │
│     │    --peer-client-cert-auth=true                          │
│     │    --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca... │
│     │    --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt    │
│     │    --peer-key-file=/etc/kubernetes/pki/etcd/peer.key     │
│     │                                                          │
│     └─ 卷挂载                                                  │
│        • /var/lib/etcd (数据目录)                              │
│        • /etc/kubernetes/pki/etcd (证书目录)                   │
│                                                                 │
│  Kubelet配置生成                                               │
│  ├─ systemd服务文件                                            │
│  │  • 文件: /etc/systemd/system/kubelet.service.d/10-kubeadm...│
│  │  • 内容:                                                   │
│  │    - EnvironmentFile: /var/lib/kubelet/kubeadm-flags.env    │
│  │    - ExecStart: 添加额外参数                                │
│  │                                                             │
│  ├─ kubelet配置文件                                            │
│  │  • 文件: /var/lib/kubelet/config.yaml                       │
│  │  • 内容:                                                   │
│  │    - kind: KubeletConfiguration                             │
│  │    - apiVersion: kubelet.config.k8s.io/v1beta1              │
│  │    - cgroupDriver: systemd                                  │
│  │    - clusterDNS: [10.96.0.10]                               │
│  │    - clusterDomain: cluster.local                           │
│  │    - authentication:                                        │
│  │        anonymous:                                           │
│  │          enabled: false                                     │
│  │        webhook:                                             │
│  │          enabled: true                                      │
│  │    - authorization:                                         │
│  │        mode: Webhook                                        │
│  │    - serverTLSBootstrap: true                               │
│  │                                                             │
│  └─ 环境变量文件                                               │
│     • 文件: /var/lib/kubelet/kubeadm-flags.env                  │
│     • 内容:                                                   │
│       KUBELET_KUBEADM_ARGS="--container-runtime=remote         │
│         --container-runtime-endpoint=/run/containerd/          │
│         containerd.sock --pod-infra-container-image=..."       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.4 引导Token模块
职责说明：管理节点加入集群所需的临时认证凭证。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    引导Token模块详细设计                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Token结构                                                     │
│  ├─ 格式: [a-z0-9]{6}.[a-z0-9]{16}                             │
│  │  • 示例: abcdef.0123456789abcdef                            │
│  │  • 前6位: Token ID (公开，用于查找)                         │
│  │  • 后16位: Token Secret (私密，用于验证)                    │
│  │                                                             │
│  └─ 存储方式                                                   │
│     • Secret名称: bootstrap-token-<token-id>                   │
│     • 命名空间: kube-system                                    │
│     • 类型: bootstrap.kubernetes.io/token                      │
│                                                                 │
│  Token属性                                                     │
│  ├─ token-id                                                   │
│  │  • Token的唯一标识符                                        │
│  │  • 6位小写字母或数字                                        │
│  │                                                             │
│  ├─ token-secret                                               │
│  │  • Token的密钥部分                                          │
│  │  • 16位小写字母或数字                                       │
│  │  • 用于验证Token有效性                                      │
│  │                                                             │
│  ├─ expiration (可选)                                          │
│  │  • Token过期时间                                            │
│  │  • 默认: 24小时                                             │
│  │  • 格式: RFC3339时间戳                                      │
│  │                                                             │
│  ├─ usage-bootstrap-authentication                             │
│  │  • 是否用于认证                                             │
│  │  • 值: "true"                                               │
│  │                                                             │
│  ├─ usage-bootstrap-signing                                    │
│  │  • 是否用于签名                                             │
│  │  • 值: "true"                                               │
│  │                                                             │
│  ├─ auth-extra-groups                                          │
│  │  • Token关联的额外组                                        │
│  │  • 值: system:bootstrappers:kubeadm:default-node-token      │
│  │                                                             │
│  └─ description                                                │
│     • Token描述信息                                            │
│     • 用于标识Token用途                                        │
│                                                                 │
│  Token生命周期                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  创建阶段                                                 │   │
│  │  ├─ 1. 生成Token ID (6位随机字符)                        │   │
│  │  ├─ 2. 生成Token Secret (16位随机字符)                   │   │
│  │  ├─ 3. 创建Secret对象                                    │   │
│  │  │    - 设置token-id                                     │   │
│  │  │    - 设置token-secret                                 │   │
│  │  │    - 设置过期时间                                     │   │
│  │  │    - 设置用途                                         │   │
│  │  │    - 设置关联组                                       │   │
│  │  └─ 4. 应用Secret到集群                                  │   │
│  │                                                          │   │
│  │  使用阶段                                                 │   │
│  │  ├─ 1. 节点使用Token连接API Server                       │   │
│  │  │    - Authorization: Bearer <token>                    │   │
│  │  ├─ 2. API Server验证Token                               │   │
│  │  │    - 查找对应Secret                                   │   │
│  │  │    - 验证Secret匹配                                   │   │
│  │  │    - 检查是否过期                                     │   │
│  │  ├─ 3. 返回临时凭证                                      │   │
│  │  │    - 允许创建CSR                                      │   │
│  │  │    - 获取集群信息                                     │   │
│  │  └─ 4. 节点获取正式证书                                  │   │
│  │                                                          │   │
│  │  清理阶段                                                 │   │
│  │  ├─ 自动清理                                              │   │
│  │  │  • Token过期后自动删除                                │   │
│  │  │  • 由Token Cleaner控制器执行                          │   │
│  │  │                                                        │   │
│  │  └─ 手动清理                                              │   │
│  │     • kubeadm token delete <token-id>                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Token管理命令                                                 │
│  ├─ 创建Token                                                 │
│  │  • kubeadm token create                                    │
│  │    - 生成新Token并创建Secret                               │
│  │    - 默认有效期24小时                                      │
│  │    - 可指定有效期: --ttl <duration>                        │
│  │    - 可指定用途: --usages <usages>                         │
│  │    - 可指定组: --groups <groups>                           │
│  │                                                             │
│  │  • kubeadm token create --print-join-command               │
│  │    - 生成Token并打印完整的join命令                         │
│  │    - 方便用户直接复制使用                                  │
│  │                                                             │
│  ├─ 列出Token                                                 │
│  │  • kubeadm token list                                      │
│  │    - 显示所有有效Token                                     │
│  │    - 包含ID、TTL、过期时间、用途等信息                     │
│  │                                                             │
│  └─ 删除Token                                                 │
│     • kubeadm token delete <token-id>                          │
│       - 删除指定Token                                          │
│       - 立即生效                                               │
│                                                                 │
│  RBAC配置                                                     │
│  ├─ Token认证权限                                             │
│  │  • ClusterRole: system:node-bootstrapper                   │
│  │    - 允许创建CSR                                           │
│  │                                                             │
│  │  • ClusterRoleBinding: kubeadm:kubelet-bootstrap           │
│  │    - 绑定到system:bootstrappers组                          │
│  │                                                             │
│  ├─ CSR自动批准权限                                           │
│  │  • ClusterRole: system:certificates.k8s.io:certificatesig..│
│  │    - 允许批准CSR                                           │
│  │                                                             │
│  │  • ClusterRoleBinding: kubeadm:node-autoapprove-bootstrap  │
│  │    - 自动批准初始CSR                                       │
│  │                                                             │
│  │  • ClusterRoleBinding: kubeadm:node-autoapprove-certificate│
│  │    - 自动批准证书轮换CSR                                   │
│  │                                                             │
│  └─ 自动批准控制器                                            │
│     • kube-controller-manager中的csrapproving控制器            │
│     • 自动批准来自bootstrap token的CSR                         │
│     • 自动批准kubelet证书轮换CSR                               │
│                                                                 │
│  安全考虑                                                     │
│  ├─ Token安全性                                               │
│  │  • Token Secret应妥善保管                                  │
│  │  • 避免在日志中打印完整Token                               │
│  │  • 设置合理的过期时间                                      │
│  │                                                             │
│  ├─ 最小权限原则                                              │
│  │  • Token仅用于初始认证                                     │
│  │  • 获取正式证书后Token失效                                 │
│  │  • 不应长期使用Token                                       │
│  │                                                             │
│  └─ 审计跟踪                                                  │
│     • 记录Token使用情况                                        │
│     • 监控异常Token使用                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.5 发现模块
职责说明：帮助新节点发现集群并验证集群身份。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    发现模块详细设计                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  发现方式                                                      │
│  ├─ Token Discovery (推荐)                                     │
│  │  ├─ 工作原理                                               │
│  │  │  1. 使用Token连接API Server                             │
│  │  │  2. 获取集群信息ConfigMap                               │
│  │  │  3. 验证CA证书哈希                                      │
│  │  │  4. 获取加入所需信息                                    │
│  │  │                                                          │
│  │  ├─ 配置参数                                               │
│  │  │  • apiServerEndpoint: API Server地址                    │
│  │  │    格式: <host>:<port>                                  │
│  │  │    示例: 192.168.1.100:6443                             │
│  │  │                                                          │
│  │  │  • token: Bootstrap Token                               │
│  │  │    格式: <id>.<secret>                                  │
│  │  │                                                          │
│  │  │  • caCertHashes: CA证书哈希                              │
│  │  │    格式: sha256:<hash>                                  │
│  │  │    作用: 验证API Server身份                             │
│  │  │                                                          │
│  │  │  • unsafeSkipCAVerification: 跳过CA验证                  │
│  │  │    不推荐: 存在中间人攻击风险                            │
│  │  │                                                          │
│  │  └─ 示例命令                                               │
│  │     kubeadm join 192.168.1.100:6443 \                      │
│  │       --token abcdef.0123456789abcdef \                    │
│  │       --discovery-token-ca-cert-hash \                     │
│  │         sha256:1234567890abcdef...                         │
│  │                                                             │
│  ├─ File Discovery                                             │
│  │  ├─ 工作原理                                               │
│  │  │  1. 从本地文件读取集群信息                              │
│  │  │  2. 文件包含kubeconfig格式配置                           │
│  │  │  3. 使用文件中的凭证连接集群                            │
│  │  │                                                          │
│  │  ├─ 配置参数                                               │
│  │  │  • file: 集群信息文件路径                               │
│  │  │    格式: /path/to/cluster-info.yaml                     │
│  │  │                                                          │
│  │  └─ 文件内容                                               │
│  │     • 包含集群CA证书                                       │
│  │     • 包含API Server地址                                   │
│  │     • 可选包含认证凭证                                     │
│  │                                                             │
│  └─ 配置文件Discovery                                          │
│     • 通过kubeadm配置文件指定发现信息                          │
│     • 支持更复杂的发现配置                                    │
│                                                                 │
│  发现流程详解                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Step 1: 连接API Server                                  │   │
│  │  ├─ 使用Token或文件凭证                                  │   │
│  │  ├─ 建立HTTPS连接                                        │   │
│  │  └─ 处理连接错误                                         │   │
│  │    - 超时处理                                            │   │
│  │    - 重试机制                                            │   │
│  │    - 网络问题诊断                                        │   │
│  │                                                          │   │
│  │  Step 2: 获取集群信息                                    │   │
│  │  ├─ 读取cluster-info ConfigMap                           │   │
│  │  │  命名空间: kube-public                                │   │
│  │  │  名称: cluster-info                                   │   │
│  │  │                                                        │   │
│  │  ├─ ConfigMap内容:                                       │   │
│  │  │  kubeconfig: |                                        │   │
│  │  │    apiVersion: v1                                     │   │
│  │  │    clusters:                                          │   │
│  │  │    - cluster:                                         │   │
│  │  │        certificate-authority-data: <ca-cert>          │   │
│  │  │        server: https://<api-server>:6443              │   │
│  │  │      name: ""                                         │   │
│  │  │                                                        │   │
│  │  └─ 解析kubeconfig获取:                                  │   │
│  │     • API Server地址                                     │   │
│  │     • CA证书                                             │   │
│  │                                                          │   │
│  │  Step 3: 验证CA证书                                      │   │
│  │  ├─ 计算CA证书SHA256哈希                                 │   │
│  │  ├─ 与提供的哈希值比较                                   │   │
│  │  │                                                        │   │
│  │  ├─ 验证通过:                                            │   │
│  │  │  • 继续加入流程                                       │   │
│  │  │                                                        │   │
│  │  └─ 验证失败:                                            │   │
│  │     • 拒绝连接                                           │   │
│  │     • 提示可能的安全风险                                 │   │
│  │     • 显示期望哈希与实际哈希                             │   │
│  │                                                          │   │
│  │  Step 4: 获取加入配置                                    │   │
│  │  ├─ 读取kubeadm-config ConfigMap                         │   │
│  │  ├─ 获取ClusterConfiguration                             │   │
│  │  │  • 网络配置                                           │   │
│  │  │  • Kubernetes版本                                     │   │
│  │  │  • 镜像仓库                                           │   │
│  │  │                                                        │   │
│  │  └─ 准备加入所需信息                                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CA证书哈希计算                                                │
│  ├─ 计算方式                                                   │
│  │  • 对CA证书进行SHA256哈希                                  │
│  │  • 格式: sha256:<hex-encoded-hash>                         │
│  │                                                             │
│  ├─ 获取哈希                                                   │
│  │  • 方式1: kubeadm init输出中包含                           │
│  │  • 方式2: kubeadm token create --print-join-command        │
│  │  • 方式3: 手动计算                                         │
│  │    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \ │
│  │    openssl rsa -pubin -outform der 2>/dev/null | \         │
│  │    openssl dgst -sha256 -hex | sed 's/^.* //'              │
│  │                                                             │
│  └─ 验证目的                                                   │
│     • 防止中间人攻击                                          │
│     • 确保连接到正确的集群                                    │
│     • 防止加入恶意集群                                        │
│                                                                 │
│  错误处理                                                      │
│  ├─ 连接失败                                                   │
│  │  • 检查网络连通性                                          │
│  │  • 检查API Server地址和端口                                │
│  │  • 检查防火墙规则                                          │
│  │                                                             │
│  ├─ Token无效                                                  │
│  │  • Token已过期                                             │
│  │  • Token不存在                                             │
│  │  • Token Secret不匹配                                      │
│  │                                                             │
│  ├─ CA验证失败                                                 │
│  │  • 哈希不匹配                                              │
│  │  • 证书被篡改                                              │
│  │  • 连接到错误的集群                                        │
│  │                                                             │
│  └─ 超时处理                                                   │
│     • 默认超时: 5分钟                                          │
│     • 可通过配置调整                                          │
│     • 提供重试机制                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### 2.6 TLS Bootstrap模块
职责说明：实现节点安全引导和证书自动签发机制。
```plainText
┌─────────────────────────────────────────────────────────────────┐
│                    TLS Bootstrap模块详细设计                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  核心概念                                                      │
│  ├─ TLS Bootstrap                                              │
│  │  • 目的: 让节点自动获取Kubernetes客户端证书                 │
│  │  • 原理: 使用临时Token认证，申请正式证书                    │
│  │  • 安全: 证书由CA签名，Token仅用于初始认证                 │
│  │                                                             │
│  ├─ CSR (Certificate Signing Request)                          │
│  │  • 证书签名请求                                            │
│  │  • 包含节点信息和公钥                                      │
│  │  • 由Kubernetes CA签名后成为有效证书                       │
│  │                                                             │
│  └─ 自动批准                                                   │
│     • kube-controller-manager自动批准特定CSR                   │
│     • 基于预设规则和RBAC权限                                   │
│                                                                 │
│  TLS Bootstrap流程                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Phase 1: 初始认证                                       │   │
│  │  ├─ 1. kubelet启动                                       │   │
│  │  │    - 读取bootstrap kubeconfig                         │   │
│  │  │    - 包含Token和API Server地址                        │   │
│  │  │                                                        │   │
│  │  ├─ 2. 使用Token认证                                     │   │
│  │  │    - 向API Server发送请求                             │   │
│  │  │    - Authorization: Bearer <token>                    │   │
│  │  │                                                        │   │
│  │  └─ 3. 获取临时访问权限                                  │   │
│  │     - 权限: 创建CSR                                      │   │
│  │     - 组: system:bootstrappers                           │   │
│  │                                                          │   │
│  │  Phase 2: 证书申请                                       │   │
│  │  ├─ 1. 生成密钥对                                        │   │
│  │  │    - 生成RSA私钥 (2048位)                             │   │
│  │  │    - 提取公钥                                         │   │
│  │  │                                                        │   │
│  │  ├─ 2. 创建CSR                                           │   │
│  │  │    - Subject:                                         │   │
│  │  │      CN: system:node:<hostname>                       │   │
│  │  │      O: system:nodes                                  │   │
│  │  │    - SignerName:                                      │   │
│  │  │      kubernetes.io/kube-apiserver-client              │   │
│  │  │    - Usages:                                          │   │
│  │  │      - client auth                                    │   │
│  │  │    - ExpirationSeconds:                               │   │
│  │  │      86400 (1年)                                      │   │
│  │  │                                                        │   │
│  │  └─ 3. 提交CSR到API Server                               │   │
│  │     - 创建CertificateSigningRequest对象                  │   │
│  │     - 状态: Pending                                      │   │
│  │                                                          │   │
│  │  Phase 3: CSR批准                                        │   │
│  │  ├─ 1. 自动批准检查                                      │   │
│  │  │    - 检查请求者身份                                   │   │
│  │  │    - 检查CSR内容                                      │   │
│  │  │    - 检查权限                                         │   │
│  │  │                                                        │   │
│  │  ├─ 2. 批准条件                                          │   │
│  │  │    - 请求者属于system:bootstrappers组                 │   │
│  │  │    - Subject CN格式正确                               │   │
│  │  │    - SignerName正确                                   │   │
│  │  │    - 请求的权限合理                                   │   │
│  │  │                                                        │   │
│  │  └─ 3. 更新CSR状态                                       │   │
│  │     - 添加批准条件                                       │   │
│  │     - 状态: Approved                                     │   │
│  │                                                          │   │
│  │  Phase 4: 证书签发                                       │   │
│  │  ├─ 1. kube-controller-manager签发证书                   │   │
│  │  │    - 使用集群CA私钥签名                               │   │
│  │  │    - 设置证书有效期                                   │   │
│  │  │    - 设置证书用途                                     │   │
│  │  │                                                        │   │
│  │  └─ 2. 更新CSR状态                                       │   │
│  │     - status.certificate: <signed-cert>                  │   │
│  │                                                          │   │
│  │  Phase 5: 证书获取                                       │   │
│  │  ├─ 1. kubelet轮询CSR状态                                │   │
│  │  │    - 检查certificate字段                              │   │
│  │  │                                                        │   │
│  │  ├─ 2. 获取签名证书                                      │   │
│  │  │    - 从CSR.status.certificate提取                     │   │
│  │  │                                                        │   │
│  │  └─ 3. 生成kubelet kubeconfig                            │   │
│  │     - 写入证书文件                                       │   │
│  │     - 创建kubeconfig                                     │   │
│  │     - 切换到正式凭证                                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CSR类型                                                       │
│  ├─ 初始CSR                                                    │
│  │  • 用途: 节点首次加入集群                                  │
│  │  • SignerName: kubernetes.io/kube-apiserver-client         │
│  │  • 自动批准: 是                                            │
│  │                                                             │
│  ├─ kubelet服务端CSR                                           │
│  │  • 用途: kubelet TLS服务端证书                             │
│  │  • SignerName: kubernetes.io/kubelet-serving               │
│  │  • 自动批准: 是 (需启用serverTLSBootstrap)                 │
│  │                                                             │
│  └─ 证书轮换CSR                                                │
│     • 用途: 证书到期前轮换                                     │
│     • SignerName: kubernetes.io/kube-apiserver-client          │
│     • 自动批准: 是                                             │
│                                                                 │
│  自动批准控制器                                                │
│  ├─ csrapproving控制器                                         │
│  │  • 位置: kube-controller-manager                           │
│  │  • 功能: 自动批准符合条件的CSR                             │
│  │                                                             │
│  ├─ 批准规则                                                   │
│  │  ├─ 初始CSR批准                                            │
│  │  │  • ClusterRoleBinding: node-autoapprove-bootstrap       │
│  │  │  • 条件:                                               │
│  │  │    - 请求者: system:bootstrappers组                     │
│  │  │    - Subject CN: system:node:*                          │
│  │  │    - SignerName: kubernetes.io/kube-apiserver-client    │
│  │  │                                                          │
│  │  ├─ 服务端证书批准                                         │
│  │  │  • ClusterRoleBinding: node-server-autoapprove          │
│  │  │  • 条件:                                               │
│  │  │    - 请求者: system:nodes组                             │
│  │  │    - SignerName: kubernetes.io/kubelet-serving          │
│  │  │                                                          │
│  │  └─ 证书轮换批准                                           │
│  │     • ClusterRoleBinding: node-autoapprove-certificate-... │
│  │     • 条件:                                                │
│  │       - 请求者: system:nodes组                             │
│  │       - 用于证书续期                                       │
│  │                                                             │
│  └─ 手动批准                                                   │
│     • kubectl certificate approve <csr-name>                   │
│     • 用于特殊情况或自定义审批流程                            │
│                                                                 │
│  证书轮换机制                                                  │
│  ├─ 客户端证书轮换                                             │
│  │  • 触发条件: 证书到期前7天                                  │
│  │  • 流程:                                                   │
│  │    1. 生成新密钥对                                         │
│  │    2. 创建新CSR                                            │
│  │    3. 等待批准                                             │
│  │    4. 获取新证书                                           │
│  │    5. 更新kubeconfig                                       │
│  │    6. 重启kubelet                                          │
│  │                                                             │
│  ├─ 服务端证书轮换                                             │
│  │  • 触发条件: 证书到期前7天                                  │
│  │  • 自动创建CSR并批准                                       │
│  │  • 无需重启kubelet                                         │
│  │                                                             │
│  └─ 配置参数                                                   │
│     • rotateCertificates: true (默认启用)                      │
│     • serverTLSBootstrap: true (服务端证书引导)                │
│                                                                 │
│  安全考虑                                                      │
│  ├─ Token保护                                                  │
│  │  • Token仅用于初始认证                                     │
│  │  • 获取证书后Token失效                                     │
│  │  • Token有效期应设置较短                                   │
│  │                                                             │
│  ├─ CSR验证                                                    │
│  │  • 验证请求者身份                                          │
│  │  • 验证Subject格式                                         │
│  │  • 验证请求权限                                            │
│  │                                                             │
│  ├─ 证书有效期                                                 │
│  │  • 客户端证书: 1年                                         │
│  │  • 服务端证书: 1年                                         │
│  │  • 自动轮换避免过期                                        │
│  │                                                             │
│  └─ 审计日志                                                   │
│     • 记录CSR创建和批准                                        │
│     • 记录证书签发                                            │
│     • 便于安全审计                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 三、高可用设计详解
### 3.1 高可用架构
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    Kubeadm 高可用架构详解                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  架构组件                                                               │
│  ├─ 负载均衡器                                                         │
│  │  ├─ HAProxy                                                        │
│  │  │  • 作用: 四层负载均衡，分发API Server流量                        │
│  │  │  • 配置:                                                        │
│  │  │    frontend k8s-api                                            │
│  │  │      bind *:6443                                               │
│  │  │      mode tcp                                                  │
│  │  │      default_backend k8s-api-nodes                            │
│  │  │    backend k8s-api-nodes                                       │
│  │  │      mode tcp                                                  │
│  │  │      balance roundrobin                                        │
│  │  │      option tcp-check                                          │
│  │  │      server master1 10.0.0.101:6443 check                     │
│  │  │      server master2 10.0.0.102:6443 check                     │
│  │  │      server master3 10.0.0.103:6443 check                     │
│  │  │                                                                 │
│  │  └─ Keepalived                                                    │
│  │     • 作用: 提供虚拟IP (VIP)，实现负载均衡器高可用                 │
│  │     • 配置:                                                        │
│  │       vrrp_instance VI_1 {                                        │
│  │         state MASTER                                              │
│  │         interface eth0                                            │
│  │         virtual_router_id 51                                      │
│  │         priority 100                                              │
│  │         virtual_ipaddress {                                       │
│  │           10.0.0.100                                              │
│  │         }                                                         │
│  │       }                                                           │
│  │                                                                    │
│  ├─ 控制平面节点                                                      │
│  │  ├─ Master节点数量: 奇数个 (推荐3或5个)                           │
│  │  │  • 3节点: 容忍1节点故障                                        │
│  │  │  • 5节点: 容忍2节点故障                                        │
│  │  │                                                                 │
│  │  └─ 每个Master节点运行:                                           │
│  │     • kube-apiserver                                              │
│  │     • kube-controller-manager (Leader Election)                   │
│  │     • kube-scheduler (Leader Election)                            │
│  │     • etcd (集群模式)                                             │
│  │                                                                    │
│  └─ etcd集群                                                          │
│     ├─ 部署方式                                                       │
│     │  • 堆叠式: 每个Master节点运行etcd (kubeadm默认)                │
│     │  • 外部式: 独立的etcd集群                                      │
│     │                                                                 │
│     └─ 集群配置                                                       │
│        • 初始集群: etcd1=https://10.0.0.101:2380,                    │
│                     etcd2=https://10.0.0.102:2380,                    │
│                     etcd3=https://10.0.0.103:2380                     │
│        • Raft一致性协议                                               │
│        • 数据复制到多数节点                                           │
│                                                                         │
│  高可用初始化流程                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Step 1: 准备负载均衡器                                          │   │
│  │  ├─ 部署HAProxy和Keepalived                                     │   │
│  │  ├─ 配置VIP: 10.0.0.100                                         │   │
│  │  ├─ 配置后端服务器列表                                          │   │
│  │  └─ 验证负载均衡器工作正常                                      │   │
│  │                                                                  │   │
│  │  Step 2: 初始化第一个Master节点                                  │   │
│  │  ├─ 执行命令:                                                    │   │
│  │  │  kubeadm init \                                              │   │
│  │  │    --control-plane-endpoint "10.0.0.100:6443" \              │   │
│  │  │    --upload-certs \                                          │   │
│  │  │    --pod-network-cidr=10.244.0.0/16                          │   │
│  │  │                                                               │   │
│  │  ├─ 关键参数说明:                                                │   │
│  │  │  --control-plane-endpoint:                                   │   │
│  │  │    • 指定负载均衡器地址                                      │   │
│  │  │    • 所有组件通过此地址访问API Server                        │   │
│  │  │    • 证书SAN包含此地址                                       │   │
│  │  │                                                               │   │
│  │  │  --upload-certs:                                              │   │
│  │  │    • 将证书上传到kubeadm-certs Secret                        │   │
│  │  │    • 其他Master节点可下载证书                                │   │
│  │  │    • 有效期: 2小时                                           │   │
│  │  │                                                               │   │
│  │  └─ 输出加入命令:                                                │   │
│  │     Master加入命令:                                              │   │
│  │     kubeadm join 10.0.0.100:6443 \                              │   │
│  │       --token <token> \                                         │   │
│  │       --discovery-token-ca-cert-hash sha256:<hash> \            │   │
│  │       --control-plane \                                         │   │
│  │       --certificate-key <cert-key>                              │   │
│  │                                                                  │   │
│  │     Worker加入命令:                                              │   │
│  │     kubeadm join 10.0.0.100:6443 \                              │   │
│  │       --token <token> \                                         │   │
│  │       --discovery-token-ca-cert-hash sha256:<hash>              │   │
│  │                                                                  │   │
│  │  Step 3: 加入其他Master节点                                      │   │
│  │  ├─ 在master2上执行Master加入命令                                │   │
│  │  │  • 下载证书                                                   │   │
│  │  │  • 生成控制平面静态Pod                                        │   │
│  │  │  • 加入etcd集群                                               │   │
│  │
```
Kubeadm 详细设计（续）

## 九、证书管理详细设计
### 9.1 证书生成策略
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    证书生成策略详解                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    证书层次结构                                    │ │
│  │                                                                   │ │
│  │   Level 0: 根CA (Root CA)                                        │ │
│  │   ┌─────────────────────────────────────────────────────────┐    │ │
│  │   │  集群根CA (ca.crt/ca.key)                               │    │ │
│  │   │  • 用于签发所有组件证书                                  │    │ │
│  │   │  • 有效期: 10年                                          │    │ │
│  │   │  • 用途: 签发证书、CRL签名                               │    │ │
│  │   └─────────────────────────────────────────────────────────┘    │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │   Level 1: 中间CA (Intermediate CA)                              │ │
│  │   ┌─────────────────────────────────────────────────────────┐    │ │
│  │   │  etcd CA (etcd/ca.crt)                                  │    │ │
│  │   │  • 独立的etcd信任域                                      │    │ │
│  │   │  • 用于签发etcd相关证书                                  │    │ │
│  │   │                                                         │    │ │
│  │   │  Front Proxy CA (front-proxy-ca.crt)                    │    │ │
│  │   │  • 用于扩展API Server代理                               │    │ │
│  │   │  • 支持聚合API层                                        │    │ │
│  │   └─────────────────────────────────────────────────────────┘    │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │   Level 2: 终端证书                          │ │
│  │   ┌─────────────────────────────────────────────────────────┐    │ │
│  │   │  服务端证书:                                            │    │ │
│  │   │  • apiserver.crt (API Server服务端)                     │    │ │
│  │   │  • etcd/server.crt (etcd服务端)                         │    │ │
│  │   │  • etcd/peer.crt (etcd对等通信)                         │    │ │
│  │   │                                                         │    │ │
│  │   │  客户端证书:                                            │    │ │
│  │   │  • apiserver-kubelet-client.crt                        │    │ │
│  │   │  • apiserver-etcd-client.crt                           │    │ │
│  │   │  • etcd/healthcheck-client.crt                         │    │ │
│  │   │  • front-proxy-client.crt                              │    │ │
│  │   └─────────────────────────────────────────────────────────┘    │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
### 9.2 证书SAN（Subject Alternative Name）设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    API Server 证书 SAN 设计                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  设计目标:                                                              │
│  • 确保客户端可以通过多种方式访问 API Server                           │
│  • 支持 kubectl、kubelet、controller-manager 等组件访问                │
│  • 支持 Pod 内部访问                                                   │
│  • 支持外部访问                                                        │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  DNS SANs (域名):                                                 │ │
│  │  ├─ kubernetes                    # 短名称                       │ │
│  │  ├─ kubernetes.default            # 默认命名空间                 │ │
│  │  ├─ kubernetes.default.svc        # 服务全名                     │ │
│  │  ├─ kubernetes.default.svc.cluster.local  # FQDN                │ │
│  │  ├─ localhost                     # 本地访问                     │ │
│  │  ├─ <hostname>                    # 节点主机名                   │ │
│  │  └─ <load-balancer-domain>        # 负载均衡器域名 (HA场景)      │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  IP SANs (IP地址):                                                │ │
│  │  ├─ 10.96.0.1                     # Kubernetes Service IP       │ │
│  │  ├─ 127.0.0.1                     # 本地回环                     │ │
│  │  ├─ <node-ip>                     # 节点IP                       │ │
│  │  ├─ <load-balancer-vip>           # 负载均衡器VIP (HA场景)       │ │
│  │  └─ <additional-ips>              # 用户自定义IP                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  设计说明:                                                              │
│  1. Service IP (10.96.0.1) 是 kube-apiserver Service 的 ClusterIP     │
│     • 确保 Pod 内通过 Service 访问 API Server 时证书验证通过           │
│     • 这个IP在 kube-apiserver 启动时通过 --service-cluster-ip-range   │
│       参数指定的范围内自动分配                                         │
│                                                                         │
│  2. 负载均衡器域名和VIP 在高可用场景中必须包含                        │
│     • 确保通过负载均衡器访问时证书验证通过                             │
│     • 所有Master节点的API Server证书都应包含这些SAN                   │
│                                                                         │
│  3. 用户可以通过配置文件添加额外的SAN                                  │
│     • certSANs 字段允许自定义域名和IP                                  │
│     • 适用于特殊网络环境或自定义域名场景                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 9.3 证书轮换机制
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    证书轮换机制                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Kubelet 证书自动轮换                            │ │
│  │                                                                   │ │
│  │  工作原理:                                                        │ │
│  │  • Kubelet 在证书即将过期时自动申请新证书                         │ │
│  │  • 默认有效期: 1年                                               │ │
│  │  • 自动轮换阈值: 过期前30天                                      │ │
│  │                                                                   │ │
│  │  轮换流程:                                                        │ │
│  │                                                                   │ │
│  │  ┌──────────┐    1. 检测证书即将过期    ┌──────────────────┐     │ │
│  │  │ Kubelet  │ ───────────────────────▶ │ 证书有效期检查    │     │ │
│  │  │          │                          │ (剩余<30天)      │     │ │
│  │  └──────────┘                          └────────┬─────────┘     │ │
│  │       │                                         │               │ │
│  │       │ 2. 生成新密钥对                         │               │ │
│  │       ▼                                         ▼               │ │
│  │  ┌──────────┐    3. 创建CSR请求           ┌──────────────────┐  │ │
│  │  │ 生成CSR  │ ─────────────────────────▶ │ CSR对象          │  │ │
│  │  │          │                            │ (Pending状态)    │  │ │
│  │  └──────────┘                            └────────┬─────────┘  │ │
│  │                                                   │              │ │
│  │                                                   ▼              │ │
│  │  ┌──────────────────────────────────────────────────────────┐   │ │
│  │  │                    CSR自动批准                            │   │ │
│  │  │  • kubeadm 启动的集群默认启用自动批准                     │   │ │
│  │  │  • 由 csr-approver 控制器处理                             │   │ │
│  │  │  • 验证请求来自合法的 kubelet                             │   │ │
│  │  └──────────────────────────────────────────────────────────┘   │ │
│  │                                                   │              │ │
│  │                                                   ▼              │ │
│  │  ┌──────────┐    4. 获取签名证书         ┌──────────────────┐   │ │
│  │  │ Kubelet  │ ◀─────────────────────── │ CSR.status.      │   │ │
│  │  │          │                          │ certificate      │   │ │
│  │  └──────────┘                          └──────────────────┘   │ │
│  │       │                                                          │ │
│  │       │ 5. 更新 kubelet.conf                                     │ │
│  │       ▼                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────┐   │ │
│  │  │  新证书写入磁盘，kubelet 重新加载配置                     │   │ │
│  │  │  旧证书保留作为备份                                        │   │ │
│  │  └──────────────────────────────────────────────────────────┘   │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    控制平面证书手动轮换                            │ │
│  │                                                                   │ │
│  │  触发条件:                                                        │ │
│  │  • 证书即将过期                                                   │ │
│  │  • 安全事件需要更换证书                                           │ │
│  │  • 合规性要求                                                     │ │
│  │                                                                   │ │
│  │  轮换命令:                                                        │ │
│  │  kubeadm certs renew all                                         │ │
│  │                                                                   │ │
│  │  轮换后操作:                                                      │ │
│  │  1. 重启控制平面组件 (静态Pod自动重启)                            │ │
│  │  2. 更新 kubeconfig 文件                                         │ │
│  │  3. 分发新证书到其他Master节点 (HA场景)                           │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十、静态Pod管理设计
### 10.1 静态Pod工作原理
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    静态Pod 工作原理                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  设计目标:                                                              │
│  • 让控制平面组件作为Pod运行，便于管理和监控                           │
│  • 不依赖 etcd 存储配置，直接从本地文件读取                            │
│  • 即使 API Server 不可用，控制平面组件也能启动                        │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    静态Pod生命周期                                 │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  1. 清单文件创建                                             │ │ │
│  │  │     路径: /etc/kubernetes/manifests/                        │ │ │
│  │  │     文件: kube-apiserver.yaml                               │ │ │
│  │  │            kube-controller-manager.yaml                     │ │ │
│  │  │            kube-scheduler.yaml                              │ │ │
│  │  │            etcd.yaml                                        │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  2. Kubelet 监控                                             │ │ │
│  │  │     • 定期扫描 manifests 目录 (默认4秒)                     │ │ │
│  │  │     • 检测文件变化 (创建/修改/删除)                          │ │ │
│  │  │     • 解析 YAML 文件为 Pod 定义                              │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  3. Pod 创建                                                 │ │ │
│  │  │     • Kubelet 创建镜像 Pod (Mirror Pod)                     │ │ │
│  │  │     • 在 API Server 中创建对应的 Pod 对象                   │ │ │
│  │  │     • Pod 名称: <node-name>-<component-name>                │ │ │
│  │  │     • 命名空间: kube-system                                 │ │ │
│  │  │     • 添加标签: tier=control-plane                          │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  4. 容器运行                                                 │ │ │
│  │  │     • 通过容器运行时 创建容器              │ │ │
│  │  │     • 使用 host network (共享主机网络)                      │ │ │
│  │  │     • 挂载主机目录 (证书、配置等)                            │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  5. 状态同步                                                 │ │ │
│  │  │     • Kubelet 监控容器状态                                   │ │ │
│  │  │     • 更新 API Server 中的 Pod 状态                         │ │ │
│  │  │     • 容器退出时自动重启                                     │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    静态Pod特性                                     │ │
│  │                                                                   │ │
│  │  1. 自动重启                                                      │ │
│  │     • 容器退出后 Kubelet 自动重启                                 │ │
│  │     • 重启策略固定为 Always                                       │ │
│  │                                                                   │ │
│  │  2. 无法通过 API 删除                                             │ │
│  │     • kubectl delete pod 后会自动重建                            │ │
│  │     • 必须删除清单文件才能彻底删除                                │ │
│  │                                                                   │ │
│  │  3. 资源限制                                                      │ │
│  │     • 可以在清单中设置资源请求和限制                              │ │
│  │     • 建议为控制平面组件设置合理的资源限制                        │ │
│  │                                                                   │ │
│  │  4. 健康检查                                                      │ │
│  │     • 支持配置 livenessProbe 和 readinessProbe                   │ │
│  │     • 健康检查失败会触发重启                                      │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 10.2 静态Pod清单设计要点
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    静态Pod清单设计要点                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kube-apiserver.yaml 设计要点                   │ │
│  │                                                                   │ │
│  │  1. 网络配置                                                      │ │
│  │     • hostNetwork: true (使用主机网络)                            │ │
│  │     • 直接监听主机端口，无需端口映射                              │ │
│  │                                                                   │ │
│  │  2. 命令行参数                                                    │ │
│  │     • --advertise-address: 对外广播地址                          │ │
│  │     • --secure-port: HTTPS端口 (默认6443)                        │ │
│  │     • --etcd-servers: etcd地址列表                               │ │
│  │     • --service-cluster-ip-range: Service CIDR                  │ │
│  │     • --authorization-mode: 授权模式                             │ │
│  │     • --enable-admission-plugins: 准入控制器                     │ │
│  │                                                                   │ │
│  │  3. 证书挂载                                                      │ │
│  │     • /etc/kubernetes/pki: 证书目录 (只读)                       │ │
│  │     • /etc/ssl/certs: 系统CA证书 (只读)                          │ │
│  │                                                                   │ │
│  │  4. 健康检查                                                      │ │
│  │     • livenessProbe: /healthz 端点                               │ │
│  │     • 失败阈值: 3次                                              │ │
│  │     • 检查间隔: 10秒                                             │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kube-controller-manager.yaml 设计要点          │ │
│  │                                                                   │ │
│  │  1. 命令行参数                                                    │ │
│  │     • --cluster-signing-cert-file: 签发证书的CA                  │ │
│  │     • --cluster-signing-key-file: 签发证书的CA私钥               │ │
│  │     • --use-service-account-credentials: 启用SA凭证              │ │
│  │     • --leader-elect: 启用领导者选举 (HA场景)                    │ │
│  │     • --controllers: 启用的控制器列表                            │ │
│  │                                                                   │ │
│  │  2. 领导者选举                                                    │ │
│  │     • 高可用场景下只有一个实例处于活动状态                        │ │
│  │     • 通过 etcd 或 ConfigMap 实现选举                            │ │
│  │     • 租约时长: 15秒                                             │ │
│  │     • 续租间隔: 10秒                                             │ │
│  │                                                                   │ │
│  │  3. kubeconfig 挂载                                               │ │
│  │     • /etc/kubernetes/controller-manager.conf                    │ │
│  │     • 用于访问 API Server                                        │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kube-scheduler.yaml 设计要点                   │ │
│  │                                                                   │ │
│  │  1. 命令行参数                                                    │ │
│  │     • --bind-address: 监听地址                                   │ │
│  │     • --leader-elect: 启用领导者选举                             │ │
│  │     • --scheduler-name: 调度器名称                               │ │
│  │     • --policy-config-file: 调度策略配置                         │ │
│  │                                                                   │ │
│  │  2. 领导者选举                                                    │ │
│  │     • 与 controller-manager 类似                                 │ │
│  │     • 确保同一时刻只有一个调度器在工作                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    etcd.yaml 设计要点                             │ │
│  │                                                                   │ │
│  │  1. 单节点模式                                                    │ │
│  │     • --name: 节点名称                                           │ │
│  │     • --listen-client-urls: 客户端监听地址                       │ │
│  │     • --advertise-client-urls: 客户端广播地址                    │ │
│  │                                                                   │ │
│  │  2. 集群模式                                                      │ │
│  │     • --initial-cluster: 初始集群成员                            │ │
│  │     • --initial-cluster-state: new/existing                      │ │
│  │     • --listen-peer-urls: 对等通信监听地址                       │ │
│  │     • --initial-advertise-peer-urls: 对等通信广播地址            │ │
│  │                                                                   │ │
│  │  3. 数据持久化                                                    │ │
│  │     • --data-dir: 数据目录                                       │ │
│  │     • 挂载主机目录: /var/lib/etcd                                │ │
│  │                                                                   │ │
│  │  4. 证书配置                                                      │ │
│  │     • --client-cert-auth: 启用客户端证书认证                     │ │
│  │     • --trusted-ca-file: CA证书                                  │ │
│  │     • --cert-file: 服务端证书                                    │ │
│  │     • --key-file: 服务端私钥                                     │ │
│  │     • --peer-client-cert-auth: 对等通信证书认证                  │ │
│  │     • --peer-trusted-ca-file: 对等通信CA证书                     │ │
│  │     • --peer-cert-file: 对等通信证书                             │ │
│  │     • --peer-key-file: 对等通信私钥                              │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十一、Token与发现机制详细设计
### 11.1 Bootstrap Token 详细设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    Bootstrap Token 详细设计                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Token 格式                                      │ │
│  │                                                                   │ │
│  │  格式: <token-id>.<token-secret>                                 │ │
│  │                                                                   │ │
│  │  示例: abcdef.0123456789abcdef                                   │ │
│  │                                                                   │ │
│  │  组成部分:                                                        │ │
│  │  • Token ID: 6个字符 [a-z0-9]                                    │ │
│  │    - 用于标识Token，公开可见                                      │ │
│  │    - 在Secret名称中使用: bootstrap-token-<token-id>              │ │
│  │                                                                   │ │
│  │  • Token Secret: 16个字符 [a-z0-9]                               │ │
│  │    - Token的密钥部分，需要保密                                    │ │
│  │    - 用于验证Token持有者的身份                                    │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Token Secret 结构                              │ │
│  │                                                                   │ │
│  │  Secret 名称: bootstrap-token-<token-id>                         │ │
│  │  命名空间: kube-system                                           │ │
│  │                                                                   │ │
│  │  数据字段:                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  token-id: "abcdef"                                         │ │ │
│  │  │  token-secret: "0123456789abcdef"                           │ │ │
│  │  │  usage-bootstrap-authentication: "true"                     │ │ │
│  │  │  usage-bootstrap-signing: "true"                            │ │ │
│  │  │  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token" │
│  │  │  expiration: "2024-01-01T00:00:00Z"                         │ │ │
│  │  │  description: "kubeadm bootstrap token"                     │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  字段说明:                                                        │ │
│  │  • usage-bootstrap-authentication: 是否用于认证                  │ │
│  │  • usage-bootstrap-signing: 是否用于签名CSR                      │ │
│  │  • auth-extra-groups: 认证后附加的组                             │ │
│  │  • expiration: 过期时间 (默认24小时)                             │ │
│  │  • description: Token描述                                        │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Token 认证流程                                  │ │
│  │                                                                   │ │
│  │  1. 客户端请求                                                    │ │
│  │     • 使用Token作为Bearer Token                                  │ │
│  │     • Authorization: Bearer <token-id>.<token-secret>            │ │
│  │                                                                   │ │
│  │  2. API Server 验证                                               │ │
│  │     • 从 kube-system 命名空间获取 bootstrap-token-<token-id>      │ │
│  │     • 验证 token-secret 是否匹配                                 │ │
│  │     • 检查是否过期                                                │ │
│  │     • 检查 usage-bootstrap-authentication 是否为 true            │ │
│  │                                                                   │ │
│  │  3. 生成用户信息                                                  │ │
│  │     • 用户名: system:bootstrap:<token-id>                        │ │
│  │     • 组: system:bootstrappers                                   │ │
│  │     • 附加组: auth-extra-groups 中指定的组                       │ │
│  │                                                                   │ │
│  │  4. RBAC 授权                                                     │ │
│  │     • 检查用户是否有权限执行请求的操作                            │ │
│  │     • kubeadm 默认创建必要的 RBAC 规则                           │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Token 相关 RBAC 规则                           │ │
│  │                                                                   │ │
│  │  1. 允许创建CSR                                                   │ │
│  │     • ClusterRole: system:node-bootstrapper                      │ │
│  │     • 资源: certificatesigningrequests                           │ │
│  │     • 动词: create, get                                          │ │
│  │                                                                   │ │
│  │  2. 自动批准CSR                                                   │ │
│  │     • ClusterRole: system:certificates.k8s.io:certificatesigningrequests:nodeclient │
│  │     • 绑定到组: system:bootstrappers                             │ │
│  │                                                                   │ │
│  │  3. 领导者选举权限                                                │ │
│  │     • 允许 kubelet 在 kube-system 命名空间创建 Lease             │ │
│  │     • 用于 kubelet 的领导者选举                                  │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 11.2 发现机制详细设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    发现机制 详细设计                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    发现方式对比                                    │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  方式1: Token Discovery (推荐)                               │ │ │
│  │  │                                                             │ │ │
│  │  │  优点:                                                       │ │ │
│  │  │  • 安全性高，验证CA证书哈希                                  │ │ │
│  │  │  • 配置简单，一条命令即可                                    │ │ │
│  │  │  • 支持自动过期                                              │ │ │
│  │  │                                                             │ │ │
│  │  │  缺点:                                                       │ │ │
│  │  │  • Token有过期时间限制                                      │ │ │
│  │  │  • 需要提前获取CA证书哈希                                    │ │ │
│  │  │                                                             │ │ │
│  │  │  使用场景:                                                   │ │ │
│  │  │  • 生产环境                                                  │ │ │
│  │  │  • 需要安全验证的场景                                        │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  方式2: File Discovery                                      │ │ │
│  │  │                                                             │ │ │
│  │  │  优点:                                                       │ │ │
│  │  │  • 不依赖Token                                              │ │ │
│  │  │  • 可以预先分发配置文件                                      │ │ │
│  │  │  • 适合离线环境                                              │ │ │
│  │  │                                                             │ │ │
│  │  │  缺点:                                                       │ │ │
│  │  │  • 需要手动管理配置文件                                      │ │ │
│  │  │  • 配置文件包含敏感信息                                      │ │ │
│  │  │                                                             │ │ │
│  │  │  使用场景:                                                   │ │ │
│  │  │  • 离线环境                                                  │ │ │
│  │  │  • 批量部署                                                  │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  方式3: Unsafe Skip CA Verification (不推荐)                 │ │ │
│  │  │                                                             │ │ │
│  │  │  优点:                                                       │ │ │
│  │  │  • 配置最简单                                                │ │ │
│  │  │  • 不需要CA证书哈希                                          │ │ │
│  │  │                                                             │ │ │
│  │  │  缺点:                                                       │ │ │
│  │  │  • 安全性低，容易受到中间人攻击                              │ │ │
│  │  │  • 仅适用于测试环境                                          │ │ │
│  │  │                                                             │ │ │
│  │  │  使用场景:                                                   │ │ │
│  │  │  • 测试环境                                                  │ │ │
│  │  │  • 完全可信的网络环境                                        │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Token Discovery 详细流程                       │ │
│  │                                                                   │ │
│  │  1. 获取集群信息                                                  │ │
│  │     • 使用Token连接API Server                                    │ │
│  │     • 请求: GET /api/v1/namespaces/kube-public/configmaps/cluster-info │
│  │     • 返回: 包含 kubeconfig 的 ConfigMap                         │ │
│  │                                                                   │ │
│  │  2. 验证CA证书                                                    │ │
│  │     • 从 ConfigMap 提取 CA 证书                                  │ │
│  │     • 计算证书哈希: SHA256(base64(cert))                         │ │
│  │     • 与用户提供的哈希对比                                       │ │
│  │     • 匹配则继续，否则拒绝连接                                   │ │
│  │                                                                   │ │
│  │  3. 获取集群配置                                                  │ │
│  │     • 从 ConfigMap 获取集群配置信息                              │ │
│  │     • 包括 API Server 地址、集群名称等                           │ │
│  │                                                                   │ │
│  │  4. 开始TLS Bootstrap                                            │ │
│  │     • 使用Token进行认证                                          │ │
│  │     • 创建CSR请求                                                │ │
│  │     • 等待证书签发                                               │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    CA证书哈希计算                                  │ │
│  │                                                                   │ │
│  │  计算方法:                                                        │ │
│  │  1. 获取CA证书 (PEM格式)                                         │ │
│  │  2. 计算SHA256哈希                                               │ │
│  │  3. 转换为十六进制字符串                                          │ │
│  │                                                                   │ │
│  │  获取方式:                                                        │ │
│  │  • kubeadm init 输出中包含                                       │ │
│  │  • kubeadm token create --print-join-command                     │ │
│  │  • openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \       │ │
│  │    openssl rsa -pubin -outform der | \                           │ │
│  │    openssl dgst -sha256 -hex                                     │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十二、配置管理详细设计
### 12.1 ConfigMap 存储设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    ConfigMap 存储设计                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kubeadm-config ConfigMap                       │ │
│  │                                                                   │ │
│  │  名称: kubeadm-config                                            │ │
│  │  命名空间: kube-system                                           │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 存储集群配置信息                                              │ │
│  │  • 用于升级时读取配置                                            │ │
│  │  • 用于其他Master节点加入时获取配置                              │ │
│  │                                                                   │ │
│  │  数据内容:                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  ClusterConfiguration: |                                    │ │ │
│  │  │    apiServer:                                               │ │ │
│  │  │      extraArgs:                                             │ │ │
│  │  │        authorization-mode: Node,RBAC                        │ │ │
│  │  │      timeoutForControlPlane: 4m0s                           │ │ │
│  │  │    certificatesDir: /etc/kubernetes/pki                     │ │ │
│  │  │    controlPlaneEndpoint: lb.example.com:6443                │ │ │
│  │  │    etcd:                                                    │ │ │
│  │  │      local:                                                 │ │ │
│  │  │        dataDir: /var/lib/etcd                               │ │ │
│  │  │    imageRepository: registry.k8s.io                         │ │ │
│  │  │    kubernetesVersion: v1.28.0                               │ │ │
│  │  │    networking:                                              │ │ │
│  │  │      dnsDomain: cluster.local                               │ │ │
│  │  │      podSubnet: 10.244.0.0/16                               │ │ │
│  │  │      serviceSubnet: 10.96.0.0/12                            │ │ │
│  │  │                                                             │ │ │
│  │  │  InitConfiguration: |                                       │ │ │
│  │  │    localAPIEndpoint:                                        │ │ │
│  │  │      advertiseAddress: 192.168.1.100                        │ │ │
│  │  │      bindPort: 6443                                         │ │ │
│  │  │    nodeRegistration:                                        │ │ │
│  │  │      criSocket: unix:///var/run/containerd/containerd.sock  │ │ │
│  │  │      name: master-1                                         │ │ │
│  │  │      taints:                                                │ │ │
│  │  │      - effect: NoSchedule                                   │ │ │
│  │  │        key: node-role.kubernetes.io/control-plane           │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kube-root-ca.crt ConfigMap                     │ │
│  │                                                                   │ │
│  │  名称: kube-root-ca.crt                                          │ │
│  │  命名空间: 每个命名空间都有                                       │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 存储集群根CA证书                                              │ │
│  │  • 供 Pod 内部验证 API Server 证书使用                           │ │
│  │  • ServiceAccount 自动挂载                                       │ │
│  │                                                                   │ │
│  │  数据内容:                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  ca.crt: |                                                  │ │ │
│  │  │    -----BEGIN CERTIFICATE-----                              │ │ │
│  │  │    MIIC... (CA证书内容)                                      │ │ │
│  │  │    -----END CERTIFICATE-----                                │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    kubeadm-certs Secret (HA场景)                  │ │
│  │                                                                   │ │
│  │  名称: kubeadm-certs                                             │ │
│  │  命名空间: kube-system                                           │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 存储控制平面证书                                              │ │
│  │  • 用于其他Master节点加入时下载证书                              │ │
│  │  • 使用 certificate-key 加密                                    │ │
│  │                                                                   │ │
│  │  数据内容:                                                        │ │
│  │  • 所有控制平面证书和私钥                                        │ │
│  │  • 使用对称密钥加密存储                                          │ │
│  │                                                                   │ │
│  │  安全说明:                                                        │ │
│  │  • certificate-key 只在 init 时显示一次                         │ │
│  │  • Secret 有效期: 2小时                                         │ │
│  │  • 建议在所有Master加入后删除此Secret                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 12.2 配置继承与覆盖机制
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    配置继承与覆盖机制                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  设计目标:                                                              │
│  • 支持全局配置和节点级配置                                            │
│  • 允许节点覆盖全局配置                                                │
│  • 保持配置的一致性和可追溯性                                          │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    配置优先级 (从高到低)                          │ │
│  │                                                                   │ │
│  │  1. 命令行参数                                                    │ │
│  │     • 最高优先级                                                  │ │
│  │     • 直接覆盖配置文件中的对应项                                  │ │
│  │     • 示例: --kubernetes-version=v1.28.0                         │ │
│  │                                                                   │ │
│  │  2. 配置文件                                                      │ │
│  │     • 用户提供的 YAML 配置文件                                   │ │
│  │     • 可以覆盖默认值                                              │ │
│  │                                                                   │ │
│  │  3. 内部默认值                                                    │ │
│  │     • kubeadm 内置的默认配置                                     │ │
│  │     • 确保在没有配置时也能正常工作                                │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    节点级配置覆盖                                  │ │
│  │                                                                   │ │
│  │  场景: 集群中不同节点使用不同的配置                               │ │
│  │                                                                   │ │
│  │  示例:                                                            │ │
│  │  • 全局配置: 所有节点使用相同的镜像仓库                           │ │
│  │  • 节点配置: 某些节点使用不同的 kubelet 参数                      │ │
│  │                                                                   │ │
│  │  实现方式:                                                        │ │
│  │  1. JoinConfiguration 中指定节点特定配置                         │ │
│  │  2. nodeRegistration.kubeletExtraArgs 设置节点参数               │ │
│  │  3. 节点标签和污点在 nodeRegistration 中设置                     │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    配置验证机制                                    │ │
│  │                                                                   │ │
│  │  验证时机:                                                        │ │
│  │  • 配置加载后立即验证                                            │ │
│  │  • 在 Preflight 阶段执行                                         │ │
│  │                                                                   │ │
│  │  验证内容:                                                        │ │
│  │  1. 必填字段检查                                                  │ │
│  │     • kubernetesVersion                                         │ │
│  │     • networking.podSubnet                                      │ │
│  │                                                                   │ │
│  │  2. 格式验证                                                      │ │
│  │     • IP地址格式                                                 │ │
│  │     • CIDR格式                                                   │ │
│  │     • 端口号范围                                                 │ │
│  │                                                                   │ │
│  │  3. 逻辑验证                                                      │ │
│  │     • Pod CIDR 和 Service CIDR 不重叠                           │ │
│  │     • API Server 地址可达                                        │ │
│  │     • 版本兼容性检查                                              │ │
│  │                                                                   │ │
│  │  4. 安全验证                                                      │ │
│  │     • 证书文件存在性                                              │ │
│  │     • 私钥权限检查                                                │ │
│  │     • 敏感信息不暴露                                              │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十三、网络配置详细设计
### 13.1 网络CIDR设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    网络CIDR设计                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Service CIDR (服务网络)                        │ │
│  │                                                                   │ │
│  │  默认值: 10.96.0.0/12                                            │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 为 Service 分配 ClusterIP                                     │ │
│  │  • 虚拟网络，不占用实际IP地址                                    │ │
│  │  • 由 kube-proxy 在节点上配置 iptables/ipvs 规则                 │ │
│  │                                                                   │ │
│  │  分配规则:                                                        │ │
│  │  • 10.96.0.1: Kubernetes API Server Service                     │ │
│  │  • 10.96.0.10: kube-dns Service                                 │ │
│  │  • 其他: 用户创建的 Service                                      │ │
│  │                                                                   │ │
│  │  设计考虑:                                                        │ │
│  │  • 必须与主机网络不重叠                                          │ │
│  │  • 必须与 Pod CIDR 不重叠                                        │ │
│  │  • 大小要足够支持预期的 Service 数量                              │ │
│  │  • /12 提供 2^20 = 约100万个IP地址                               │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Pod CIDR (Pod网络)                             │ │
│  │                                                                   │ │
│  │  默认值: 无 (必须由用户指定或CNI插件分配)                         │ │
│  │  常见值: 10.244.0.0/16, 192.168.0.0/16                           │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 为 Pod 分配IP地址                                             │ │
│  │  • 由 CNI 插件管理                                               │ │
│  │  • 每个节点分配一个子网                                          │ │
│  │                                                                   │ │
│  │  分配方式:                                                        │ │
│  │  • 方式1: kubeadm 配置 --pod-network-cidr                        │ │
│  │  • 方式2: CNI 插件配置文件中指定                                 │ │
│  │  • 方式3: 使用 kube-controller-manager --allocate-node-cidrs     │ │
│  │                                                                   │ │
│  │  设计考虑:                                                        │ │
│  │  • 必须与主机网络不重叠                                          │ │
│  │  • 必须与 Service CIDR 不重叠                                    │ │
│  │  • 要考虑节点数量和每节点Pod数量                                  │ │
│  │  • 示例: 10.244.0.0/16                                           │ │
│  │    - 可分配 256 个 /24 子网                                      │ │
│  │    - 每个子网支持 254 个Pod                                      │ │
│  │    - 总共支持 256 * 254 = 65024 个Pod                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    网络规划示例                                    │ │
│  │                                                                   │ │
│  │  场景: 企业内网部署                                               │ │
│  │                                                                   │ │
│  │  主机网络: 192.168.1.0/24                                         │ │
│  │  ├─ Master节点: 192.168.1.10-192.168.1.12                        │ │
│  │  └─ Worker节点: 192.168.1.20-192.168.1.50                        │ │
│  │                                                                   │ │
│  │  Service CIDR: 10.96.0.0/12                                       │ │
│  │  ├─ 范围: 10.96.0.0 - 10.111.255.255                             │ │
│  │  └─ 与主机网络不重叠 ✓                                           │ │
│  │                                                                   │ │
│  │  Pod CIDR: 10.244.0.0/16                                          │ │
│  │  ├─ 范围: 10.244.0.0 - 10.244.255.255                            │ │
│  │  ├─ 与主机网络不重叠 ✓                                           │ │
│  │  ├─ 与 Service CIDR 不重叠 ✓                                     │ │
│  │  └─ 节点子网分配:                                                │ │
│  │      • node-1: 10.244.1.0/24                                     │ │
│  │      • node-2: 10.244.2.0/24                                     │ │
│  │      • node-3: 10.244.3.0/24                                     │ │
│  │      • ...                                                        │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 13.2 DNS配置设计
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    DNS配置设计                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    集群DNS域名                                     │ │
│  │                                                                   │ │
│  │  默认值: cluster.local                                            │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 集群内部服务的域名后缀                                         │ │
│  │  • Service 的 FQDN 格式: <service>.<namespace>.svc.<domain>       │ │
│  │  • Pod 的 FQDN 格式: <pod-ip>.<namespace>.pod.<domain>            │ │
│  │                                                                   │ │
│  │  示例:                                                            │ │
│  │  • Service: kubernetes.default.svc.cluster.local                 │ │
│  │  • Pod: 10-244-1-5.default.pod.cluster.local                     │ │
│  │                                                                   │ │
│  │  设计考虑:                                                        │ │
│  │  • 建议使用默认值 cluster.local                                   │ │
│  │  • 如需修改，确保不与外部域名冲突                                 │ │
│  │  • 修改后需要重新配置 CoreDNS                                     │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    CoreDNS 配置                                    │ │
│  │                                                                   │ │
│  │  部署方式:                                                        │ │
│  │  • kubeadm 自动部署 CoreDNS                                      │ │
│  │  • 作为 Deployment 运行在 kube-system 命名空间                   │ │
│  │  • 通过 Service kube-dns 暴露                                    │ │
│  │                                                                   │ │
│  │  Service 配置:                                                    │ │
│  │  • 名称: kube-dns                                                │ │
│  │  • 命名空间: kube-system                                         │ │
│  │  • ClusterIP: 10.96.0.10 (Service CIDR 内的第一个可用IP)         │ │
│  │  • 端口: 53 (UDP/TCP)                                            │ │
│  │                                                                   │ │
│  │  Corefile 配置:                                                   │ │
│  │  • 定义DNS解析规则                                                │ │
│  │  • 配置上游DNS服务器                                              │ │
│  │  • 配置自定义域名解析                                             │ │
│  │                                                                   │ │
│  │  Kubelet 配置:                                                    │ │
│  │  • --cluster-dns: CoreDNS Service IP                             │ │
│  │  • --cluster-domain: 集群域名                                    │ │
│  │  • 自动注入到 Pod 的 /etc/resolv.conf                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    NodeLocal DNS Cache (可选)                     │ │
│  │                                                                   │ │
│  │  作用:                                                            │ │
│  │  • 在每个节点上运行 DNS 缓存                                     │ │
│  │  • 减少 CoreDNS 压力                                             │ │
│  │  • 降低 DNS 查询延迟                                             │ │
│  │                                                                   │ │
│  │  工作原理:                                                        │ │
│  │  • DaemonSet 在每个节点运行 dnscache                             │ │
│  │  • 监听 169.254.20.10:53                                         │ │
│  │  • Kubelet 配置使用 NodeLocal DNS                                │ │
│  │  • 缓存命中直接返回，未命中转发到 CoreDNS                        │ │
│  │                                                                   │ │
│  │  安装方式:                                                        │ │
│  │  • kubeadm 不自动安装                                            │ │
│  │  • 需要手动部署清单文件                                          │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十四、安全设计
### 14.1 RBAC默认配置
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    RBAC默认配置                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    默认角色和绑定                                  │ │
│  │                                                                   │ │
│  │  1. 管理员权限                                                    │ │
│  │     • ClusterRole: cluster-admin                                 │ │
│  │     • ClusterRoleBinding: cluster-admin                          │ │
│  │     • 绑定到: system:masters 组                                  │ │
│  │     • 权限: 所有资源的所有操作                                   │ │
│  │                                                                   │ │
│  │  2. 控制平面组件权限                                              │ │
│  │     • ClusterRole: system:kube-controller-manager                │ │
│  │     • ClusterRoleBinding: system:kube-controller-manager         │ │
│  │     • 绑定到: system:kube-controller-manager 用户                │ │
│  │     • 权限: 控制器所需的资源访问权限                             │ │
│  │                                                                   │ │
│  │     • ClusterRole: system:kube-scheduler                         │ │
│  │     • ClusterRoleBinding: system:kube-scheduler                  │ │
│  │     • 绑定到: system:kube-scheduler 用户                         │ │
│  │     • 权限: 调度器所需的资源访问权限                             │ │
│  │                                                                   │ │
│  │  3. 节点权限                                                      │ │
│  │     • ClusterRole: system:node                                   │ │
│  │     • 权限: kubelet 所需的资源访问权限                           │ │
│  │     • 每个节点自动创建对应的 RoleBinding                         │ │
│  │                                                                   │ │
│  │  4. Bootstrap Token 权限                                         │ │
│  │     • ClusterRole: system:node-bootstrapper                      │ │
│  │     • 权限: 创建 CSR                                             │ │
│  │                                                                   │ │
│  │     • ClusterRole: system:certificates.k8s.io:certificatesigningrequests:nodeclient │
│  │     • 权限: 自动批准节点客户端 CSR                               │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    准入控制器配置                                  │ │
│  │                                                                   │ │
│  │  默认启用的准入控制器:                                            │ │
│  │                                                                   │ │
│  │  1. NodeRestriction                                               │ │
│  │     • 限制 kubelet 只能修改自己节点的资源                         │ │
│  │     • 防止 kubelet 修改其他节点的状态                             │ │
│  │                                                                   │ │
│  │  2. ServiceAccount                                                │ │
│  │     • 自动为 Pod 挂载 ServiceAccount                             │ │
│  │     • 自动创建 ServiceAccount token                              │ │
│  │                                                                   │ │
│  │  3. NamespaceLifecycle                                            │ │
│  │     • 防止删除系统命名空间                                        │ │
│  │     • 删除命名空间时清理资源                                      │ │
│  │                                                                   │ │
│  │  4. LimitRanger                                                   │ │
│  │     • 为没有资源限制的 Pod 设置默认限制                           │ │
│  │     • 确保资源配额生效                                            │ │
│  │                                                                   │ │
│  │  5. ResourceQuota                                                 │ │
│  │     • 限制命名空间的资源使用量                                    │ │
│  │     • 防止资源过度使用                                            │ │
│  │                                                                   │ │
│  │  6. PodSecurity (替代 PodSecurityPolicy)                         │ │
│  │     • 强制执行 Pod 安全标准                                       │ │
│  │     • 限制特权容器                                                │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### 14.2 安全加固建议
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    安全加固建议                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    控制平面安全                                    │ │
│  │                                                                   │ │
│  │  1. API Server                                                    │ │
│  │     • 启用审计日志: --audit-log-path                             │ │
│  │     • 限制匿名访问: --anonymous-auth=false                       │ │
│  │     • 启用加密: --encryption-provider-config                     │ │
│  │     • 限制端口访问: --bind-address                               │ │
│  │                                                                   │ │
│  │  2. Etcd                                                          │ │
│  │     • 启用TLS认证: --client-cert-auth                            │ │
│  │     • 限制访问: --listen-client-urls 绑定到本地                  │ │
│  │     • 定期备份                                                    │ │
│  │     • 使用专用磁盘                                                │ │
│  │                                                                   │ │
│  │  3. Controller Manager                                            │ │
│  │     • 使用 SA 凭证: --use-service-account-credentials=true       │ │
│  │     • 限制签名权限: 绑定到特定的 SA                              │ │
│  │                                                                   │ │
│  │  4. Scheduler                                                     │ │
│  │     • 限制绑定地址: --bind-address                               │ │
│  │     • 启用领导者选举: --leader-elect                             │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    节点安全                                        │ │
│  │                                                                   │ │
│  │  1. Kubelet                                                       │ │
│  │     • 禁用只读端口: --read-only-port=0                           │ │
│  │     • 启用认证: --authentication-token-webhook                   │ │
│  │     • 启用授权: --authorization-mode=Webhook                     │ │
│  │     • 限制匿名访问: --anonymous-auth=false                       │ │
│  │     • 启用证书轮换: --rotate-certificates                        │ │
│  │                                                                   │ │
│  │  2. 容器运行时                                                    │ │
│  │     • 启用 Pod Security Standards                                 │ │
│  │     • 限制特权容器                                                │ │
│  │     • 使用只读根文件系统                                          │ │
│  │                                                                   │ │
│  │  3. 网络安全                                                      │ │
│  │     • 配置网络策略                                                │ │
│  │     • 限制节点间通信                                              │ │
│  │     • 使用 NetworkPolicy                                          │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    证书安全                                        │ │
│  │                                                                   │ │
│  │  1. 证书管理                                                      │ │
│  │     • 定期检查证书有效期                                          │ │
│  │     • 在过期前轮换证书                                            │ │
│  │     • 使用强密钥 (RSA 2048+ 或 ECDSA)                            │ │
│  │                                                                   │ │
│  │  2. 私钥保护                                                      │ │
│  │     • 限制私钥文件权限 (600)                                      │ │
│  │     • 不要在版本控制中存储私钥                                    │ │
│  │     • 使用硬件安全模块 (HSM) 存储私钥 (可选)                      │ │
│  │                                                                   │ │
│  │  3. CA 保护                                                       │ │
│  │     • 离线存储 CA 私钥                                            │ │
│  │     • 使用中间 CA 签发证书                                        │ │
│  │     • 记录证书签发日志                                            │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
## 十五、故障排查设计
### 15.1 常见问题诊断流程
```plainText
┌─────────────────────────────────────────────────────────────────────────┐
│                    故障排查流程                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    初始化失败排查                                  │ │
│  │                                                                   │ │
│  │  1. Preflight 失败                                                │ │
│  │     检查项:                                                       │ │
│  │     ├─ 系统要求: 内存、CPU、内核版本                             │ │
│  │     ├─ 端口占用: netstat -tlnp | grep <port>                     │ │
│  │     ├─ 容器运行时: systemctl status containerd/docker            │ │
│  │     └─ 已有配置: ls /etc/kubernetes/                             │ │
│  │                                                                   │ │
│  │     解决方案:                                                     │ │
│  │     • kubeadm reset 清理环境                                     │ │
│  │     • 释放占用端口                                                │ │
│  │     • 安装/启动容器运行时                                         │ │
│  │                                                                   │ │
│  │  2. 证书生成失败                                                  │ │
│  │     检查项:                                                       │ │
│  │     ├─ 磁盘空间: df -h /etc/kubernetes                           │ │
│  │     ├─ 文件权限: ls -la /etc/kubernetes/pki                      │ │
│  │     └─ 已有证书: 是否与现有证书冲突                               │ │
│  │                                                                   │ │
│  │     解决方案:                                                     │ │
│  │     • 清理旧证书: rm -rf /etc/kubernetes/pki/*                   │ │
│  │     • 检查磁盘空间                                                │ │
│  │     • 使用 --ignore-preflight-errors 跳过检查                    │ │
│  │                                                                   │ │
│  │  3. 控制平面启动失败                                              │ │
│  │     检查项:                                                       │ │
│  │     ├─ 静态Pod清单: ls /etc/kubernetes/manifests/                │ │
│  │     ├─ 容器状态: crictl ps -a                                    │ │
│  │     ├─ 容器日志: crictl logs <container-id>                      │ │
│  │     ├─ Kubelet日志: journalctl -u kubelet                        │ │
│  │     └─ Etcd状态: etcdctl endpoint health                         │ │
│  │                                                                   │ │
│  │     解决方案:                                                     │ │
│  │     • 检查镜像是否拉取成功                                        │ │
│  │     • 检查证书和配置文件                                          │ │
│  │     • 检查网络连通性                                              │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    节点加入失败排查                                │ │
│  │                                                                   │ │
│  │  1. Token 无效                                                    │ │
│  │     检查项:                                                       │ │
│  │     ├─ Token是否存在: kubeadm token list                         │ │
│  │     ├─ Token是否过期: 检查 expiration 字段                       │ │
│  │     └─ CA证书哈希: openssl x509 -pubkey -in ca.crt | ...         │ │
│  │                                                                   │ │
│  │     解决方案:                                                     │ │
│  │     • 创建新Token: kubeadm token create --print-join-command     │ │
│  │     • 使用正确的CA证书哈希                                        │ │
│  │                                                                   │ │
│  │  2. CSR 未批准                                                    │ │

# 
