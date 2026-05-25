# kubeadm 完整配置列表
基于 CAPI 源码中的 `upstreamv1beta4/types.go`（对应 kubeadm v1beta4 API），以下是 **kubeadm 完整配置列表**：

## 一、配置类型总览
kubeadm 配置分为 **4 种类型**：

| 类型 | 用途 | 是否存入 ConfigMap |
|------|------|-------------------|
| **ClusterConfiguration** | 集群级别配置 | ✅ 是 |
| **InitConfiguration** | `kubeadm init` 专用配置 | ❌ 否 |
| **JoinConfiguration** | `kubeadm join` 专用配置 | ❌ 否 |
| **ResetConfiguration** | `kubeadm reset` 专用配置 | ❌ 否 |

## 二、ClusterConfiguration（集群配置）
存储于 `kube-system/kubeadm-config` ConfigMap，升级时持久使用。

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `kind` | string | 对象类型 | `ClusterConfiguration` |
| `apiVersion` | string | API 版本 | `kubeadm.k8s.io/v1beta4` |
| `kubernetesVersion` | string | 目标 K8s 版本 | 自动检测 |
| `clusterName` | string | 集群名称 | `""` |
| `certificatesDir` | string | 证书存储目录 | `/etc/kubernetes/pki` |
| `imageRepository` | string | 镜像仓库地址 | `registry.k8s.io` |
| `controlPlaneEndpoint` | string | 控制面端点 (IP:Port 或 DNS) | `""` |
| `featureGates` | map[string]bool | 功能开关 | `{}` |
| `encryptionAlgorithm` | string | 非对称加密算法 | `RSA-2048` |
| `certificateValidityPeriod` | Duration | 非 CA 证书有效期 | `8760h` (1年) |
| `caCertificateValidityPeriod` | Duration | CA 证书有效期 | `87600h` (10年) |

### 2.1 Networking（网络配置）
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `networking.serviceSubnet` | Service CIDR | `10.96.0.0/12` |
| `networking.podSubnet` | Pod CIDR | `""` (需手动指定) |
| `networking.dnsDomain` | Cluster DNS 域名 | `cluster.local` |

### 2.2 APIServer
| 字段 | 说明 |
|------|------|
| `apiServer.extraArgs` | 传递给 API Server 的额外参数 (覆盖默认参数) |
| `apiServer.extraVolumes` | 挂载的主机卷 |
| `apiServer.extraEnvs` | 传递给 API Server 的环境变量 (≥1.31) |
| `apiServer.certSANs` | API Server 证书的额外 SAN |

**extraArgs 常用参数**:
```yaml
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
    enable-admission-plugins: "NodeRestriction,PodSecurityPolicy"
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
    encryption-provider-config: "/etc/kubernetes/encryption-config.yaml"
    max-requests-inflight: "2000"
    max-mutating-requests-inflight: "1000"
```

### 2.3 ControllerManager
| 字段 | 说明 |
|------|------|
| `controllerManager.extraArgs` | 额外参数 |
| `controllerManager.extraVolumes` | 挂载卷 |
| `controllerManager.extraEnvs` | 环境变量 (≥1.31) |

**extraArgs 常用参数**:
```yaml
controllerManager:
  extraArgs:
    leader-elect: "true"
    node-cidr-mask-size: "24"
    horizontal-pod-autoscaler-sync-period: "15s"
    terminated-pod-gc-threshold: "100"
```

### 2.4 Scheduler
| 字段 | 说明 |
|------|------|
| `scheduler.extraArgs` | 额外参数 |
| `scheduler.extraVolumes` | 挂载卷 |
| `scheduler.extraEnvs` | 环境变量 (≥1.31) |

**extraArgs 常用参数**:
```yaml
scheduler:
  extraArgs:
    leader-elect: "true"
    profiling: "false"
```

### 2.5 Etcd
| 字段 | 说明 |
|------|------|
| `etcd.local` | 本地 etcd 配置 (与 external 互斥) |
| `etcd.external` | 外部 etcd 配置 (与 local 互斥) |

**Local Etcd**:
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `etcd.local.imageRepository` | etcd 镜像仓库 | 继承 `imageRepository` |
| `etcd.local.imageTag` | etcd 镜像标签 | 自动选择 |
| `etcd.local.dataDir` | etcd 数据目录 | `/var/lib/etcd` |
| `etcd.local.extraArgs` | etcd 额外参数 | `{}` |
| `etcd.local.extraEnvs` | etcd 环境变量 (≥1.31) | `{}` |
| `etcd.local.serverCertSANs` | etcd server 证书 SAN | `{}` |
| `etcd.local.peerCertSANs` | etcd peer 证书 SAN | `{}` |

**External Etcd**:
| 字段 | 说明 | 是否必填 |
|------|------|----------|
| `etcd.external.endpoints` | etcd 成员地址列表 | ✅ 必填 |
| `etcd.external.caFile` | CA 证书路径 | ✅ 必填 |
| `etcd.external.certFile` | 客户端证书路径 | ✅ 必填 |
| `etcd.external.keyFile` | 客户端密钥路径 | ✅ 必填 |

### 2.6 DNS
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `dns.imageRepository` | CoreDNS 镜像仓库 | 继承 `imageRepository` |
| `dns.imageTag` | CoreDNS 镜像标签 | 自动选择 |
| `dns.disabled` | 是否禁用 CoreDNS | `false` |

### 2.7 Proxy
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `proxy.disabled` | 是否禁用 kube-proxy | `false` |

## 三、InitConfiguration（初始化配置）
仅 `kubeadm init` 时使用，**不**存入 ConfigMap。

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `kind` | string | 对象类型 | `InitConfiguration` |
| `apiVersion` | string | API 版本 | `kubeadm.k8s.io/v1beta4` |
| `dryRun` | bool | 是否仅输出不执行 | `false` |
| `certificateKey` | string | 证书加密密钥 (用于证书上传) | `""` |
| `skipPhases` | []string | 跳过的阶段 | `[]` |
| `bootstrapTokens` | []BootstrapToken | 创建的引导令牌 | `[]` |
| `localAPIEndpoint` | APIEndpoint | 本地 API 端点 | 自动检测 |
| `nodeRegistration` | NodeRegistrationOptions | 节点注册配置 | - |
| `patches` | Patches | Static Pod 补丁目录 | - |
| `timeouts` | Timeouts | 超时配置 | - |

### 3.1 APIEndpoint
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `localAPIEndpoint.advertiseAddress` | API Server 广播地址 | 默认接口 IP |
| `localAPIEndpoint.bindPort` | API Server 端口 | `6443` |

### 3.2 BootstrapToken
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `bootstrapTokens[].token` | 令牌值 (格式: `abcdef.0123456789abcdef`) | 自动生成 |
| `bootstrapTokens[].description` | 描述信息 | `""` |
| `bootstrapTokens[].ttl` | 令牌有效期 | `24h` |
| `bootstrapTokens[].expires` | 过期时间戳 | 根据 TTL 计算 |
| `bootstrapTokens[].usages` | 用途列表 | `["signing", "authentication"]` |
| `bootstrapTokens[].groups` | 认证组 | `["system:bootstrappers:kubeadm:default-node-token"]` |

## 四、JoinConfiguration（加入配置）
仅 `kubeadm join` 时使用。

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `kind` | string | 对象类型 | `JoinConfiguration` |
| `apiVersion` | string | API 版本 | `kubeadm.k8s.io/v1beta4` |
| `dryRun` | bool | 是否仅输出不执行 | `false` |
| `caCertPath` | string | CA 证书路径 | `/etc/kubernetes/pki/ca.crt` |
| `skipPhases` | []string | 跳过的阶段 | `[]` |
| `discovery` | Discovery | 发现配置 | - |
| `controlPlane` | JoinControlPlane | 控制面加入配置 | - |
| `nodeRegistration` | NodeRegistrationOptions | 节点注册配置 | - |
| `patches` | Patches | Static Pod 补丁目录 | - |
| `timeouts` | Timeouts | 超时配置 | - |

### 4.1 Discovery
| 字段 | 说明 |
|------|------|
| `discovery.bootstrapToken` | Token 发现方式 |
| `discovery.file` | 文件发现方式 (kubeconfig) |
| `discovery.tlsBootstrapToken` | TLS Bootstrap 令牌 |

**BootstrapTokenDiscovery**:
| 字段 | 说明 |
|------|------|
| `token` | 验证令牌 |
| `apiServerEndpoint` | API Server 地址 |
| `caCertHashes` | CA 证书哈希 (SHA256) |
| `unsafeSkipCAVerification` | 跳过 CA 验证 (不安全) |

**FileDiscovery**:
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `kubeConfigPath` | kubeconfig 文件路径 | - |
| `kubeConfig.cluster.server` | API Server 地址 | 自动填充 |
| `kubeConfig.cluster.tlsServerName` | TLS 服务器名称 | - |
| `kubeConfig.cluster.insecureSkipTLSVerify` | 跳过 TLS 验证 | `false` |
| `kubeConfig.cluster.certificateAuthorityData` | CA 数据 | 自动填充 |
| `kubeConfig.cluster.proxyURL` | 代理 URL | - |
| `kubeConfig.user.authProvider` | 认证提供者 | - |
| `kubeConfig.user.exec` | Exec 认证插件 | - |

### 4.2 JoinControlPlane
| 字段 | 说明 |
|------|------|
| `controlPlane.localAPIEndpoint` | 本地 API 端点 |
| `controlPlane.certificateKey` | 证书解密密钥 |

## 五、NodeRegistrationOptions（节点注册配置）
在 InitConfiguration 和 JoinConfiguration 中共用。

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `name` | string | 节点名称 | 主机名 |
| `criSocket` | string | CRI Socket 路径 | 自动检测 |
| `taints` | []Taint | 节点污点 | `node-role.kubernetes.io/control-plane:NoSchedule` |
| `kubeletExtraArgs` | []Arg | Kubelet 额外参数 | `[]` |
| `ignorePreflightErrors` | []string | 忽略的预检错误 | `[]` |
| `imagePullPolicy` | PullPolicy | 镜像拉取策略 | `IfNotPresent` |
| `imagePullSerial` | bool | 是否串行拉取镜像 (≥1.31) | `true` |

**kubeletExtraArgs 常用参数**:
```yaml
nodeRegistration:
  kubeletExtraArgs:
    max-pods: "250"
    kube-reserved: "cpu=500m,memory=512Mi"
    system-reserved: "cpu=200m,memory=256Mi"
    eviction-hard: "memory.available<500Mi,nodefs.available<10%"
    feature-gates: "RotateKubeletServerCertificate=true"
    serialize-image-pull: "false"
    image-gc-high-threshold: "85"
    image-gc-low-threshold: "80"
```

## 六、Patches（补丁配置）
| 字段 | 说明 |
|------|------|
| `patches.directory` | 补丁文件目录 |

**补丁文件命名规则**: `target[suffix][+patchtype].extension`

| 部分 | 说明 | 示例 |
|------|------|------|
| `target` | 目标组件 | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`, `kubeletconfiguration`, `corednsdeployment` |
| `suffix` | 可选后缀 (字母数字排序) | `0`, `custom` |
| `patchtype` | 补丁类型 | `strategic` (默认), `merge`, `json` |
| `extension` | 文件扩展名 | `json`, `yaml` |

**示例**:
```
/etc/kubernetes/patches/
├── kube-apiserver0+merge.yaml
├── kube-apiserver+json.json
├── etcd.json
└── corednsdeployment.yaml
```

## 七、Timeouts（超时配置）
| 字段 | 说明 | 默认值 |
|------|------|--------|
| `timeouts.controlPlaneComponentHealthCheck` | 控制面组件健康检查超时 | `4m` |
| `timeouts.kubeletHealthCheck` | Kubelet 健康检查超时 | `4m` |
| `timeouts.kubernetesAPICall` | K8s API 调用超时 | `1m` |
| `timeouts.etcdAPICall` | etcd API 调用超时 | `2m` |
| `timeouts.tlsBootstrap` | TLS Bootstrap 超时 | `5m` |
| `timeouts.discovery` | 发现阶段超时 | `5m` |
| `timeouts.upgradeManifests` | 升级 Manifests 超时 | `5m` |

## 八、辅助类型

### HostPathMount（主机卷挂载）
| 字段 | 说明 |
|------|------|
| `name` | 卷名称 |
| `hostPath` | 主机路径 |
| `mountPath` | 容器内挂载路径 |
| `readOnly` | 是否只读 |
| `pathType` | 路径类型 (`Directory`, `File`, `DirectoryOrCreate` 等) |

### Arg（参数）
```yaml
- name: "max-pods"
  value: "250"
```

### EnvVar（环境变量）
```yaml
- name: "HTTP_PROXY"
  value: "http://proxy.example.com:8080"
```

## 九、完整配置示例
```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.0
clusterName: my-cluster
controlPlaneEndpoint: "lb.example.com:6443"
imageRepository: registry.example.com/k8s
certificatesDir: /etc/kubernetes/pki
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
    audit-log-path: "/var/log/kubernetes/audit.log"
    max-requests-inflight: "2000"
  certSANs:
    - "lb.example.com"
    - "192.168.1.100"
controllerManager:
  extraArgs:
    leader-elect: "true"
    node-cidr-mask-size: "24"
scheduler:
  extraArgs:
    leader-elect: "true"
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      listen-metrics-urls: "http://0.0.0.0:2381"
featureGates:
  RotateKubeletServerCertificate: true
encryptionAlgorithm: RSA-2048
certificateValidityPeriod: 8760h
caCertificateValidityPeriod: 87600h
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.1.101"
  bindPort: 6443
nodeRegistration:
  name: "master-01"
  criSocket: "unix:///run/containerd/containerd.sock"
  kubeletExtraArgs:
    - name: "max-pods"
      value: "250"
    - name: "kube-reserved"
      value: "cpu=500m,memory=512Mi"
  ignorePreflightErrors:
    - "Swap"
bootstrapTokens:
  - token: "abcdef.0123456789abcdef"
    description: "Bootstrap token for worker nodes"
    ttl: "24h"
    usages:
      - "signing"
      - "authentication"
patches:
  directory: "/etc/kubernetes/patches"
timeouts:
  controlPlaneComponentHealthCheck: 4m
  kubeletHealthCheck: 4m
```
