# KubeadmConfig 功能列表

## 一、核心定位
KubeadmConfig 是 CAPI 的 **Bootstrap Provider**，负责为 Machine 生成引导数据（cloud-init/Ignition），使节点能够加入 Kubernetes 集群。

## 二、完整功能列表

### 1. 集群初始化配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **ClusterConfiguration** | 集群级别配置 | `spec.clusterConfiguration` |
| **InitConfiguration** | kubeadm init 专用配置 | `spec.initConfiguration` |
| **Kubernetes 版本** | 目标 K8s 版本 | `spec.clusterConfiguration.kubernetesVersion` |
| **控制面端点** | 控制面访问地址 | `spec.clusterConfiguration.controlPlaneEndpoint` |
| **网络配置** | Pod/Service CIDR、DNS 域名 | `spec.clusterConfiguration.networking` |
| **镜像仓库** | 容器镜像拉取地址 | `spec.clusterConfiguration.imageRepository` |
| **功能开关** | K8s 特性门控 | `spec.clusterConfiguration.featureGates` |

### 2. 控制面组件配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **API Server 配置** | extraArgs、extraVolumes、extraEnvs、certSANs | `spec.clusterConfiguration.apiServer` |
| **Controller Manager 配置** | extraArgs、extraVolumes、extraEnvs | `spec.clusterConfiguration.controllerManager` |
| **Scheduler 配置** | extraArgs、extraVolumes、extraEnvs | `spec.clusterConfiguration.scheduler` |
| **跳过阶段** | 跳过 kubeadm 特定阶段 | `spec.initConfiguration.skipPhases` |
| **Patches** | Static Pod 补丁目录 | `spec.initConfiguration.patches` |

### 3. etcd 配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **Stacked etcd** | 本地 etcd 配置 | `spec.clusterConfiguration.etcd.local` |
| **External etcd** | 外部 etcd 集群连接 | `spec.clusterConfiguration.etcd.external` |
| **etcd 数据目录** | etcd 数据存储路径 | `spec.clusterConfiguration.etcd.local.dataDir` |
| **etcd 镜像** | etcd 镜像仓库/标签 | `spec.clusterConfiguration.etcd.local.imageRepository` |
| **etcd 证书 SAN** | etcd server/peer 证书 SAN | `spec.clusterConfiguration.etcd.local.serverCertSANs` |

### 4. 节点注册配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **节点名称** | 注册到集群的节点名 | `spec.initConfiguration.nodeRegistration.name` |
| **CRI Socket** | 容器运行时接口 | `spec.initConfiguration.nodeRegistration.criSocket` |
| **节点污点** | 注册时应用的污点 | `spec.initConfiguration.nodeRegistration.taints` |
| **Kubelet 额外参数** | 传递给 kubelet 的参数 | `spec.initConfiguration.nodeRegistration.kubeletExtraArgs` |
| **忽略预检错误** | 跳过的预检项 | `spec.initConfiguration.nodeRegistration.ignorePreflightErrors` |
| **镜像拉取策略** | 镜像拉取行为 | `spec.initConfiguration.nodeRegistration.imagePullPolicy` |

### 5. 节点加入配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **JoinConfiguration** | kubeadm join 配置 | `spec.joinConfiguration` |
| **发现方式** | Token 或文件发现 | `spec.joinConfiguration.discovery` |
| **控制面加入** | 作为控制面节点加入 | `spec.joinConfiguration.controlPlane` |
| **CA 证书路径** | CA 证书位置 | `spec.joinConfiguration.caCertPath` |

### 6. 证书管理
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **证书生成** | 自动生成集群证书 | 内置逻辑 |
| **证书目录** | 证书存储路径 | `spec.clusterConfiguration.certificatesDir` |
| **证书有效期** | 非 CA 证书有效期 | `spec.clusterConfiguration.certificateValidityPeriodDays` |
| **CA 证书有效期** | CA 证书有效期 | `spec.clusterConfiguration.caCertificateValidityPeriodDays` |
| **加密算法** | 非对称加密算法类型 | `spec.clusterConfiguration.encryptionAlgorithm` |
| **证书发现** | 查找或生成证书 | 控制器逻辑 |

### 7. 文件与脚本
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **文件写入** | 在节点上创建文件 | `spec.files` |
| **文件内容** | 内联内容或引用 Secret | `spec.files[].content` / `spec.files[].contentFrom` |
| **文件权限** | 文件所有权和权限 | `spec.files[].owner` / `spec.files[].permissions` |
| **文件编码** | base64/gzip 编码 | `spec.files[].encoding` |
| **preKubeadmCommands** | kubeadm 执行前运行的命令 | `spec.preKubeadmCommands` |
| **postKubeadmCommands** | kubeadm 执行后运行的命令 | `spec.postKubeadmCommands` |
| **bootCommands** | 启动时运行的命令 (cloud-init) | `spec.bootCommands` |

### 8. 用户管理
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **用户创建** | 在节点上创建用户 | `spec.users` |
| **用户密码** | 明文或引用 Secret | `spec.users[].passwd` / `spec.users[].passwdFrom` |
| **SSH 密钥** | 授权 SSH 公钥 | `spec.users[].sshAuthorizedKeys` |
| **用户组** | 附加用户组 | `spec.users[].groups` |
| **Sudo 权限** | sudo 配置 | `spec.users[].sudo` |
| **Shell** | 用户默认 shell | `spec.users[].shell` |

### 9. NTP 配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **NTP 服务器** | NTP 服务器列表 | `spec.ntp.servers` |
| **NTP 启用** | 是否启用 NTP | `spec.ntp.enabled` |

### 10. 磁盘与文件系统
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **分区配置** | 磁盘分区布局 | `spec.diskSetup.partitions` |
| **文件系统** | 文件系统创建 | `spec.diskSetup.filesystems` |
| **挂载点** | 文件系统挂载 | `spec.mounts` |
| **分区百分比** | 分区占用磁盘百分比 | `spec.diskSetup.partitions[].diskLayout[].percentage` |
| **分区类型** | 分区类型 (Linux/Swap/RAID 等) | `spec.diskSetup.partitions[].diskLayout[].partitionType` |

### 11. Bootstrap Token
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **Token 创建** | 创建引导令牌 | `spec.initConfiguration.bootstrapTokens` |
| **Token TTL** | 令牌有效期 | `spec.initConfiguration.bootstrapTokens[].ttlSeconds` |
| **Token 用途** | 签名/认证 | `spec.initConfiguration.bootstrapTokens[].usages` |
| **Token 组** | 认证组 | `spec.initConfiguration.bootstrapTokens[].groups` |

### 12. DNS 配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **CoreDNS 镜像** | CoreDNS 镜像仓库/标签 | `spec.clusterConfiguration.dns.imageRepository` / `imageTag` |
| **禁用 CoreDNS** | 是否跳过 CoreDNS 安装 | `spec.clusterConfiguration.dns.disabled` |

### 13. Kube-proxy 配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **禁用 Kube-proxy** | 是否跳过 kube-proxy 安装 | `spec.clusterConfiguration.proxy.disabled` |

### 14. 超时配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **控制面健康检查超时** | 等待控制面组件健康的时间 | `spec.initConfiguration.timeouts.controlPlaneComponentHealthCheckSeconds` |
| **Kubelet 健康检查超时** | 等待 kubelet 健康的时间 | `spec.initConfiguration.timeouts.kubeletHealthCheckSeconds` |
| **API 调用超时** | K8s API 调用超时 | `spec.initConfiguration.timeouts.kubernetesAPICallSeconds` |
| **etcd API 超时** | etcd API 调用超时 | `spec.initConfiguration.timeouts.etcdAPICallSeconds` |
| **TLS Bootstrap 超时** | TLS Bootstrap 超时 | `spec.initConfiguration.timeouts.tlsBootstrapSeconds` |
| **发现超时** | 节点发现超时 | `spec.initConfiguration.timeouts.discoverySeconds` |

### 15. 输出格式
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **cloud-config** | 默认 cloud-init 格式 | `spec.format: cloud-config` |
| **Ignition** | Ignition 格式 (需 feature gate) | `spec.format: ignition` |
| **Ignition 配置** | Ignition 特定配置 | `spec.ignition` |
| **Container Linux Config** | CLC 额外配置 | `spec.ignition.containerLinuxConfig` |

### 16. Kubelet 配置
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **KubeletConfiguration** | Kubelet 配置 (ConfigMap 级别) | `spec.kubeletConfig` |
| **最大 Pod 数** | 节点最大 Pod 数 | `spec.kubeletConfig.maxPods` |
| **容器日志** | 容器日志大小/数量 | `spec.kubeletConfig.containerLogMaxSize` |
| **节点标签** | 通过 extraArgs 添加 | `spec.initConfiguration.nodeRegistration.kubeletExtraArgs` |

### 17. 状态与条件
| 功能 | 说明 | 对应字段 |
|------|------|----------|
| **Ready 条件** | 配置是否就绪 | `status.conditions` |
| **DataSecretAvailable** | Bootstrap Secret 是否可用 | `status.conditions` |
| **CertificatesAvailable** | 证书是否可用 | `status.conditions` |
| **DataSecretName** | 生成的 Secret 名称 | `status.dataSecretName` |
| **DataSecretCreated** | Secret 是否已创建 | `status.initialization.dataSecretCreated` |

### 18. 控制器行为
| 功能 | 说明 |
|------|------|
| **Init 锁** | 确保只有一个节点执行 kubeadm init |
| **证书查找/生成** | 查找现有证书或生成新证书 |
| **Token 管理** | 创建和管理 bootstrap token |
| **Cloud-init 生成** | 生成 cloud-init 脚本 |
| **Ignition 生成** | 生成 Ignition 配置 |
| **Secret 存储** | 将 bootstrap 数据存储为 Secret |
| **Owner 引用** | 设置正确的 owner references |

## 三、完整配置示例
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
kind: KubeadmConfig
metadata:
  name: my-cluster-config
  namespace: default
spec:
  # 集群配置
  clusterConfiguration:
    kubernetesVersion: v1.31.0
    clusterName: my-cluster
    controlPlaneEndpoint: "lb.example.com:6443"
    imageRepository: registry.example.com/k8s
    certificatesDir: /etc/kubernetes/pki
    certificateValidityPeriodDays: 365
    caCertificateValidityPeriodDays: 3650
    encryptionAlgorithm: RSA-2048
    featureGates:
      RotateKubeletServerCertificate: true
    
    networking:
      podSubnet: "10.244.0.0/16"
      serviceSubnet: "10.96.0.0/12"
      dnsDomain: "cluster.local"
    
    apiServer:
      extraArgs:
        - name: "authorization-mode"
          value: "Node,RBAC"
        - name: "audit-log-path"
          value: "/var/log/kubernetes/audit.log"
        - name: "max-requests-inflight"
          value: "2000"
      certSANs:
        - "lb.example.com"
        - "192.168.1.100"
      extraVolumes:
        - name: "audit-config"
          hostPath: "/etc/kubernetes/audit-policy.yaml"
          mountPath: "/etc/kubernetes/audit-policy.yaml"
          readOnly: true
          pathType: File
    
    controllerManager:
      extraArgs:
        - name: "leader-elect"
          value: "true"
        - name: "node-cidr-mask-size"
          value: "24"
    
    scheduler:
      extraArgs:
        - name: "leader-elect"
          value: "true"
    
    etcd:
      local:
        dataDir: /var/lib/etcd
        extraArgs:
          - name: "listen-metrics-urls"
            value: "http://0.0.0.0:2381"
    
    dns:
      imageRepository: registry.example.com/coredns
      imageTag: "1.11.1"
    
    proxy:
      disabled: false
  
  # Init 配置
  initConfiguration:
    localAPIEndpoint:
      advertiseAddress: "192.168.1.101"
      bindPort: 6443
    skipPhases:
      - "addon/kube-proxy"
    patches:
      directory: "/etc/kubernetes/patches"
    timeouts:
      controlPlaneComponentHealthCheckSeconds: 240
      kubeletHealthCheckSeconds: 240
    
    nodeRegistration:
      name: "master-01"
      criSocket: "unix:///run/containerd/containerd.sock"
      taints:
        - key: "node-role.kubernetes.io/control-plane"
          effect: "NoSchedule"
      kubeletExtraArgs:
        - name: "max-pods"
          value: "250"
        - name: "kube-reserved"
          value: "cpu=500m,memory=512Mi"
        - name: "system-reserved"
          value: "cpu=200m,memory=256Mi"
        - name: "eviction-hard"
          value: "memory.available<500Mi,nodefs.available<10%"
      ignorePreflightErrors:
        - "Swap"
      imagePullPolicy: IfNotPresent
    
    bootstrapTokens:
      - token: "abcdef.0123456789abcdef"
        description: "Bootstrap token for worker nodes"
        ttlSeconds: 86400
        usages:
          - "signing"
          - "authentication"
  
  # 文件配置
  files:
    - path: /etc/kubernetes/audit-policy.yaml
      content: |
        apiVersion: audit.k8s.io/v1
        kind: Policy
        rules:
          - level: Metadata
      owner: root:root
      permissions: "0644"
    - path: /etc/containerd/config.toml
      contentFrom:
        secret:
          name: containerd-config
          key: config.toml
      owner: root:root
      permissions: "0644"
  
  # 命令配置
  preKubeadmCommands:
    - |
      #!/bin/bash
      # 配置 NTP
      cat > /etc/chrony.conf <<EOF
      server ntp.example.com iburst
      EOF
      systemctl enable --now chronyd
      
      # 禁用 Swap
      swapoff -a
      sed -i '/swap/d' /etc/fstab
      
      # 加载内核模块
      modprobe br_netfilter
      modprobe overlay
      cat > /etc/modules-load.d/k8s.conf <<EOF
      br_netfilter
      overlay
      EOF
  
  postKubeadmCommands:
    - |
      #!/bin/bash
      # 安装 Calico CNI
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
  
  # 用户配置
  users:
    - name: ubuntu
      passwd: "$6$rounds=4096$..."
      sshAuthorizedKeys:
        - "ssh-rsa AAAAB3..."
      sudo: "ALL=(ALL) NOPASSWD:ALL"
      shell: /bin/bash
  
  # NTP 配置
  ntp:
    enabled: true
    servers:
      - "ntp1.example.com"
      - "ntp2.example.com"
  
  # 磁盘配置
  diskSetup:
    partitions:
      - device: /dev/sdb
        layout: true
        tableType: gpt
    filesystems:
      - device: /dev/sdb1
        filesystem: ext4
        label: DATA
  
  # 挂载配置
  mounts:
    - - /dev/sdb1
      - /mnt/data
      - ext4
      - defaults
      - "0"
      - "2"
  
  # 输出格式
  format: cloud-config
  
  # 日志级别
  verbosity: 2
```

## 四、与 KubeadmControlPlane 的关系
| 维度 | KubeadmConfig | KubeadmControlPlane |
|------|---------------|---------------------|
| **定位** | 单节点引导配置 | 控制面生命周期管理 |
| **作用范围** | 单个 Machine | 整个控制面 (多 Machine) |
| **生命周期** | 一次性使用 (生成 Secret 后) | 持续管理 (扩缩容/升级/修复) |
| **包含关系** | KCP 内部引用 KubeadmConfigSpec | 包含 KubeadmConfigSpec + 控制面管理逻辑 |

**KCP 使用 KubeadmConfig 的方式**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  kubeadmConfigSpec:  # 这就是 KubeadmConfigSpec
    clusterConfiguration:
      ...

    initConfiguration:
      ...
```
