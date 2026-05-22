# 需求
基于Cluster-API完成如下需求，请给出方案设计(可行性分析及技术选型)：
1. 用户提供机器列表(机器已完成OS的安装)，提供主机名、IP、用户名与密码等基本信息
2. 用户提供如下服务：NFS、NTP、镜像仓库、二进制安装源、chart仓库、外部负载均衡器
3. K8s控制面有如下要求：
   - etcd支持外接配置
   - etcd可通过标签指定要安装的节点，并进行自动化安装
   - api-server可通过标签指定要安装的节点，支持扩缩容,并可指定负载均衡器(kubeadm是否支持)
   - scheduler可通过标签指定要安装的节点，支持主备(kubeadm是否支持)
   - controller-manager可通过标签指定要安装的节点，支持主备(kubeadm是否支持)
   - kubelet对某些节点的配置需要支持定制化
   - containerd对某些节点的配置需要支持定制化
   - calico可通过cluster-api的扩展机制进行安装
   - coredns可通过cluster-api的扩展机制进行安装
   - kube-proxy 可通过cluster-api的扩展机制进行安装
4. 其它应用可通过cluster-api的扩展机制进行安装
   - 支持按照应用间的依赖关系进行拓扑化安装
5. 支持证书体系的自动轮转
6. 升级方案
   - 每个版本有个组件清单
   - 升级时根据组件清单进行升级
   - 升级异常时要能够回滚
   - 支持组件的原地升级
7. 在大规模集群下，api-server/controller-manager/scheduler需要分别部署在不同节点上
8. 分析使用的开源组件及其作用与限制
9. 在OS上安装kubeadm/kubelet/kubectl/containerd/ntp等二进制组件的设计
   - 支持kubeadm/kubelet/kubectl/containerd/ntp的原地升级
   - kubeadm/kubelet/kubectl/containerd/ntp也包含在版本的组件清单里
   - 失败回滚到上一版本的组件版本

# 基于 Cluster-API 的裸金属集群全生命周期管理方案（1）
## 一、总体架构
```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Management Cluster                              │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────┐ │
│  │  CAPI Core   │  │  CAPBM        │  │  Runtime Extensions           │ │
│  │  (内置)      │  │  (自研)       │  │   ┌──────────┐ ┌──────────────┐ │ │
│  │              │  │               │  │  │ Addon    │ │ Upgrade      │ │ │
│  │ - Cluster    │  │ - InfraCluster│  │  │ Manager  │ │ Orchestrator │ │ │
│  │ - Machine    │  │ - InfraMachine│  │  └──────────┘ └──────────────┘ │ │
│  │ - MD/MS      │  │ - SSH Client  │  │  ┌──────────┐ ┌──────────────┐ │ │
│  │ - ClusterClass│ │ - IPMI/RF     │  │  │ Cert     │ │ Topology     │ │ │
│  │ - Topology   │  └──────────────┘   │  │ Rotator  │ │ Installer    │ │ │
│  └──────┬───────┘                     │  └──────────┘ └──────────────┘ │ │
│         │                             └────────────────────────────────┘ │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │                    Kubeadm Bootstrap + KCP                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │   │
│  │  │ KubeadmConfig│  │ KCP          │  │ In-Place Update       │  │   │
│  │  │ (bootstrap)  │  │ (controlplane)│ │ (experimental)        │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Workload Cluster (Bare Metal)                    │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ API Server  │  │ Scheduler   │  │ Controller  │  │    etcd        │  │
│  │ Nodes (N)   │  │ Nodes (M)   │  │ Manager (M) │  │  Nodes (K)     │  │
│  │             │  │             │  │             │  │                │  │
│  │ HA: 3-9     │  │ Active/     │  │ Active/     │  │ External or    │  │
│  │             │  │ Standby     │  │ Standby     │  │ Stacked        │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───────┬────────┘  │
│         │                │                │                  │          │
│  ┌──────▼────────────────▼────────────────▼──────────────────▼───────┐  │
│  │                    External Load Balancer                         │  │
│  │              (用户提供: HAProxy/F5/Nginx)                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Worker Nodes (MachineDeployments)              │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────────────┐  │  │
│  │  │ Calico  │ │CoreDNS  │ │KubeProxy│ │ Custom Apps (topology)  │  │  │
│  │  │ (CNI)   │ │ (DNS)   │ │(Proxy)  │ │                         │  │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## 二、需求逐项分析与技术方案

### 需求 1: 机器列表管理
**可行性**: ✅ 完全可行

**技术选型**: 自研 `CAPBM` (Cluster API Provider Bare Metal)

**核心 CRD 设计**:
```yaml
# BareMetalMachine - 单台机器
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalMachine
metadata:
  name: node-01
spec:
  hostName: "k8s-master-01"
  ipAddress: "192.168.1.101"
  sshPort: 22
  credentialsRef:
    name: node-01-credentials   # Secret包含username/password
    namespace: default
  powerManagement:              # 可选：电源管理
    type: "ipmi"                # ipmi | redfish | none
    address: "192.168.1.101:623"
    credentialsRef:
      name: node-01-bmc-credentials
---
# Secret - 凭据存储
apiVersion: v1
kind: Secret
metadata:
  name: node-01-credentials
type: Opaque
stringData:
  username: "root"
  password: "encrypted-password"
```
**关键实现**:
- 通过 SSH 连接已安装 OS 的裸金属机器
- `ProviderID` 格式: `baremetal://<hostname>`
- 机器就绪判定: SSH 可达 + 基础环境检查通过

### 需求 2: 外部服务集成
**可行性**: ✅ 完全可行

**技术方案**: ClusterClass Variables + KubeadmConfig preKubeadmCommands
```yaml
# ClusterClass 变量定义
variables:
  - name: externalServices
    schema:
      openAPIV3Schema:
        type: object
        properties:
          nfs:
            type: object
            properties:
              server: { type: string }
              shares: { type: array, items: { type: string } }
          ntp:
            type: object
            properties:
              servers: { type: array, items: { type: string } }
          registry:
            type: object
            properties:
              endpoint: { type: string }
              authSecret: { type: string }
          packageMirror:
            type: object
            properties:
              type: { type: string, enum: [apt, yum] }
              url: { type: string }
          helmRegistry:
            type: object
            properties:
              endpoint: { type: string }
          externalLoadBalancer:
            type: object
            properties:
              endpoint: { type: string }
              port: { type: integer, default: 6443 }
```

**注入方式**:
```yaml
kubeadmConfigSpec:
  preKubeadmCommands:
    - |
      #!/bin/bash
      set -euo pipefail
      
      # 1. 配置NTP
      cat > /etc/chrony.conf <<EOF
      {{ range .externalServices.ntp.servers }}
      server {{ . }} iburst
      {{ end }}
      makestep 1 3
      EOF
      systemctl enable --now chronyd
      
      # 2. 配置容器镜像仓库
      mkdir -p /etc/containerd
      containerd config default > /etc/containerd/config.toml
      sed -i 's|registry.k8s.io|{{ .externalServices.registry.endpoint }}|g' /etc/containerd/config.toml
      systemctl restart containerd
      
      # 3. 配置包管理器镜像源
      {{ if eq .externalServices.packageMirror.type "yum" }}
      cat > /etc/yum.repos.d/mirror.repo <<EOF
      [base]
      baseurl={{ .externalServices.packageMirror.url }}/base
      enabled=1
      gpgcheck=0
      EOF
      {{ end }}
      
      # 4. 挂载NFS共享存储
      mkdir -p /mnt/k8s-data
      mount -t nfs {{ .externalServices.nfs.server }}:{{ .externalServices.nfs.shares[0] }} /mnt/k8s-data
      echo "{{ .externalServices.nfs.server }}:{{ .externalServices.nfs.shares[0] }} /mnt/k8s-data nfs defaults 0 0" >> /etc/fstab
```

### 需求 3: K8s 控制面配置

#### 3.1 etcd 支持外接配置
**可行性**: ✅ kubeadm 原生支持
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    etcd:
      external:
        endpoints:
          - "https://etcd-01:2379"
          - "https://etcd-02:2379"
          - "https://etcd-03:2379"
        caFile: "/etc/kubernetes/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
```

#### 3.2 etcd 通过标签指定节点并自动化安装
**可行性**: ⚠️ 部分可行，有架构限制

**kubeadm 限制分析**:

| 模式 | 是否支持 | 说明 |
|------|----------|------|
| Stacked etcd (与控制面同节点) | ✅ 原生支持 | kubeadm 在每个 control plane 节点自动部署 etcd static pod |
| External etcd + 独立节点池 | ⚠️ 需扩展 | kubeadm 不管理 external etcd 节点的生命周期 |

**推荐方案**:

**方案 A: Stacked etcd (推荐用于中小规模)**
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3  # 每个节点同时运行 etcd + 控制面组件
  kubeadmConfigSpec:
    clusterConfiguration:
      etcd:
        local:
          dataDir: /var/lib/etcd
          extraArgs:
            - name: listen-metrics-urls
              value: "http://0.0.0.0:2381"
```

**方案 B: External etcd + MachineDeployment (大规模场景)**
```yaml
# 独立的 etcd 节点池
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  name: etcd-nodes
  labels:
    role: etcd
spec:
  replicas: 3
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
          kind: KubeadmConfig
          name: etcd-bootstrap-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: etcd-machine-template
---
# etcd 安装通过 Runtime Extension 实现
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: etcd-installer
spec:
  clientConfig:
    service:
      name: etcd-installer-service
      namespace: capi-system
```
**etcd 自动化安装流程**:
1. MachineDeployment 创建 etcd 节点
2. KubeadmConfig 通过 `postKubeadmCommands` 安装 etcd 二进制
3. Runtime Extension (`AfterControlPlaneInitialized` hook) 配置 etcd 集群
4. 将 etcd endpoints 注入 KCP 的 external etcd 配置

#### 3.3 API Server 节点指定、扩缩容、负载均衡器
**kubeadm 支持情况**: ❌ **kubeadm 不支持将 API Server 单独部署在特定节点**

**kubeadm 架构限制**:
- kubeadm 将 API Server、Scheduler、Controller Manager 作为 **static pods** 部署
- 所有加入 control plane 的节点 (`kubeadm join --control-plane`) **都会运行全部三个组件**
- **无法**通过 kubeadm 配置实现 "节点A只跑API Server，节点B只跑Scheduler"

**解决方案**:

**方案 A: 标准 kubeadm HA (组件同节点，推荐)**
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3  # 3个节点，每个节点运行 api-server + scheduler + controller-manager
  kubeadmConfigSpec:
    clusterConfiguration:
      controlPlaneEndpoint: "{{ .externalServices.externalLoadBalancer.endpoint }}:{{ .externalServices.externalLoadBalancer.port }}"
      apiServer:
        certSANs:
          - "{{ .externalServices.externalLoadBalancer.endpoint }}"
        extraArgs:
          - name: advertise-address
            value: "${LOCAL_IP}"
```
**扩缩容**: 直接修改 `spec.replicas`，KCP controller 自动处理

**负载均衡器**: 通过 `controlPlaneEndpoint` 指定用户提供的 LB 地址

**方案 B: 大规模分离部署 (需自定义 ControlPlane Provider)**

> ⚠️ 这是需求 7 的核心挑战，详见下文专门分析

#### 3.4 Scheduler 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    scheduler:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
        - name: leader-elect-resource-name
          value: "kube-scheduler"
```
**说明**: 
- kubeadm 部署的 scheduler 默认启用 leader election
- 多副本时，只有一个 active leader，其余为 standby
- 通过 `--leader-elect=true` 确保主备模式

#### 3.5 Controller Manager 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    controllerManager:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
        - name: leader-elect-resource-name
          value: "kube-controller-manager"
```
**说明**: 与 scheduler 相同，controller-manager 也通过 leader election 实现主备

#### 3.6 Kubelet 定制化配置
**可行性**: ✅ 完全支持
```yaml
kubeadmConfigSpec:
  initConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        - name: kube-reserved
          value: "cpu=1000m,memory=1Gi"
        - name: system-reserved
          value: "cpu=500m,memory=512Mi"
        - name: eviction-hard
          value: "memory.available<500Mi,nodefs.available<10%"
        - name: max-pods
          value: "250"
        - name: serialize-image-pull
          value: "false"
        - name: image-gc-high-threshold
          value: "85"
        - name: image-gc-low-threshold
          value: "80"
  # 全局 kubelet 配置通过 KubeletConfiguration
  kubeletConfig:
    maxPods: 250
    containerLogMaxSize: "100Mi"
    containerLogMaxFiles: 5
```

**节点差异化配置** (通过 ClusterClass Patch):
```yaml
patches:
  - name: kubelet-gpu-nodes
    definitions:
      - selector:
          apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
          kind: BareMetalMachineTemplate
          matchResources:
            machineDeploymentClass:
              names:
                - gpu-workers
        jsonPatches:
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/-
            value:
              name: "feature-gates"
              value: "DevicePlugins=true"
```

#### 3.7 Containerd 定制化配置
**可行性**: ✅ 通过 `files` + `preKubeadmCommands`
```yaml
kubeadmConfigSpec:
  files:
    - path: /etc/containerd/config.toml
      content: |
        version = 2
        [plugins]
          [plugins."io.containerd.grpc.v1.cri"]
            sandbox_image = "{{ .externalServices.registry.endpoint }}/pause:3.9"
            [plugins."io.containerd.grpc.v1.cri".containerd]
              default_runtime_name = "runc"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                runtime_type = "io.containerd.runc.v2"
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                  SystemdCgroup = true
            [plugins."io.containerd.grpc.v1.cri".registry]
              [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
                [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
                  endpoint = ["{{ .externalServices.registry.endpoint }}"]
      owner: "root:root"
      permissions: "0644"
  preKubeadmCommands:
    - systemctl daemon-reload && systemctl restart containerd
```

#### 3.8 Calico / CoreDNS / Kube-proxy 通过扩展机制安装
**可行性**: ✅ 完全可行

**方案 A: 使用 KCP 内置管理 (CoreDNS, Kube-proxy)**
```yaml
# 禁用内置的 CoreDNS/Kube-proxy，使用自定义版本
kubeadmConfigSpec:
  clusterConfiguration:
    dns:
      disabled: true        # 禁用内置 CoreDNS
      imageRepository: "{{ .externalServices.registry.endpoint }}"
      imageTag: "1.11.1"
    proxy:
      disabled: true        # 禁用内置 Kube-proxy
```

**方案 B: 使用 Runtime Extension (推荐，统一管理)**
```go
// AddonManager Runtime Extension
type AddonManager struct{}

func (a *AddonManager) AfterControlPlaneInitialized(
    ctx context.Context,
    req *runtimehooksv1.AfterControlPlaneInitializedRequest,
    resp *runtimehooksv1.AfterControlPlaneInitializedResponse,
) {
    client := a.getWorkloadClient(req.Cluster)
    
    // 按依赖顺序安装基础组件
    addons := []AddonDefinition{
        {
            Name:     "calico",
            Priority: 1,  // 最先安装 (CNI)
            Manifest: calicoManifests,
            WaitReady: true,
        },
        {
            Name:     "kube-proxy",
            Priority: 2,  // CNI 之后
            Manifest: kubeProxyManifests,
            DependsOn: []string{"calico"},
        },
        {
            Name:     "coredns",
            Priority: 3,
            Manifest: corednsManifests,
            DependsOn: []string{"calico"},
        },
    }
    
    for _, addon := range addons {
        if err := a.installAndWait(client, addon); err != nil {
            resp.Status = runtimehooksv1.ResponseStatusFailure
            resp.Message = fmt.Sprintf("Addon %s failed: %v", addon.Name, err)
            return
        }
    }
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}
```

### 需求 4: 其它应用拓扑化安装
**可行性**: ✅ 通过 Runtime Extension + 自定义 Controller

**技术方案**:
```yaml
# ClusterClass 变量定义应用及依赖
variables:
  - name: applications
    schema:
      openAPIV3Schema:
        type: array
        items:
          type: object
          properties:
            name: { type: string }
            type: { type: string, enum: [helm, manifest, operator] }
            chart:
              type: object
              properties:
                repo: { type: string }
                name: { type: string }
                version: { type: string }
            manifest:
              type: object
              properties:
                url: { type: string }
            dependsOn:
              type: array
              items: { type: string }
            namespace: { type: string }
            values:
              type: object
              x-kubernetes-preserve-unknown-fields: true
```

**拓扑安装 Controller**:
```go
type ApplicationReconciler struct {
    client.Client
}

func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    cluster := &clusterv1.Cluster{}
    r.Get(ctx, req.NamespacedName, cluster)
    
    apps := r.parseApplications(cluster)
    
    // 1. 构建依赖图 (DAG)
    graph := r.buildDAG(apps)
    
    // 2. 检测循环依赖
    if !graph.IsDAG() {
        return ctrl.Result{}, fmt.Errorf("circular dependency detected")
    }
    
    // 3. 拓扑排序
    order := graph.TopologicalSort()
    
    // 4. 按层级并行安装
    for _, level := range order.Levels() {
        var wg sync.WaitGroup
        errChan := make(chan error, len(level))
        
        for _, app := range level {
            if !r.allDepsReady(app.DependsOn) {
                return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
            }
            
            wg.Add(1)
            go func(a Application) {
                defer wg.Done()
                if err := r.installApp(ctx, cluster, a); err != nil {
                    errChan <- err
                }
            }(app)
        }
        
        wg.Wait()
        select {
        case err := <-errChan:
            return ctrl.Result{}, err
        default:
        }
    }
    
    return ctrl.Result{}, nil
}
```

**依赖关系示例**:
```yaml
applications:
  - name: calico
    type: manifest
    dependsOn: []
    
  - name: coredns
    type: helm
    dependsOn: [calico]
    
  - name: metrics-server
    type: helm
    dependsOn: [calico]
    
  - name: ingress-nginx
    type: helm
    dependsOn: [calico, coredns]
    
  - name: cert-manager
    type: helm
    dependsOn: [calico, coredns]
    
  - name: my-app
    type: helm
    dependsOn: [ingress-nginx, cert-manager]
```

### 需求 5: 证书体系自动轮转
**可行性**: ✅ 多层方案

**方案 A: Kubeadm 内置证书管理**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    certificateValidityPeriodDays: 365      # 非CA证书有效期
    caCertificateValidityPeriodDays: 3650   # CA证书有效期 (10年)
```

**KCP 自动轮转触发**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  rolloutBefore:
    certificatesExpiryDays: 30  # 证书到期前30天自动触发滚动更新
```

**方案 B: cert-manager (推荐用于完整证书生命周期)**
```yaml
# 通过 Addon Manager 安装 cert-manager
applications:
  - name: cert-manager
    type: helm
    chart:
      repo: https://charts.jetstack.io
      name: cert-manager
      version: v1.14.0
    values:
      installCRDs: true

# ClusterIssuer 使用集群CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-ca
spec:
  ca:
    secretName: cluster-ca  # CAPI生成的CA Secret
```

**方案 C: 自定义证书轮转控制器**
```go
type CertificateRotator struct {
    client.Client
}

func (r *CertificateRotator) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    kcp := &controlplanev1.KubeadmControlPlane{}
    r.Get(ctx, req.NamespacedName, kcp)
    
    workloadClient := r.getWorkloadClient(kcp)
    
    // 检查关键证书
    certs := []string{
        "apiserver",
        "apiserver-kubelet-client",
        "front-proxy-client",
        "apiserver-etcd-client",
    }
    
    needsRotation := false
    for _, certName := range certs {
        secret := &corev1.Secret{}
        workloadClient.Get(ctx, types.NamespacedName{
            Namespace: "kube-system",
            Name:      fmt.Sprintf("%s-%s", kcp.Name, certName),
        }, secret)
        
        cert, _ := tls.X509KeyPair(secret.Data["tls.crt"], secret.Data["tls.key"])
        expiry := cert.Leaf.NotAfter
        
        if expiry.Sub(time.Now()) < 30*24*time.Hour {
            needsRotation = true
            break
        }
    }
    
    if needsRotation {
        // 触发 KCP 滚动更新（会自动重新生成证书）
        if kcp.Annotations == nil {
            kcp.Annotations = make(map[string]string)
        }
        kcp.Annotations["controlplane.cluster.x-k8s.io/force-cert-rotation"] = time.Now().Format(time.RFC3339)
        r.Update(ctx, kcp)
    }
    
    return ctrl.Result{RequeueAfter: 24 * time.Hour}, nil
}
```

### 需求 6: 升级方案
**可行性**: ✅ CAPI 原生支持 + 扩展

#### 6.1 版本组件清单
```yaml
# VersionManifest CRD - 定义每个版本的组件清单
apiVersion: upgrade.cluster.x-k8s.io/v1alpha1
kind: VersionManifest
metadata:
  name: v1.31.0
spec:
  kubernetesVersion: "v1.31.0"
  components:
    # 控制面组件
    - name: kube-apiserver
      type: static-pod
      image: "{{ registry }}/kube-apiserver:v1.31.0"
      
    - name: kube-controller-manager
      type: static-pod
      image: "{{ registry }}/kube-controller-manager:v1.31.0"
      
    - name: kube-scheduler
      type: static-pod
      image: "{{ registry }}/kube-scheduler:v1.31.0"
      
    - name: etcd
      type: static-pod
      image: "{{ registry }}/etcd:3.5.15-0"
      
    # 核心插件
    - name: coredns
      type: deployment
      namespace: kube-system
      image: "{{ registry }}/coredns:1.11.1"
      
    - name: kube-proxy
      type: daemonset
      namespace: kube-system
      image: "{{ registry }}/kube-proxy:v1.31.0"
      
    - name: calico
      type: operator
      version: "v3.28.0"
      
    # 其他应用
    - name: metrics-server
      type: deployment
      namespace: kube-system
      image: "{{ registry }}/metrics-server:v0.7.0"
```

#### 6.2 升级流程

```yaml
# 升级操作 - 修改 Cluster 版本触发
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: my-cluster
spec:
  topology:
    version: "v1.31.0"  # 从 v1.30.0 升级到 v1.31.0
```

**CAPI 升级流程** (内置):

```
1. BeforeClusterUpgrade Hook (可选阻塞)
   ↓
2. 升级 Control Plane (KCP)
   ├── 更新 kubeadm-config ConfigMap
   ├── 逐个升级 control plane 节点 (滚动更新)
   │   ├── 创建新 Machine (新版本)
   │   ├── 等待新 Machine Ready
   │   ├── 删除旧 Machine
   │   └── 等待 etcd 稳定
   └── 验证 Control Plane 健康
   ↓
3. AfterControlPlaneUpgrade Hook
   ↓
4. 升级 Workers (MachineDeployments)
   ├── BeforeWorkersUpgrade Hook
   ├── 逐个 MachineDeployment 滚动升级
   └── AfterWorkersUpgrade Hook
   ↓
5. AfterClusterUpgrade Hook
   ├── 升级 Add-ons (通过 Addon Manager)
   └── 验证集群整体健康
```

#### 6.3 回滚方案

**方案 A: CAPI 内置回滚 (版本回退)**

```yaml
# 直接修改版本回退
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
spec:
  topology:
    version: "v1.30.0"  # 回退到旧版本
```

**方案 B: 升级前快照 + 恢复**

```go
type UpgradeOrchestrator struct {
    client.Client
}

func (r *UpgradeOrchestrator) PreUpgradeSnapshot(ctx context.Context, cluster *clusterv1.Cluster) error {
    // 1. 备份 etcd
    etcdClient := r.getEtcdClient(cluster)
    snapshot, _ := etcdClient.Snapshot(ctx)
    r.saveSnapshot(snapshot, cluster.Name)
    
    // 2. 备份关键 CRD 资源
    resources := []string{
        "clusters.cluster.x-k8s.io",
        "machines.cluster.x-k8s.io",
        "kubeadmcontrolplanes.controlplane.cluster.x-k8s.io",
        "machinedeployments.cluster.x-k8s.io",
    }
    for _, res := range resources {
        r.backupResources(ctx, res, cluster.Namespace)
    }
    
    // 3. 记录当前版本状态
    r.recordState(cluster)
    
    return nil
}

func (r *UpgradeOrchestrator) Rollback(ctx context.Context, cluster *clusterv1.Cluster) error {
    // 1. 恢复 etcd 快照
    snapshot := r.getLatestSnapshot(cluster.Name)
    r.restoreEtcdSnapshot(ctx, snapshot)
    
    // 2. 恢复 CRD 资源
    r.restoreResources(ctx, cluster.Namespace)
    
    // 3. 触发 CAPI 回滚
    // 修改版本回退
    return nil
}
```

**方案 C: 使用 CAPI Lifecycle Hooks 实现升级门禁**

```yaml
# 升级前验证 Hook
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: upgrade-validator
spec:
  clientConfig:
    service:
      name: upgrade-validator-service
      namespace: capi-system
  namespaceSelector: {}
```

```go
func (v *UpgradeValidator) BeforeClusterUpgrade(
    ctx context.Context,
    req *runtimehooksv1.BeforeClusterUpgradeRequest,
    resp *runtimehooksv1.BeforeClusterUpgradeResponse,
) {
    // 1. 验证版本兼容性
    if !v.isCompatibleUpgrade(req.FromKubernetesVersion, req.ToKubernetesVersion) {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = "Incompatible upgrade path"
        resp.RetryAfterSeconds = 0  // 不重试，直接失败
        return
    }
    
    // 2. 执行升级前检查
    checks := []func() error{
        v.checkEtcdHealth,
        v.checkNodeReady,
        v.checkStorageAvailable,
        v.checkBackupCompleted,
    }
    
    for _, check := range checks {
        if err := check(); err != nil {
            resp.Status = runtimehooksv1.ResponseStatusFailure
            resp.Message = fmt.Sprintf("Pre-upgrade check failed: %v", err)
            resp.RetryAfterSeconds = 60
            return
        }
    }
    
    // 3. 创建升级前快照
    v.createPreUpgradeSnapshot(req.Cluster)
    
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}
```

### 需求 7: 大规模集群控制面组件分离部署

**这是最具挑战性的需求**。

#### kubeadm 架构限制深度分析

```
kubeadm 设计的控制面架构:
┌─────────────────────────────────────────────┐
│              Control Plane Node              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │  API     │ │Scheduler │ │  Controller  │ │
│  │ Server   │ │          │ │   Manager    │ │
│  │(static   │ │(static   │ │  (static     │ │
│  │  pod)    │ │  pod)    │ │   pod)       │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│  ┌──────────┐                                │
│  │  etcd    │  (stacked 模式)                │
│  │(static   │                                │
│  │  pod)    │                                │
│  └──────────┘                                │
│                                              │
│  所有组件绑定在同一节点，无法分离                │
└─────────────────────────────────────────────┘
```

**kubeadm 核心限制**:
1. `kubeadm init` 和 `kubeadm join --control-plane` 会在节点上部署 **所有** 控制面组件
2. 没有配置项可以禁止某个节点运行特定组件
3. static pod manifest 由 kubeadm 自动生成和管理

#### 解决方案对比

| 方案 | 复杂度 | 可维护性 | 与CAPI集成 | 推荐度 |
|------|--------|----------|------------|--------|
| A. 接受同节点部署 | 低 | 高 | 原生 | ⭐⭐⭐⭐⭐ |
| B. 自定义 ControlPlane Provider | 高 | 中 | 需开发 | ⭐⭐⭐ |
| C. 混合模式 (kubeadm + 手动) | 中 | 低 | 部分 | ⭐⭐ |
| D. 修改 static pod manifests | 中 | 低 | 脆弱 | ⭐ |

**方案 A: 接受同节点部署 (强烈推荐)**

对于大多数场景，3-9 个控制面节点，每个节点运行全部组件是完全可行的：

```yaml
# 9节点大规模控制面
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 9  # 9个控制面节点
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          # 限制 API Server 资源使用
          - name: max-requests-inflight
            value: "2000"
          - name: max-mutating-requests-inflight
            value: "1000"
      scheduler:
        extraArgs:
          # Scheduler 资源占用极小，无需特殊配置
          - name: leader-elect
            value: "true"
      controllerManager:
        extraArgs:
          - name: leader-elect
            value: "true"
          - name: concurrent-gc-syncs
            value: "30"
          - name: node-cidr-mask-size
            value: "24"
```

**为什么这是最佳方案**:
- API Server 是无状态的，通过 LB 水平扩展
- Scheduler 和 Controller Manager 通过 leader election 保证只有一个 active
- 实际资源消耗: Scheduler ~50MB, Controller Manager ~100MB，影响极小
- 运维复杂度最低，升级/回滚路径清晰

**方案 B: 自定义 ControlPlane Provider (仅在极端需求下)**

如果确实需要物理分离，需要开发自定义 ControlPlane Provider:

```go
// SeparatedControlPlane CRD
type SeparatedControlPlaneSpec struct {
    APIServer     ComponentSpec `json:"apiServer"`
    Scheduler     ComponentSpec `json:"scheduler"`
    ControllerMgr ComponentSpec `json:"controllerManager"`
    Etcd          EtcdSpec      `json:"etcd"`
}

type ComponentSpec struct {
    Replicas         int32              `json:"replicas"`
    MachineTemplate  MachineTemplateRef `json:"machineTemplate"`
    Image            string             `json:"image"`
    ExtraArgs        []Arg              `json:"extraArgs,omitempty"`
}
```

**实现要点**:
- 不使用 kubeadm 管理控制面组件
- 自行管理证书、配置、健康检查
- 需要实现完整的升级/回滚逻辑
- 工作量巨大，建议仅在 kubeadm 完全无法满足需求时考虑

## 三、开源组件分析

| 组件 | 作用 | 版本 | 限制 |
|------|------|------|------|
| **Cluster API (CAPI)** | 核心框架，提供 Cluster/Machine/MD 等 CRD 及控制器 | v1beta2 | 不直接管理裸金属，需要 Infrastructure Provider |
| **Kubeadm Bootstrap Provider** | 生成 kubeadm 配置，执行 init/join | v1beta2 | 控制面组件必须同节点；不支持组件分离 |
| **Kubeadm Control Plane (KCP)** | 管理控制面生命周期、升级、扩缩容 | v1beta2 | 仅支持 stacked etcd 或 external etcd；不支持分离部署 |
| **CAPBM (自研)** | 裸金属 Infrastructure Provider | 自研 | 需要自行实现 SSH/电源管理/环境检查 |
| **Runtime SDK** | 扩展机制，提供 Lifecycle Hooks + Topology Mutation | v1alpha1 | experimental 阶段；需要自行实现 Extension Server |
| **cert-manager** | 证书自动轮转 | v1.14+ | 需要手动集成 CAPI 生成的 CA |
| **cluster-api-provider-ibmcloud/vsphere/aws** | 参考实现 | 各版本 | 可参考其 Infrastructure Provider 实现模式 |
| **Tinkerbell / Metal3** | 现有裸金属 CAPI Provider | Metal3 v1beta1 | Metal3 需要 PXE/Baremetal Operator，不适用于已有OS场景 |
| **coredns/corefile-migration** | CoreDNS 配置迁移 | 内置于 KCP | 仅管理版本升级时的 corefile 迁移 |
| **kubeadm** | K8s 引导工具 | v1.28+ | 控制面组件同节点绑定；升级路径固定 |

### 关键限制总结

| 限制项 | 影响 | 缓解方案 |
|--------|------|----------|
| kubeadm 不支持控制面组件分离 | 需求 7 无法完全满足 | 接受同节点部署，或开发自定义 Provider |
| kubeadm 不管理 external etcd 生命周期 | etcd 独立节点需自行管理 | Runtime Extension + MachineDeployment |
| Runtime SDK 仍为 experimental | API 可能变化 | 关注 CAPI 社区进展，做好版本锁定 |
| CAPI 不处理包安装 (kubeadm/containerd) | 需在 bootstrap 阶段处理 | preKubeadmCommands 中完成 |
| Metal3 不适用于已有 OS 场景 | 不能直接使用现有裸金属 Provider | 必须自研 CAPBM |

## 四、推荐实施路径

### Phase 1: 基础设施 (4-6 周)
- [ ] 开发 CAPBM Infrastructure Provider (BareMetalMachine, BareMetalCluster)
- [ ] 实现 SSH 连接、环境检查、ProviderID 管理
- [ ] 实现基本的 Machine 生命周期管理

### Phase 2: 集群模板 (2-3 周)
- [ ] 定义 ClusterClass (包含所有变量定义)
- [ ] 配置 KubeadmConfig (外部服务注入、kubelet/containerd 定制)
- [ ] 配置 KCP (etcd、LB、扩缩容)

### Phase 3: 扩展组件 (3-4 周)
- [ ] 开发 Addon Manager Runtime Extension
- [ ] 实现 Calico/CoreDNS/Kube-proxy 安装
- [ ] 实现应用拓扑化安装 (依赖图 + 拓扑排序)

### Phase 4: 升级与证书 (3-4 周)
- [ ] 实现 VersionManifest CRD 及升级编排
- [ ] 实现升级前快照与回滚机制
- [ ] 集成 cert-manager 实现证书自动轮转
- [ ] 配置 KCP 证书到期自动触发滚动更新

### Phase 5: 测试与优化 (持续)
- [ ] 端到端测试 (创建/升级/扩缩容/回滚)
- [ ] 大规模集群测试 (50+ 节点)
- [ ] 故障注入测试 (节点宕机/网络分区)

## 五、核心问答总结

| 需求 | kubeadm 是否支持 | 解决方案 |
|------|------------------|----------|
| etcd 外接配置 | ✅ 支持 | `clusterConfiguration.etcd.external` |
| etcd 标签指定节点 | ⚠️ 部分 | stacked: 原生; external: 需 Runtime Extension |
| API Server 标签指定节点 | ❌ 不支持 | 所有 control plane 节点都运行 API Server |
| API Server 扩缩容 | ✅ 支持 | 修改 KCP `spec.replicas` |
| API Server 指定 LB | ✅ 支持 | `controlPlaneEndpoint` |
| Scheduler 主备 | ✅ 支持 | leader election 默认启用 |
| Controller Manager 主备 | ✅ 支持 | leader election 默认启用 |
| 控制面组件分离部署 | ❌ 不支持 | 需自定义 Provider 或接受同节点 |
| kubelet 定制 | ✅ 支持 | `kubeletExtraArgs` + `kubeletConfig` |
| containerd 定制 | ✅ 支持 | `files` + `preKubeadmCommands` |
| 证书自动轮转 | ✅ 支持 | KCP `rolloutBefore.certificatesExpiryDays` |

# 基于 Cluster-API 的裸金属集群全生命周期管理方案(2)（完整版）

## 一、总体架构
```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Management Cluster                              │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────┐ │
│  │  CAPI Core   │  │  CAPBM       │  │  Runtime Extensions            │ │
│  │  (内置)      │  │  (自研)      │  │  ┌──────────┐ ┌──────────────┐ │ │
│  │              │  │              │  │  │ Addon    │ │ Upgrade      │ │ │
│  │ - Cluster    │  │ - InfraCluster│  │  │ Manager  │ │ Orchestrator │ │ │
│  │ - Machine    │  │ - InfraMachine│  │  └──────────┘ └──────────────┘ │ │
│  │ - MD/MS      │  │ - SSH Client  │  │  ┌──────────┐ ┌──────────────┐ │ │
│  │ - ClusterClass│ │ - IPMI/RF     │  │  │ Cert     │ │ Topology     │ │ │
│  │ - Topology   │  └──────────────┘  │  │ Rotator  │ │ Installer    │ │ │
│  └──────┬───────┘                    │  └──────────┘ └──────────────┘ │ │
│         │                            └────────────────────────────────┘ │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │                    Kubeadm Bootstrap + KCP                       │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │   │
│  │  │ KubeadmConfig│  │ KCP          │  │ In-Place Update       │  │   │
│  │  │ (bootstrap)  │  │ (controlplane)│  │ (experimental)        │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Workload Cluster (Bare Metal)                     │
│                                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐ │
│  │ API Server  │  │ Scheduler   │  │ Controller  │  │    etcd        │ │
│  │ Nodes (N)   │  │ Nodes (M)   │  │ Manager (M) │  │  Nodes (K)     │ │
│  │             │  │             │  │             │  │                │ │
│  │ HA: 3-9     │  │ Active/     │  │ Active/     │  │ External or    │ │
│  │             │  │ Standby     │  │ Standby     │  │ Stacked        │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───────┬────────┘ │
│         │                │                │                  │          │
│  ┌──────▼────────────────▼────────────────▼──────────────────▼───────┐  │
│  │                    External Load Balancer                          │  │
│  │              (用户提供: HAProxy/F5/Nginx)                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Worker Nodes (MachineDeployments)               │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────────────┐ │  │
│  │  │ Calico  │ │CoreDNS  │ │KubeProxy│ │ Custom Apps (topology)  │ │  │
│  │  │ (CNI)   │ │ (DNS)   │ │(Proxy)  │ │                         │ │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## 二、需求逐项分析与技术方案

### 需求 1: 机器列表管理
**可行性**: ✅ 完全可行

**技术选型**: 自研 `CAPBM` (Cluster API Provider Bare Metal)

**核心 CRD 设计**:
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalMachine
metadata:
  name: node-01
spec:
  hostName: "k8s-master-01"
  ipAddress: "192.168.1.101"
  sshPort: 22
  credentialsRef:
    name: node-01-credentials
    namespace: default
  powerManagement:
    type: "ipmi"                # ipmi | redfish | none
    address: "192.168.1.101:623"
    credentialsRef:
      name: node-01-bmc-credentials
```

### 需求 2: 外部服务集成
**可行性**: ✅ 完全可行

**技术方案**: ClusterClass Variables + KubeadmConfig preKubeadmCommands
```yaml
variables:
  - name: externalServices
    schema:
      openAPIV3Schema:
        type: object
        properties:
          nfs:
            type: object
            properties:
              server: { type: string }
              shares: { type: array, items: { type: string } }
          ntp:
            type: object
            properties:
              servers: { type: array, items: { type: string } }
          registry:
            type: object
            properties:
              endpoint: { type: string }
          packageMirror:
            type: object
            properties:
              type: { type: string, enum: [apt, yum] }
              url: { type: string }
          helmRegistry:
            type: object
            properties:
              endpoint: { type: string }
          externalLoadBalancer:
            type: object
            properties:
              endpoint: { type: string }
              port: { type: integer, default: 6443 }
```

### 需求 3: K8s 控制面配置

#### 3.1 etcd 支持外接配置
**可行性**: ✅ kubeadm 原生支持
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    etcd:
      external:
        endpoints:
          - "https://etcd-01:2379"
          - "https://etcd-02:2379"
          - "https://etcd-03:2379"
        caFile: "/etc/kubernetes/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
```

#### 3.2 etcd 通过标签指定节点并自动化安装
**kubeadm 支持情况**: ⚠️ 部分支持

| 模式 | 是否支持 | 说明 |
|------|----------|------|
| Stacked etcd | ✅ 原生支持 | kubeadm 在每个 control plane 节点自动部署 etcd static pod |
| External etcd + 独立节点池 | ⚠️ 需扩展 | kubeadm 不管理 external etcd 节点的生命周期 |

**推荐方案**:

**方案 A: Stacked etcd (中小规模推荐)**
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3
  kubeadmConfigSpec:
    clusterConfiguration:
      etcd:
        local:
          dataDir: /var/lib/etcd
```

**方案 B: External etcd + MachineDeployment + Runtime Extension (大规模)**
```yaml
# 独立的 etcd 节点池
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  name: etcd-nodes
  labels:
    role: etcd
spec:
  replicas: 3
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
          kind: KubeadmConfig
          name: etcd-bootstrap-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: etcd-machine-template
```
etcd 自动化安装通过 Runtime Extension (`AfterControlPlaneInitialized` hook) 完成集群配置。

#### 3.3 API Server 节点指定、扩缩容、负载均衡器
**kubeadm 支持情况**: ❌ **kubeadm 不支持将 API Server 单独部署在特定节点**

**kubeadm 架构限制**:
- kubeadm 将 API Server、Scheduler、Controller Manager 作为 **static pods** 部署
- 所有加入 control plane 的节点 (`kubeadm join --control-plane`) **都会运行全部三个组件**
- **无法**通过 kubeadm 配置实现 "节点A只跑API Server，节点B只跑Scheduler"

**解决方案**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3  # 支持动态扩缩容
  kubeadmConfigSpec:
    clusterConfiguration:
      controlPlaneEndpoint: "{{ .externalServices.externalLoadBalancer.endpoint }}:{{ .externalServices.externalLoadBalancer.port }}"
      apiServer:
        certSANs:
          - "{{ .externalServices.externalLoadBalancer.endpoint }}"
        extraArgs:
          - name: advertise-address
            value: "${LOCAL_IP}"
```
**扩缩容**: 直接修改 `spec.replicas`，KCP controller 自动处理

#### 3.4 Scheduler 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    scheduler:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
```

#### 3.5 Controller Manager 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    controllerManager:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
```

#### 3.6 Kubelet 定制化配置
**可行性**: ✅ 完全支持
```yaml
kubeadmConfigSpec:
  initConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        - name: kube-reserved
          value: "cpu=1000m,memory=1Gi"
        - name: system-reserved
          value: "cpu=500m,memory=512Mi"
        - name: eviction-hard
          value: "memory.available<500Mi,nodefs.available<10%"
        - name: max-pods
          value: "250"
  kubeletConfig:
    maxPods: 250
    containerLogMaxSize: "100Mi"
    containerLogMaxFiles: 5
```

**节点差异化配置** (通过 ClusterClass Patch):
```yaml
patches:
  - name: kubelet-gpu-nodes
    definitions:
      - selector:
          apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
          kind: BareMetalMachineTemplate
          matchResources:
            machineDeploymentClass:
              names:
                - gpu-workers
        jsonPatches:
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/-
            value:
              name: "feature-gates"
              value: "DevicePlugins=true"
```

#### 3.7 Containerd 定制化配置
**可行性**: ✅ 通过 `files` + `preKubeadmCommands`
```yaml
kubeadmConfigSpec:
  files:
    - path: /etc/containerd/config.toml
      content: |
        version = 2
        [plugins]
          [plugins."io.containerd.grpc.v1.cri"]
            sandbox_image = "{{ .externalServices.registry.endpoint }}/pause:3.9"
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
              SystemdCgroup = true
      owner: "root:root"
      permissions: "0644"
  preKubeadmCommands:
    - systemctl daemon-reload && systemctl restart containerd
```

#### 3.8 Calico / CoreDNS / Kube-proxy 通过扩展机制安装
**可行性**: ✅ 完全可行

**方案 A: 禁用内置管理**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    dns:
      disabled: true
    proxy:
      disabled: true
```

**方案 B: Runtime Extension (推荐)**
```go
func (a *AddonManager) AfterControlPlaneInitialized(...) {
    addons := []AddonDefinition{
        {Name: "calico", Priority: 1, Manifest: calicoManifests},
        {Name: "kube-proxy", Priority: 2, Manifest: kubeProxyManifests},
        {Name: "coredns", Priority: 3, Manifest: corednsManifests},
    }
    // 按优先级顺序安装
}
```

### 需求 4: 其它应用拓扑化安装
**可行性**: ✅ 通过 Runtime Extension + 自定义 Controller
```yaml
variables:
  - name: applications
    schema:
      openAPIV3Schema:
        type: array
        items:
          type: object
          properties:
            name: { type: string }
            type: { type: string, enum: [helm, manifest, operator] }
            dependsOn:
              type: array
              items: { type: string }
```
**拓扑安装 Controller**:
1. 构建依赖图 (DAG)
2. 检测循环依赖
3. 拓扑排序
4. 按层级并行安装

### 需求 5: 证书体系自动轮转
**可行性**: ✅ 多层方案

**方案 A: Kubeadm 内置**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    certificateValidityPeriodDays: 365
    caCertificateValidityPeriodDays: 3650
```

**KCP 自动触发轮转**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  rolloutBefore:
    certificatesExpiryDays: 30  # 到期前30天自动滚动
```

**方案 B: cert-manager (推荐)**
```yaml
applications:
  - name: cert-manager
    type: helm
    chart:
      repo: https://charts.jetstack.io
      name: cert-manager
      version: v1.14.0
```

### 需求 6: 升级方案
**可行性**: ✅ CAPI 原生支持 + 扩展

#### 6.1 版本组件清单
```yaml
apiVersion: upgrade.cluster.x-k8s.io/v1alpha1
kind: VersionManifest
metadata:
  name: v1.31.0
spec:
  kubernetesVersion: "v1.31.0"
  components:
    - name: kube-apiserver
      type: static-pod
      image: "{{ registry }}/kube-apiserver:v1.31.0"
    - name: kube-controller-manager
      type: static-pod
      image: "{{ registry }}/kube-controller-manager:v1.31.0"
    - name: kube-scheduler
      type: static-pod
      image: "{{ registry }}/kube-scheduler:v1.31.0"
    - name: etcd
      type: static-pod
      image: "{{ registry }}/etcd:3.5.15-0"
    - name: coredns
      type: deployment
      namespace: kube-system
      image: "{{ registry }}/coredns:1.11.1"
    - name: kube-proxy
      type: daemonset
      namespace: kube-system
      image: "{{ registry }}/kube-proxy:v1.31.0"
    - name: calico
      type: operator
      version: "v3.28.0"
```

#### 6.2 升级流程
```
1. BeforeClusterUpgrade Hook (验证 + 快照)
   ↓
2. 升级 Control Plane (KCP 滚动更新)
   ↓
3. AfterControlPlaneUpgrade Hook
   ↓
4. 升级 Workers (MachineDeployments 滚动更新)
   ↓
5. AfterClusterUpgrade Hook (升级 Add-ons)
```

#### 6.3 回滚方案
**方案 A: CAPI 内置回滚** - 直接修改版本回退
**方案 B: 升级前快照** - 备份 etcd + CRD 资源
**方案 C: Lifecycle Hooks** - 升级前验证门禁

### 需求 7: 大规模集群控制面组件分离部署
**kubeadm 核心限制**:
1. `kubeadm init` 和 `kubeadm join --control-plane` 会在节点上部署 **所有** 控制面组件
2. 没有配置项可以禁止某个节点运行特定组件
3. static pod manifest 由 kubeadm 自动生成和管理

**解决方案对比**:

| 方案 | 复杂度 | 可维护性 | 推荐度 |
|------|--------|----------|--------|
| A. 接受同节点部署 | 低 | 高 | ⭐⭐⭐⭐⭐ |
| B. 自定义 ControlPlane Provider | 高 | 中 | ⭐⭐⭐ |
| C. 修改 static pod manifests | 中 | 低 | ⭐ |

**方案 A (强烈推荐)**: 9 节点控制面，每个节点运行全部组件
- API Server 无状态，通过 LB 水平扩展
- Scheduler/Controller Manager 通过 leader election 保证单 active
- 实际资源消耗: Scheduler ~50MB, Controller Manager ~100MB

### 需求 8: 开源组件分析
| 组件 | 作用 | 限制 |
|------|------|------|
| **Cluster API** | 核心框架 | 不直接管理裸金属 |
| **Kubeadm Bootstrap** | 生成 kubeadm 配置 | 控制面组件必须同节点 |
| **Kubeadm Control Plane** | 管理控制面生命周期 | 仅支持 stacked/external etcd |
| **CAPBM (自研)** | 裸金属 Infrastructure Provider | 需自行实现 SSH/电源管理 |
| **Runtime SDK** | 扩展机制 (Hooks + Patches) | experimental 阶段 |
| **cert-manager** | 证书自动轮转 | 需手动集成 CAPI CA |
| **Metal3** | 现有裸金属 Provider | 需要 PXE，不适用于已有 OS |
| **kubeadm** | K8s 引导工具 | 控制面组件同节点绑定 |

### 需求 9: 在 OS 上安装二进制组件的设计
**可行性**: ✅ 完全可行

**技术方案**: 通过 `KubeadmConfigSpec.preKubeadmCommands` + `Files` 实现

#### 9.1 整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                    二进制安装流程 (preKubeadmCommands)               │
│                                                                      │
│  1. 环境检查                                                         │
│     ├── OS 版本验证 (CentOS/Ubuntu 等)                               │
│     ├── 内核参数检查                                                 │
│     ├── 网络连通性检查                                               │
│     └── 磁盘空间检查                                                 │
│                                                                      │
│  2. 系统依赖安装                                                     │
│     ├── socat, conntrack, ipset, ebtables                            │
│     └── 内核模块加载 (br_netfilter, overlay)                         │
│                                                                      │
│  3. NTP 安装与配置                                                   │
│     ├── chrony 或 ntp 安装                                           │
│     └── 配置 NTP 服务器                                              │
│                                                                      │
│  4. Containerd 安装与配置                                            │
│     ├── 二进制下载/包安装                                            │
│     ├── runc 安装                                                    │
│     ├── CNI 插件安装                                                 │
│     └── 配置文件生成                                                 │
│                                                                      │
│  5. Kubernetes 二进制安装                                            │
│     ├── kubeadm 安装                                                 │
│     ├── kubelet 安装                                                 │
│     ├── kubectl 安装                                                 │
│     └── 版本一致性验证                                               │
│                                                                      │
│  6. 系统配置                                                         │
│     ├── 内核参数设置 (sysctl)                                        │
│     ├── 防火墙/SELinux 配置                                          │
│     ├── Swap 禁用                                                    │
│     └── 服务启用 (kubelet, containerd)                               │
│                                                                      │
│  7. 安装验证                                                         │
│     ├── 二进制版本检查                                               │
│     ├── 服务状态检查                                                 │
│     └── 网络连通性验证                                               │
└─────────────────────────────────────────────────────────────────────┘
```

#### 9.2 详细实现方案
**方案 A: 通过包管理器安装 (推荐 - 适用于有二进制安装源的场景)**
```yaml
kubeadmConfigSpec:
  preKubeadmCommands:
    - |
      #!/bin/bash
      set -euo pipefail
      
      echo "=== [1/7] 环境检查 ==="
      # OS 版本检查
      if [ -f /etc/os-release ]; then
        . /etc/os-release
        echo "OS: $NAME $VERSION_ID"
        case "$ID" in
          centos|rhel|almalinux|rocky)
            PKG_MGR="yum"
            ;;
          ubuntu|debian)
            PKG_MGR="apt"
            ;;
          *)
            echo "Unsupported OS: $ID"
            exit 1
            ;;
        esac
      else
        echo "Cannot detect OS"
        exit 1
      fi
      
      # 内核版本检查
      KERNEL_VERSION=$(uname -r | cut -d'-' -f1)
      REQUIRED_VERSION="3.10"
      if [ "$(printf '%s\n' "$REQUIRED_VERSION" "$KERNEL_VERSION" | sort -V | head -n1)" != "$REQUIRED_VERSION" ]; then
        echo "Kernel version $KERNEL_VERSION is too old, need >= $REQUIRED_VERSION"
        exit 1
      fi
      
      # 磁盘空间检查 (至少 20GB 可用)
      AVAILABLE_GB=$(df -BG / | awk 'NR==2 {print $4}' | tr -d 'G')
      if [ "$AVAILABLE_GB" -lt 20 ]; then
        echo "Insufficient disk space: ${AVAILABLE_GB}GB available, need 20GB"
        exit 1
      fi
      
      echo "=== [2/7] 系统依赖安装 ==="
      # 加载必需的内核模块
      modprobe br_netfilter 2>/dev/null || true
      modprobe overlay 2>/dev/null || true
      
      # 写入模块加载配置
      cat > /etc/modules-load.d/k8s.conf <<EOF
      br_netfilter
      overlay
      EOF
      
      # 配置内核参数
      cat > /etc/sysctl.d/k8s.conf <<EOF
      net.bridge.bridge-nf-call-iptables = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward = 1
      net.ipv4.conf.all.forwarding = 1
      vm.overcommit_memory = 1
      vm.panic_on_oom = 0
      fs.inotify.max_user_watches = 524288
      fs.file-max = 1048576
      EOF
      sysctl --system
      
      # 安装系统依赖包
      if [ "$PKG_MGR" = "yum" ]; then
        yum install -y socat conntrack-tools ipset ebtables
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get update
        apt-get install -y socat conntrack ipset ebtables
      fi
      
      echo "=== [3/7] NTP 安装与配置 ==="
      if [ "$PKG_MGR" = "yum" ]; then
        yum install -y chrony
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get install -y chrony
      fi
      
      cat > /etc/chrony.conf <<EOF
      {{ range .externalServices.ntp.servers }}
      server {{ . }} iburst
      {{ end }}
      makestep 1 3
      rtcsync
      EOF
      
      systemctl enable --now chronyd
      chronyc waitsync 30
      
      echo "=== [4/7] Containerd 安装与配置 ==="
      # 安装 containerd
      if [ "$PKG_MGR" = "yum" ]; then
        # 配置本地镜像源
        cat > /etc/yum.repos.d/containerd.repo <<EOF
        [containerd]
        name=containerd
        baseurl={{ .externalServices.packageMirror.url }}/containerd
        enabled=1
        gpgcheck=0
        EOF
        yum install -y containerd.io
      elif [ "$PKG_MGR" = "apt" ]; then
        # 配置 apt 源
        cat > /etc/apt/sources.list.d/containerd.list <<EOF
        deb {{ .externalServices.packageMirror.url }}/containerd ./
        EOF
        apt-get update
        apt-get install -y containerd.io
      fi
      
      # 安装 runc
      if [ "$PKG_MGR" = "yum" ]; then
        yum install -y runc
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get install -y runc
      fi
      
      # 安装 CNI 插件
      mkdir -p /opt/cni/bin
      curl -L {{ .externalServices.packageMirror.url }}/cni-plugins/cni-plugins-linux-amd64-v1.5.0.tgz | tar -C /opt/cni/bin -xz
      
      # 配置 containerd
      mkdir -p /etc/containerd
      containerd config default > /etc/containerd/config.toml
      
      # 修改 containerd 配置
      sed -i 's|sandbox_image = "registry.k8s.io/pause:.*"|sandbox_image = "{{ .externalServices.registry.endpoint }}/pause:3.9"|' /etc/containerd/config.toml
      sed -i 's|SystemdCgroup = false|SystemdCgroup = true|' /etc/containerd/config.toml
      
      # 配置镜像仓库
      cat >> /etc/containerd/config.toml <<EOF
      
      [plugins."io.containerd.grpc.v1.cri".registry]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
          [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
            endpoint = ["{{ .externalServices.registry.endpoint }}"]
      EOF
      
      systemctl daemon-reload
      systemctl enable --now containerd
      
      echo "=== [5/7] Kubernetes 二进制安装 ==="
      KUBE_VERSION="{{ .kubernetesVersion }}"  # 例如 v1.31.0
      
      if [ "$PKG_MGR" = "yum" ]; then
        cat > /etc/yum.repos.d/kubernetes.repo <<EOF
        [kubernetes]
        name=Kubernetes
        baseurl={{ .externalServices.packageMirror.url }}/kubernetes/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=0
        EOF
        
        yum install -y kubeadm-${KUBE_VERSION#v} kubelet-${KUBE_VERSION#v} kubectl-${KUBE_VERSION#v}
        
      elif [ "$PKG_MGR" = "apt" ]; then
        # 安装依赖
        apt-get install -y apt-transport-https ca-certificates curl gpg
        
        # 添加 Kubernetes GPG key
        curl -fsSL {{ .externalServices.packageMirror.url }}/kubernetes/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
        
        cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
        deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] {{ .externalServices.packageMirror.url }}/kubernetes/apt/ kubernetes-xenial main
        EOF
        
        apt-get update
        apt-get install -y kubeadm=${KUBE_VERSION#v}-* kubelet=${KUBE_VERSION#v}-* kubectl=${KUBE_VERSION#v}-*
        
        # 锁定版本防止自动升级
        apt-mark hold kubeadm kubelet kubectl
      fi
      
      echo "=== [6/7] 系统配置 ==="
      # 禁用 Swap
      swapoff -a
      sed -i '/swap/d' /etc/fstab
      
      # 禁用防火墙 (或配置放行规则)
      systemctl disable --now firewalld 2>/dev/null || true
      systemctl disable --now ufw 2>/dev/null || true
      
      # 禁用 SELinux
      setenforce 0 2>/dev/null || true
      sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config 2>/dev/null || true
      
      # 配置 kubelet 额外参数
      cat > /etc/sysconfig/kubelet <<EOF
      KUBELET_EXTRA_ARGS=--node-ip=$(hostname -i | awk '{print $1}')
      EOF
      
      # 启用 kubelet (但不启动，由 kubeadm 控制)
      systemctl enable kubelet
      
      echo "=== [7/7] 安装验证 ==="
      # 验证二进制版本
      KUBEADM_VERSION=$(kubeadm version -o short)
      KUBELET_VERSION=$(kubelet --version | awk '{print $2}')
      KUBECTL_VERSION=$(kubectl version --client -o short 2>/dev/null || kubectl version --client --short 2>/dev/null | awk '{print $3}')
      CONTAINERD_VERSION=$(containerd --version | awk '{print $3}')
      
      echo "kubeadm: $KUBEADM_VERSION"
      echo "kubelet: $KUBELET_VERSION"
      echo "kubectl: $KUBECTL_VERSION"
      echo "containerd: $CONTAINERD_VERSION"
      
      # 版本一致性检查
      if [ "$KUBEADM_VERSION" != "$KUBELET_VERSION" ]; then
        echo "ERROR: kubeadm version ($KUBEADM_VERSION) != kubelet version ($KUBELET_VERSION)"
        exit 1
      fi
      
      # 服务状态检查
      systemctl is-active --quiet containerd || { echo "ERROR: containerd not running"; exit 1; }
      systemctl is-active --quiet chronyd || { echo "ERROR: chronyd not running"; exit 1; }
      
      # 网络连通性检查
      ping -c 1 {{ .externalServices.registry.endpoint }} || { echo "WARNING: Cannot reach registry"; }
      
      echo "=== 所有前置检查通过 ==="
```

**方案 B: 通过二进制文件直接安装 (适用于离线环境)**
```yaml
kubeadmConfigSpec:
  files:
    # 预下载的二进制文件通过 Secret 注入
    - path: /tmp/binaries/kubeadm
      contentFrom:
        secret:
          name: k8s-binaries
          key: kubeadm
      owner: "root:root"
      permissions: "0755"
    - path: /tmp/binaries/kubelet
      contentFrom:
        secret:
          name: k8s-binaries
          key: kubelet
      owner: "root:root"
      permissions: "0755"
    - path: /tmp/binaries/kubectl
      contentFrom:
        secret:
          name: k8s-binaries
          key: kubectl
      owner: "root:root"
      permissions: "0755"
    - path: /tmp/binaries/containerd
      contentFrom:
        secret:
          name: containerd-binaries
          key: containerd
      owner: "root:root"
      permissions: "0755"
    - path: /tmp/binaries/runc
      contentFrom:
        secret:
          name: containerd-binaries
          key: runc
      owner: "root:root"
      permissions: "0755"
      
  preKubeadmCommands:
    - |
      #!/bin/bash
      set -euo pipefail
      
      # 安装二进制文件
      cp /tmp/binaries/kubeadm /usr/local/bin/kubeadm
      cp /tmp/binaries/kubelet /usr/local/bin/kubelet
      cp /tmp/binaries/kubectl /usr/local/bin/kubectl
      
      # 安装 containerd
      mkdir -p /usr/local/bin
      cp /tmp/binaries/containerd* /usr/local/bin/
      cp /tmp/binaries/runc /usr/local/bin/runc
      chmod +x /usr/local/bin/containerd* /usr/local/bin/runc
      
      # 创建 systemd 服务文件
      cat > /etc/systemd/system/containerd.service <<EOF
      [Unit]
      Description=containerd container runtime
      Documentation=https://containerd.io
      After=network.target local-fs.target
      
      [Service]
      ExecStartPre=-/sbin/modprobe overlay
      ExecStart=/usr/local/bin/containerd
      Type=notify
      Delegate=yes
      KillMode=process
      Restart=always
      RestartSec=5
      LimitNOFILE=1048576
      LimitNPROC=infinity
      LimitCORE=infinity
      
      [Install]
      WantedBy=multi-user.target
      EOF
      
      # 创建 kubelet systemd 服务文件
      cat > /etc/systemd/system/kubelet.service <<EOF
      [Unit]
      Description=kubelet: The Kubernetes Node Agent
      Documentation=https://kubernetes.io/docs/home/
      Wants=network-online.target
      After=network-online.target containerd.service
      
      [Service]
      ExecStart=/usr/local/bin/kubelet
      Restart=always
      StartLimitIntervalSec=0
      RestartSec=10
      
      [Install]
      WantedBy=multi-user.target
      EOF
      
      cat > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf <<EOF
      [Service]
      Environment="KUBELET_EXTRA_ARGS=--node-ip=$(hostname -i | awk '{print $1}')"
      EOF
      
      systemctl daemon-reload
      systemctl enable --now containerd
      systemctl enable kubelet
```

**方案 C: 通过 CAPBM 预安装 (推荐用于大规模场景)**

在 CAPBM 的 `BareMetalMachine` controller 中实现预安装逻辑:
```go
// BareMetalMachineReconciler
func (r *BareMetalMachineReconciler) reconcile(ctx context.Context, bmm *infrastructurev1beta2.BareMetalMachine) (ctrl.Result, error) {
    // 1. SSH 连接到目标机器
    client, err := r.sshClient.Connect(bmm.Spec.IPAddress, bmm.Spec.SSHPort, bmm.Spec.CredentialsRef)
    
    // 2. 检查是否已安装必需的二进制
    installed, err := r.checkBinariesInstalled(client)
    if !installed {
        // 3. 执行预安装脚本
        if err := r.installBinaries(client, bmm); err != nil {
            r.Recorder.Event(bmm, "Warning", "BinaryInstallFailed", err.Error())
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
    }
    
    // 4. 标记机器就绪
    bmm.Status.Ready = true
    bmm.Status.ProviderID = fmt.Sprintf("baremetal://%s", bmm.Spec.HostName)
    
    return ctrl.Result{}, nil
}
```

#### 9.3 版本管理设计
```yaml
# BinaryVersion CRD - 管理二进制版本
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BinaryVersion
metadata:
  name: k8s-v1.31.0
spec:
  kubernetesVersion: "v1.31.0"
  binaries:
    kubeadm:
      url: "{{ .packageMirror.url }}/kubernetes/bin/v1.31.0/kubeadm"
      checksum: "sha256:abc123..."
    kubelet:
      url: "{{ .packageMirror.url }}/kubernetes/bin/v1.31.0/kubelet"
      checksum: "sha256:def456..."
    kubectl:
      url: "{{ .packageMirror.url }}/kubernetes/bin/v1.31.0/kubectl"
      checksum: "sha256:ghi789..."
    containerd:
      version: "1.7.20"
      url: "{{ .packageMirror.url }}/containerd/containerd-1.7.20-linux-amd64.tar.gz"
      checksum: "sha256:jkl012..."
    runc:
      version: "1.1.13"
      url: "{{ .packageMirror.url }}/containerd/runc.amd64"
      checksum: "sha256:mno345..."
    cniPlugins:
      version: "1.5.0"
      url: "{{ .packageMirror.url }}/cni-plugins/cni-plugins-linux-amd64-v1.5.0.tgz"
      checksum: "sha256:pqr678..."
```

#### 9.4 安装验证机制
```yaml
kubeadmConfigSpec:
  preKubeadmCommands:
    - |
      # 安装后验证函数
      verify_installation() {
        local errors=0
        
        # 检查二进制是否存在且可执行
        for bin in kubeadm kubelet kubectl containerd runc; do
          if ! command -v $bin &>/dev/null; then
            echo "ERROR: $bin not found in PATH"
            errors=$((errors + 1))
          fi
        done
        
        # 检查版本一致性
        KUBEADM_VER=$(kubeadm version -o short)
        KUBELET_VER=$(kubelet --version | awk '{print $2}')
        if [ "$KUBEADM_VER" != "$KUBELET_VER" ]; then
          echo "ERROR: kubeadm ($KUBEADM_VER) != kubelet ($KUBELET_VER)"
          errors=$((errors + 1))
        fi
        
        # 检查服务状态
        for svc in containerd kubelet; do
          if ! systemctl is-active --quiet $svc; then
            echo "ERROR: $svc service is not active"
            errors=$((errors + 1))
          fi
        done
        
        # 检查内核参数
        for param in "net.bridge.bridge-nf-call-iptables=1" "net.ipv4.ip_forward=1"; do
          key=$(echo $param | cut -d= -f1)
          val=$(echo $param | cut -d= -f2)
          current=$(sysctl -n $key 2>/dev/null || echo "0")
          if [ "$current" != "$val" ]; then
            echo "ERROR: sysctl $key = $current, expected $val"
            errors=$((errors + 1))
          fi
        done
        
        # 检查 Swap 是否禁用
        if swapon --show | grep -q .; then
          echo "ERROR: Swap is still enabled"
          errors=$((errors + 1))
        fi
        
        # 检查 NTP 同步
        if ! chronyc tracking | grep -q "Leap status.*Normal"; then
          echo "WARNING: NTP not synchronized"
        fi
        
        if [ $errors -gt 0 ]; then
          echo "=== Installation verification FAILED with $errors errors ==="
          exit 1
        fi
        
        echo "=== Installation verification PASSED ==="
      }
      
      verify_installation
```

#### 9.5 方案对比
| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **A. 包管理器安装** | 有镜像源/包管理源 | 标准化、易维护、支持自动依赖 | 需要配置源 |
| **B. 二进制直接安装** | 离线环境 | 不依赖包管理器 | 需要手动管理服务文件 |
| **C. CAPBM 预安装** | 大规模集群 | 与 CAPI 深度集成、可复用 | 开发工作量大 |

**推荐**: 
- 中小规模: **方案 A** (包管理器)
- 离线环境: **方案 B** (二进制)
- 大规模 (>100节点): **方案 C** (CAPBM 预安装)

## 三、核心问答总结
| 需求 | kubeadm 是否支持 | 解决方案 |
|------|------------------|----------|
| etcd 外接配置 | ✅ 支持 | `clusterConfiguration.etcd.external` |
| etcd 标签指定节点 | ⚠️ 部分 | stacked: 原生; external: 需 Runtime Extension |
| API Server 标签指定节点 | ❌ 不支持 | 所有 control plane 节点都运行 API Server |
| API Server 扩缩容 | ✅ 支持 | 修改 KCP `spec.replicas` |
| API Server 指定 LB | ✅ 支持 | `controlPlaneEndpoint` |
| Scheduler 主备 | ✅ 支持 | leader election 默认启用 |
| Controller Manager 主备 | ✅ 支持 | leader election 默认启用 |
| 控制面组件分离部署 | ❌ 不支持 | 需自定义 Provider 或接受同节点 |
| kubelet 定制 | ✅ 支持 | `kubeletExtraArgs` + `kubeletConfig` |
| containerd 定制 | ✅ 支持 | `files` + `preKubeadmCommands` |
| 证书自动轮转 | ✅ 支持 | KCP `rolloutBefore.certificatesExpiryDays` |
| 二进制安装 | ✅ 支持 | `preKubeadmCommands` + `Files` |

# 基于 Cluster-API 的裸金属集群全生命周期管理方案(3)

## 一、总体架构
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Management Cluster                                │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────────┐ │
│  │  CAPI Core   │  │  CAPBM       │  │  Runtime Extensions                │ │
│  │  (内置)      │  │  (自研)      │  │  ┌──────────┐ ┌──────────────────┐ │ │
│  │              │  │              │  │  │ Addon    │ │ Upgrade          │ │ │
│  │ - Cluster    │  │ - InfraCluster│  │  │ Manager  │ │ Orchestrator     │ │ │
│  │ - Machine    │  │ - InfraMachine│  │  └──────────┘ └──────────────────┘ │ │
│  │ - MD/MS      │  │ - SSH Client  │  │  ┌──────────┐ ┌──────────────────┐ │ │
│  │ - ClusterClass│ │ - IPMI/Redfish│  │  │ Cert     │ │ In-Place Update  │ │ │
│  │ - Topology   │  └──────────────┘  │  │ Rotator  │ │ Extension        │ │ │
│  └──────┬───────┘                    │  └──────────┘ └──────────────────┘ │ │
│         │                            └────────────────────────────────────┘ │
│  ┌──────▼──────────────────────────────────────────────────────────────┐   │
│  │                    Kubeadm Bootstrap + KCP                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │   │
│  │  │ KubeadmConfig│  │ KCP          │  │ In-Place Update           │  │   │
│  │  │ (bootstrap)  │  │ (controlplane)│  │ (Alpha feature gate)      │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Workload Cluster (Bare Metal)                       │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────────────┐  │
│  │ API Server  │  │ Scheduler   │  │ Controller  │  │    etcd            │  │
│  │ Nodes (N)   │  │ Nodes (M)   │  │ Manager (M) │  │  Nodes (K)         │  │
│  │             │  │             │  │             │  │                    │  │
│  │ HA: 3-9     │  │ Active/     │  │ Active/     │  │ External or        │  │
│  │             │  │ Standby     │  │ Standby     │  │ Stacked            │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────────┬───────────┘  │
│         │                │                │                   │              │
│  ┌──────▼────────────────▼────────────────▼───────────────────▼──────────┐  │
│  │                    External Load Balancer                              │  │
│  │              (用户提供: HAProxy/F5/Nginx)                              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Worker Nodes (MachineDeployments)                   │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────────────────┐ │  │
│  │  │ Calico  │ │CoreDNS  │ │KubeProxy│ │ Custom Apps (topology)      │ │  │
│  │  │ (CNI)   │ │ (DNS)   │ │(Proxy)  │ │                             │ │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 二、需求逐项分析与技术方案

### 需求 1: 机器列表管理
**可行性**: ✅ 完全可行

**技术选型**: 自研 `CAPBM` (Cluster API Provider Bare Metal)
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: BareMetalMachine
metadata:
  name: node-01
spec:
  hostName: "k8s-master-01"
  ipAddress: "192.168.1.101"
  sshPort: 22
  credentialsRef:
    name: node-01-credentials   # Secret 包含 username/password
    namespace: default
  powerManagement:
    type: "ipmi"                # ipmi | redfish | none
    address: "192.168.1.101:623"
    credentialsRef:
      name: node-01-bmc-credentials
```
**关键实现**:
- 通过 SSH 连接已安装 OS 的裸金属机器
- `ProviderID` 格式: `baremetal://<hostname>`
- Credentials 存储为 Kubernetes Secret

### 需求 2: 外部服务集成
**可行性**: ✅ 完全可行

**技术方案**: ClusterClass Variables + KubeadmConfig preKubeadmCommands
```yaml
variables:
  - name: externalServices
    schema:
      openAPIV3Schema:
        type: object
        properties:
          nfs:
            type: object
            properties:
              server: { type: string }
              shares: { type: array, items: { type: string } }
          ntp:
            type: object
            properties:
              servers: { type: array, items: { type: string } }
          registry:
            type: object
            properties:
              endpoint: { type: string }
          packageMirror:
            type: object
            properties:
              type: { type: string, enum: [apt, yum] }
              url: { type: string }
          helmRegistry:
            type: object
            properties:
              endpoint: { type: string }
          externalLoadBalancer:
            type: object
            properties:
              endpoint: { type: string }
              port: { type: integer, default: 6443 }
```

### 需求 3: K8s 控制面配置

#### 3.1 etcd 支持外接配置

**可行性**: ✅ kubeadm 原生支持
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    etcd:
      external:
        endpoints:
          - "https://etcd-01:2379"
          - "https://etcd-02:2379"
          - "https://etcd-03:2379"
        caFile: "/etc/kubernetes/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
```

#### 3.2 etcd 通过标签指定节点并自动化安装
**kubeadm 支持情况**: ⚠️ 部分支持

| 模式 | 是否支持 | 说明 |
|------|----------|------|
| Stacked etcd (与控制面同节点) | ✅ 原生支持 | kubeadm 在每个 control plane 节点自动部署 etcd static pod |
| External etcd + 独立节点池 | ⚠️ 需扩展 | kubeadm 不管理 external etcd 节点的生命周期 |

**推荐方案**:

**方案 A: Stacked etcd (中小规模)**
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3
  kubeadmConfigSpec:
    clusterConfiguration:
      etcd:
        local:
          dataDir: /var/lib/etcd
```

**方案 B: External etcd + MachineDeployment + Runtime Extension (大规模)**

通过 MachineDeployment 管理独立 etcd 节点池，使用 Runtime Extension (`AfterControlPlaneInitialized` hook) 完成 etcd 集群配置。

#### 3.3 API Server 节点指定、扩缩容、负载均衡器
**kubeadm 支持情况**: ❌ **kubeadm 不支持将 API Server 单独部署在特定节点**

**kubeadm 架构限制**:
- kubeadm 将 API Server、Scheduler、Controller Manager 作为 **static pods** 部署
- 所有加入 control plane 的节点 (`kubeadm join --control-plane`) **都会运行全部三个组件**
- **无法**通过 kubeadm 配置实现 "节点A只跑API Server，节点B只跑Scheduler"

**解决方案**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3  # 支持动态扩缩容
  kubeadmConfigSpec:
    clusterConfiguration:
      controlPlaneEndpoint: "{{ .externalServices.externalLoadBalancer.endpoint }}:{{ .externalServices.externalLoadBalancer.port }}"
      apiServer:
        certSANs:
          - "{{ .externalServices.externalLoadBalancer.endpoint }}"
        extraArgs:
          - name: advertise-address
            value: "${LOCAL_IP}"
```
**扩缩容**: 直接修改 `spec.replicas`，KCP controller 自动处理

#### 3.4 Scheduler 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    scheduler:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
```

#### 3.5 Controller Manager 主备模式
**kubeadm 支持情况**: ✅ **原生支持**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    controllerManager:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
```

#### 3.6 Kubelet 定制化配置
**可行性**: ✅ 完全支持
```yaml
kubeadmConfigSpec:
  initConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        - name: kube-reserved
          value: "cpu=1000m,memory=1Gi"
        - name: system-reserved
          value: "cpu=500m,memory=512Mi"
        - name: eviction-hard
          value: "memory.available<500Mi,nodefs.available<10%"
        - name: max-pods
          value: "250"
  kubeletConfig:
    maxPods: 250
    containerLogMaxSize: "100Mi"
    containerLogMaxFiles: 5
```

**节点差异化配置** (通过 ClusterClass Patch):
```yaml
patches:
  - name: kubelet-gpu-nodes
    definitions:
      - selector:
          apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
          kind: BareMetalMachineTemplate
          matchResources:
            machineDeploymentClass:
              names: [gpu-workers]
        jsonPatches:
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/-
            value:
              name: "feature-gates"
              value: "DevicePlugins=true"
```

#### 3.7 Containerd 定制化配置
**可行性**: ✅ 通过 `files` + `preKubeadmCommands`
```yaml
kubeadmConfigSpec:
  files:
    - path: /etc/containerd/config.toml
      content: |
        version = 2
        [plugins."io.containerd.grpc.v1.cri"]
          sandbox_image = "{{ .externalServices.registry.endpoint }}/pause:3.9"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
      owner: "root:root"
      permissions: "0644"
  preKubeadmCommands:
    - systemctl daemon-reload && systemctl restart containerd
```

#### 3.8 Calico / CoreDNS / Kube-proxy 通过扩展机制安装
**可行性**: ✅ 完全可行
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    dns:
      disabled: true    # 禁用内置 CoreDNS
    proxy:
      disabled: true    # 禁用内置 Kube-proxy
```
通过 Runtime Extension (`AfterControlPlaneInitialized` hook) 按顺序安装。

### 需求 4: 其它应用拓扑化安装
**可行性**: ✅ 通过 Runtime Extension + 自定义 Controller
```yaml
variables:
  - name: applications
    schema:
      openAPIV3Schema:
        type: array
        items:
          type: object
          properties:
            name: { type: string }
            type: { type: string, enum: [helm, manifest, operator] }
            dependsOn:
              type: array
              items: { type: string }
```
**拓扑安装流程**:
1. 构建依赖图 (DAG)
2. 检测循环依赖
3. 拓扑排序
4. 按层级并行安装

### 需求 5: 证书体系自动轮转
**可行性**: ✅ 多层方案

**方案 A: Kubeadm 内置**
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    certificateValidityPeriodDays: 365
    caCertificateValidityPeriodDays: 3650
```

**KCP 自动触发轮转**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  rolloutBefore:
    certificatesExpiryDays: 30  # 到期前30天自动滚动
```

**方案 B: cert-manager (推荐)**
```yaml
applications:
  - name: cert-manager
    type: helm
    chart:
      repo: https://charts.jetstack.io
      name: cert-manager
      version: v1.14.0
```

### 需求 6: 升级方案
**可行性**: ✅ CAPI 原生支持 + In-Place Update Extension

#### 6.1 版本组件清单
```yaml
apiVersion: upgrade.cluster.x-k8s.io/v1alpha1
kind: VersionManifest
metadata:
  name: v1.31.0
spec:
  kubernetesVersion: "v1.31.0"
  components:
    # 控制面组件
    - name: kube-apiserver
      type: static-pod
      image: "{{ registry }}/kube-apiserver:v1.31.0"
    - name: kube-controller-manager
      type: static-pod
      image: "{{ registry }}/kube-controller-manager:v1.31.0"
    - name: kube-scheduler
      type: static-pod
      image: "{{ registry }}/kube-scheduler:v1.31.0"
    - name: etcd
      type: static-pod
      image: "{{ registry }}/etcd:3.5.15-0"
    # 核心插件
    - name: coredns
      type: deployment
      namespace: kube-system
      image: "{{ registry }}/coredns:1.11.1"
    - name: kube-proxy
      type: daemonset
      namespace: kube-system
      image: "{{ registry }}/kube-proxy:v1.31.0"
    - name: calico
      type: operator
      version: "v3.28.0"
    # 二进制组件
    - name: kubeadm
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubeadm"
    - name: kubelet
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubelet"
    - name: kubectl
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubectl"
    - name: containerd
      type: binary
      version: "1.7.20"
      url: "{{ registry }}/containerd/containerd-1.7.20-linux-amd64.tar.gz"
    - name: runc
      type: binary
      version: "1.1.13"
      url: "{{ registry }}/containerd/runc.amd64"
    - name: cni-plugins
      type: binary
      version: "1.5.0"
      url: "{{ registry }}/cni-plugins/cni-plugins-linux-amd64-v1.5.0.tgz"
    - name: chrony
      type: binary
      version: "4.4"
      url: "{{ registry }}/ntp/chrony-4.4.tar.gz"
```

#### 6.2 升级流程
```
1. BeforeClusterUpgrade Hook (验证 + 快照)
   ↓
2. 升级 Control Plane (KCP 滚动更新 或 原地升级)
   ├── 2a. 原地升级路径 (In-Place Update Extension)
   │   ├── CanUpdateMachine Hook: 检查是否支持原地升级
   │   ├── UpdateMachine Hook: SSH 执行升级脚本
   │   │   ├── 下载新版本二进制
   │   │   ├── 备份当前版本
   │   │   ├── 替换二进制文件
   │   │   ├── 执行 kubeadm upgrade
   │   │   └── 重启服务
   │   └── 标记 Machine UpToDate
   │
   └── 2b. 滚动升级路径 (Fallback)
       ├── 创建新 Machine (新版本)
       ├── 等待新 Machine Ready
       └── 删除旧 Machine
   ↓
3. AfterControlPlaneUpgrade Hook
   ↓
4. 升级 Workers (MachineDeployments)
   ↓
5. AfterClusterUpgrade Hook (升级 Add-ons)
```

#### 6.3 原地升级实现 (In-Place Update Extension)
```go
// BinaryUpdateExtension 实现二进制组件原地升级
type BinaryUpdateExtension struct{}

func (e *BinaryUpdateExtension) CanUpdateMachine(
    ctx context.Context,
    req *runtimehooksv1.CanUpdateMachineRequest,
    resp *runtimehooksv1.CanUpdateMachineResponse,
) {
    // 检查变更是否为版本升级
    if req.Desired.Machine.Spec.Version != req.Current.Machine.Spec.Version {
        // 返回支持的变更 patch
        resp.MachinePatch = Patch{
            PatchType: "JSONPatch",
            Patch: []byte(`[{"op":"replace","path":"/spec/version","value":"` + req.Desired.Machine.Spec.Version + `"}]`),
        }
    }
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}

func (e *BinaryUpdateExtension) UpdateMachine(
    ctx context.Context,
    req *runtimehooksv1.UpdateMachineRequest,
    resp *runtimehooksv1.UpdateMachineResponse,
) {
    // 1. SSH 连接到目标机器
    client := e.sshConnect(req.Desired.Machine)
    
    // 2. 获取目标版本
    targetVersion := req.Desired.Machine.Spec.Version
    
    // 3. 执行升级脚本
    script := fmt.Sprintf(`#!/bin/bash
set -euo pipefail

# 备份当前版本
BACKUP_DIR=/opt/k8s-backup/$(date +%%Y%%m%%d_%%H%%M%%S)
mkdir -p $BACKUP_DIR
cp /usr/bin/kubeadm $BACKUP_DIR/
cp /usr/bin/kubelet $BACKUP_DIR/
cp /usr/bin/kubectl $BACKUP_DIR/

# 下载新版本
curl -o /usr/bin/kubeadm %s/kubeadm
curl -o /usr/bin/kubelet %s/kubelet
curl -o /usr/bin/kubectl %s/kubectl
chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl

# 执行 kubeadm upgrade
kubeadm upgrade apply %s -y

# 重启 kubelet
systemctl daemon-reload
systemctl restart kubelet

# 验证
kubeadm version -o short
kubelet --version
`, binaryRepo, binaryRepo, binaryRepo, targetVersion)

    output, err := client.Run(script)
    if err != nil {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = fmt.Sprintf("Upgrade failed: %v, output: %s", err, output)
        return
    }
    
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}
```

#### 6.4 回滚方案
**方案 A: CAPI 内置回滚** - 修改版本回退

**方案 B: 升级前快照 + 恢复**
```go
func (r *UpgradeOrchestrator) PreUpgradeSnapshot(ctx context.Context, cluster *clusterv1.Cluster) error {
    // 1. 备份 etcd
    etcdClient := r.getEtcdClient(cluster)
    snapshot, _ := etcdClient.Snapshot(ctx)
    r.saveSnapshot(snapshot, cluster.Name)
    
    // 2. 备份关键 CRD 资源
    resources := []string{
        "clusters.cluster.x-k8s.io",
        "machines.cluster.x-k8s.io",
        "kubeadmcontrolplanes.controlplane.cluster.x-k8s.io",
        "machinedeployments.cluster.x-k8s.io",
    }
    for _, res := range resources {
        r.backupResources(ctx, res, cluster.Namespace)
    }
    
    return nil
}
```

**方案 C: 原地升级失败回滚**
```bash
#!/bin/bash
# 原地升级回滚脚本
set -euo pipefail

# 获取最新备份目录
BACKUP_DIR=$(ls -td /opt/k8s-backup/*/ | head -1)

if [ -z "$BACKUP_DIR" ]; then
    echo "No backup found"
    exit 1
fi

echo "Rolling back from $BACKUP_DIR"

# 恢复二进制
cp $BACKUP_DIR/kubeadm /usr/bin/kubeadm
cp $BACKUP_DIR/kubelet /usr/bin/kubelet
cp $BACKUP_DIR/kubectl /usr/bin/kubectl
chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl

# 恢复 etcd (如果是 etcd 升级失败)
# etcdctl snapshot restore $BACKUP_DIR/etcd-snapshot.db --data-dir /var/lib/etcd

# 重启服务
systemctl daemon-reload
systemctl restart kubelet

echo "Rollback completed"
```

**方案 D: Lifecycle Hooks 升级门禁**
```go
func (v *UpgradeValidator) BeforeClusterUpgrade(...) {
    // 1. 验证版本兼容性
    if !v.isCompatibleUpgrade(req.FromKubernetesVersion, req.ToKubernetesVersion) {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = "Incompatible upgrade path"
        resp.RetryAfterSeconds = 0
        return
    }
    
    // 2. 执行升级前检查
    checks := []func() error{
        v.checkEtcdHealth,
        v.checkNodeReady,
        v.checkStorageAvailable,
        v.checkBackupCompleted,
    }
    
    for _, check := range checks {
        if err := check(); err != nil {
            resp.Status = runtimehooksv1.ResponseStatusFailure
            resp.Message = fmt.Sprintf("Pre-upgrade check failed: %v", err)
            resp.RetryAfterSeconds = 60
            return
        }
    }
    
    // 3. 创建升级前快照
    v.createPreUpgradeSnapshot(req.Cluster)
    
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}
```

### 需求 7: 大规模集群控制面组件分离部署
**kubeadm 核心限制**:
1. `kubeadm init` 和 `kubeadm join --control-plane` 会在节点上部署 **所有** 控制面组件
2. 没有配置项可以禁止某个节点运行特定组件
3. static pod manifest 由 kubeadm 自动生成和管理

**解决方案对比**:

| 方案 | 复杂度 | 可维护性 | 推荐度 |
|------|--------|----------|--------|
| A. 接受同节点部署 | 低 | 高 | ⭐⭐⭐⭐⭐ |
| B. 自定义 ControlPlane Provider | 高 | 中 | ⭐⭐⭐ |
| C. 修改 static pod manifests | 中 | 低 | ⭐ |

**方案 A (强烈推荐)**: 9 节点控制面，每个节点运行全部组件
- API Server 无状态，通过 LB 水平扩展
- Scheduler/Controller Manager 通过 leader election 保证单 active
- 实际资源消耗: Scheduler ~50MB, Controller Manager ~100MB

### 需求 8: 开源组件分析
| 组件 | 作用 | 版本 | 限制 |
|------|------|------|------|
| **Cluster API** | 核心框架 | v1beta2 | 不直接管理裸金属 |
| **Kubeadm Bootstrap** | 生成 kubeadm 配置 | v1beta2 | 控制面组件必须同节点 |
| **Kubeadm Control Plane** | 管理控制面生命周期 | v1beta2 | 仅支持 stacked/external etcd |
| **CAPBM (自研)** | 裸金属 Infrastructure Provider | 自研 | 需自行实现 SSH/电源管理 |
| **Runtime SDK** | 扩展机制 (Hooks + Patches) | v1alpha1 | **experimental 阶段** |
| **In-Place Update** | 原地升级扩展 | Alpha (feature gate) | **不支持自动回滚**，失败需手动修复或替换机器 |
| **cert-manager** | 证书自动轮转 | v1.14+ | 需手动集成 CAPI CA |
| **Metal3** | 现有裸金属 Provider | v1beta1 | 需要 PXE，不适用于已有 OS |
| **kubeadm** | K8s 引导工具 | v1.28+ | 控制面组件同节点绑定 |
| **coredns/corefile-migration** | CoreDNS 配置迁移 | 内置于 KCP | 仅管理版本升级时的 corefile 迁移 |

**关键限制总结**:

| 限制项 | 影响 | 缓解方案 |
|--------|------|----------|
| kubeadm 不支持控制面组件分离 | 需求 7 无法完全满足 | 接受同节点部署，或开发自定义 Provider |
| kubeadm 不管理 external etcd 生命周期 | etcd 独立节点需自行管理 | Runtime Extension + MachineDeployment |
| Runtime SDK 仍为 experimental | API 可能变化 | 关注 CAPI 社区进展，做好版本锁定 |
| **In-Place Update 不支持自动回滚** | 需求 6 回滚需自行实现 | 升级前备份 + 手动回滚脚本 |
| CAPI 不处理包安装 | 需在 bootstrap 阶段处理 | preKubeadmCommands 中完成 |
| Metal3 不适用于已有 OS 场景 | 不能直接使用现有裸金属 Provider | 必须自研 CAPBM |

### 需求 9: 在 OS 上安装二进制组件的设计
**可行性**: ✅ 完全可行

**技术方案**: 通过 `KubeadmConfigSpec.preKubeadmCommands` + `Files` 实现

#### 9.1 整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                    二进制安装流程 (preKubeadmCommands)               │
│                                                                      │
│  1. 环境检查                                                         │
│     ├── OS 版本验证 (CentOS/Ubuntu 等)                               │
│     ├── 内核参数检查                                                 │
│     ├── 网络连通性检查                                               │
│     └── 磁盘空间检查                                                 │
│                                                                      │
│  2. 系统依赖安装                                                     │
│     ├── socat, conntrack, ipset, ebtables                            │
│     └── 内核模块加载 (br_netfilter, overlay)                         │
│                                                                      │
│  3. NTP 安装与配置                                                   │
│     ├── chrony 或 ntp 安装                                           │
│     └── 配置 NTP 服务器                                              │
│                                                                      │
│  4. Containerd 安装与配置                                            │
│     ├── 二进制下载/包安装                                            │
│     ├── runc 安装                                                    │
│     ├── CNI 插件安装                                                 │
│     └── 配置文件生成                                                 │
│                                                                      │
│  5. Kubernetes 二进制安装                                            │
│     ├── kubeadm 安装                                                 │
│     ├── kubelet 安装                                                 │
│     ├── kubectl 安装                                                 │
│     └── 版本一致性验证                                               │
│                                                                      │
│  6. 系统配置                                                         │
│     ├── 内核参数设置 (sysctl)                                        │
│     ├── 防火墙/SELinux 配置                                          │
│     ├── Swap 禁用                                                    │
│     └── 服务启用 (kubelet, containerd)                               │
│                                                                      │
│  7. 安装验证                                                         │
│     ├── 二进制版本检查                                               │
│     ├── 服务状态检查                                                 │
│     └── 网络连通性验证                                               │
└─────────────────────────────────────────────────────────────────────┘
```

#### 9.2 详细实现方案
**方案 A: 通过包管理器安装 (推荐)**
```yaml
kubeadmConfigSpec:
  preKubeadmCommands:
    - |
      #!/bin/bash
      set -euo pipefail
      
      echo "=== [1/7] 环境检查 ==="
      if [ -f /etc/os-release ]; then
        . /etc/os-release
        case "$ID" in
          centos|rhel|almalinux|rocky) PKG_MGR="yum" ;;
          ubuntu|debian) PKG_MGR="apt" ;;
          *) echo "Unsupported OS: $ID"; exit 1 ;;
        esac
      else
        echo "Cannot detect OS"; exit 1
      fi
      
      # 内核版本检查
      KERNEL_VERSION=$(uname -r | cut -d'-' -f1)
      REQUIRED_VERSION="3.10"
      if [ "$(printf '%s\n' "$REQUIRED_VERSION" "$KERNEL_VERSION" | sort -V | head -n1)" != "$REQUIRED_VERSION" ]; then
        echo "Kernel version $KERNEL_VERSION is too old"; exit 1
      fi
      
      # 磁盘空间检查
      AVAILABLE_GB=$(df -BG / | awk 'NR==2 {print $4}' | tr -d 'G')
      if [ "$AVAILABLE_GB" -lt 20 ]; then
        echo "Insufficient disk space: ${AVAILABLE_GB}GB"; exit 1
      fi
      
      echo "=== [2/7] 系统依赖安装 ==="
      modprobe br_netfilter 2>/dev/null || true
      modprobe overlay 2>/dev/null || true
      
      cat > /etc/modules-load.d/k8s.conf <<EOF
      br_netfilter
      overlay
      EOF
      
      cat > /etc/sysctl.d/k8s.conf <<EOF
      net.bridge.bridge-nf-call-iptables = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward = 1
      vm.overcommit_memory = 1
      fs.inotify.max_user_watches = 524288
      EOF
      sysctl --system
      
      if [ "$PKG_MGR" = "yum" ]; then
        yum install -y socat conntrack-tools ipset ebtables
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get update && apt-get install -y socat conntrack ipset ebtables
      fi
      
      echo "=== [3/7] NTP 安装与配置 ==="
      if [ "$PKG_MGR" = "yum" ]; then
        yum install -y chrony
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get install -y chrony
      fi
      
      cat > /etc/chrony.conf <<EOF
      {{ range .externalServices.ntp.servers }}
      server {{ . }} iburst
      {{ end }}
      makestep 1 3
      rtcsync
      EOF
      
      systemctl enable --now chronyd
      chronyc waitsync 30
      
      echo "=== [4/7] Containerd 安装与配置 ==="
      if [ "$PKG_MGR" = "yum" ]; then
        cat > /etc/yum.repos.d/containerd.repo <<EOF
        [containerd]
        name=containerd
        baseurl={{ .externalServices.packageMirror.url }}/containerd
        enabled=1
        gpgcheck=0
        EOF
        yum install -y containerd.io runc
      elif [ "$PKG_MGR" = "apt" ]; then
        apt-get update && apt-get install -y containerd.io runc
      fi
      
      # 安装 CNI 插件
      mkdir -p /opt/cni/bin
      curl -L {{ .externalServices.packageMirror.url }}/cni-plugins/cni-plugins-linux-amd64-v1.5.0.tgz | tar -C /opt/cni/bin -xz
      
      # 配置 containerd
      mkdir -p /etc/containerd
      containerd config default > /etc/containerd/config.toml
      sed -i 's|sandbox_image = "registry.k8s.io/pause:.*"|sandbox_image = "{{ .externalServices.registry.endpoint }}/pause:3.9"|' /etc/containerd/config.toml
      sed -i 's|SystemdCgroup = false|SystemdCgroup = true|' /etc/containerd/config.toml
      
      systemctl daemon-reload && systemctl enable --now containerd
      
      echo "=== [5/7] Kubernetes 二进制安装 ==="
      KUBE_VERSION="{{ .kubernetesVersion }}"
      
      if [ "$PKG_MGR" = "yum" ]; then
        cat > /etc/yum.repos.d/kubernetes.repo <<EOF
        [kubernetes]
        name=Kubernetes
        baseurl={{ .externalServices.packageMirror.url }}/kubernetes/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=0
        EOF
        yum install -y kubeadm-${KUBE_VERSION#v} kubelet-${KUBE_VERSION#v} kubectl-${KUBE_VERSION#v}
      elif [ "$PKG_MGR" = "apt" ]; then
        curl -fsSL {{ .externalServices.packageMirror.url }}/kubernetes/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
        cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
        deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] {{ .externalServices.packageMirror.url }}/kubernetes/apt/ kubernetes-xenial main
        EOF
        apt-get update && apt-get install -y kubeadm=${KUBE_VERSION#v}-* kubelet=${KUBE_VERSION#v}-* kubectl=${KUBE_VERSION#v}-*
        apt-mark hold kubeadm kubelet kubectl
      fi
      
      echo "=== [6/7] 系统配置 ==="
      swapoff -a && sed -i '/swap/d' /etc/fstab
      systemctl disable --now firewalld 2>/dev/null || true
      setenforce 0 2>/dev/null || true
      systemctl enable kubelet
      
      echo "=== [7/7] 安装验证 ==="
      KUBEADM_VERSION=$(kubeadm version -o short)
      KUBELET_VERSION=$(kubelet --version | awk '{print $2}')
      CONTAINERD_VERSION=$(containerd --version | awk '{print $3}')
      
      echo "kubeadm: $KUBEADM_VERSION"
      echo "kubelet: $KUBELET_VERSION"
      echo "containerd: $CONTAINERD_VERSION"
      
      if [ "$KUBEADM_VERSION" != "$KUBELET_VERSION" ]; then
        echo "ERROR: kubeadm version != kubelet version"; exit 1
      fi
      
      systemctl is-active --quiet containerd || { echo "ERROR: containerd not running"; exit 1; }
      systemctl is-active --quiet chronyd || { echo "ERROR: chronyd not running"; exit 1; }
      
      echo "=== 所有前置检查通过 ==="
```

#### 9.3 原地升级设计 (In-Place Update Extension)
**CAPI In-Place Update 机制**:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    原地升级流程                                      │
│                                                                     │
│  1. 用户修改 KCP/MD 的 version 字段                                  │
│     ↓                                                               │
│  2. KCP/MD Controller 检测变更                                       │
│     ↓                                                                │
│  3. 调用 CanUpdateMachine Hook                                       │
│     ├── 检查变更是否支持原地升级                                      │
│     ├── 返回支持的变更 patch                                         │
│     └── 如果所有变更都支持 → 进入原地升级路径                         │
│         否则 → Fallback 到滚动升级                                    │
│     ↓                                                                │
│  4. 标记 Machine 为 Updating in-place                                │
│     ↓                                                                │
│  5. Machine Controller 调用 UpdateMachine Hook                       │
│     ├── SSH 连接到目标机器                                           │
│     ├── 执行升级脚本:                                                │
│     │   ├── 备份当前版本二进制                                        │
│     │   ├── 下载新版本二进制                                          │
│     │   ├── 替换二进制文件                                           │
│     │   ├── 执行 kubeadm upgrade apply                               │
│     │   ├── 重启 kubelet                                             │
│     │   └── 验证升级结果                                             │
│     └── 返回升级状态 (Success/Retry/Failure)                         │
│     ↓                                                               │
│  6. 升级完成 → 标记 Machine 为 UpToDate                              │
│     ↓                                                               │
│  7. 升级失败 → 需要手动修复或替换机器                                 │
│     ⚠️ CAPI In-Place Update 当前不支持自动回滚                       │
└─────────────────────────────────────────────────────────────────────┘
```

**In-Place Update Extension 实现**:
```go
// BinaryUpgradeExtension 实现二进制组件原地升级
type BinaryUpgradeExtension struct {
    client.Client
    SSHClient SSHClient
}

func (e *BinaryUpgradeExtension) CanUpdateMachine(
    ctx context.Context,
    req *runtimehooksv1.CanUpdateMachineRequest,
    resp *runtimehooksv1.CanUpdateMachineResponse,
) {
    // 检查是否为版本升级
    currentVersion := req.Current.Machine.Spec.Version
    desiredVersion := req.Desired.Machine.Spec.Version
    
    if currentVersion != desiredVersion {
        // 支持版本升级的原地更新
        resp.MachinePatch = runtimehooksv1.Patch{
            PatchType: "JSONPatch",
            Patch: []byte(fmt.Sprintf(
                `[{"op":"replace","path":"/spec/version","value":"%s"}]`,
                desiredVersion,
            )),
        }
    }
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}

func (e *BinaryUpgradeExtension) UpdateMachine(
    ctx context.Context,
    req *runtimehooksv1.UpdateMachineRequest,
    resp *runtimehooksv1.UpdateMachineResponse,
) {
    machine := req.Desired.Machine
    targetVersion := machine.Spec.Version
    
    // 1. 获取机器连接信息
    ip := e.getMachineIP(machine)
    creds := e.getMachineCredentials(machine)
    
    // 2. SSH 连接
    client, err := e.SSHClient.Connect(ip, 22, creds)
    if err != nil {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = fmt.Sprintf("SSH connection failed: %v", err)
        return
    }
    
    // 3. 执行原地升级脚本
    script := e.buildUpgradeScript(targetVersion)
    output, err := client.Run(script)
    
    if err != nil {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = fmt.Sprintf("Upgrade failed: %v\nOutput: %s", err, output)
        resp.RetryAfterSeconds = 0  // 不重试，需要手动修复
        return
    }
    
    // 4. 验证升级结果
    verifyOutput, err := client.Run(`
        kubeadm version -o short
        kubelet --version
        systemctl is-active kubelet
    `)
    if err != nil {
        resp.Status = runtimehooksv1.ResponseStatusFailure
        resp.Message = fmt.Sprintf("Verification failed: %v", err)
        return
    }
    
    resp.Status = runtimehooksv1.ResponseStatusSuccess
    resp.RetryAfterSeconds = 0
}

func (e *BinaryUpgradeExtension) buildUpgradeScript(targetVersion string) string {
    return fmt.Sprintf(`#!/bin/bash
set -euo pipefail

# ===== 升级前备份 =====
BACKUP_DIR=/opt/k8s-backup/$(date +%%Y%%m%%d_%%H%%M%%S)
mkdir -p "$BACKUP_DIR"

# 备份二进制
cp /usr/bin/kubeadm "$BACKUP_DIR/" 2>/dev/null || true
cp /usr/bin/kubelet "$BACKUP_DIR/" 2>/dev/null || true
cp /usr/bin/kubectl "$BACKUP_DIR/" 2>/dev/null || true

# 备份 systemd 服务文件
cp /etc/systemd/system/kubelet.service "$BACKUP_DIR/" 2>/dev/null || true
cp -r /etc/systemd/system/kubelet.service.d "$BACKUP_DIR/" 2>/dev/null || true

# 备份 etcd 数据 (如果是 control plane 节点)
if [ -d /var/lib/etcd ]; then
    etcdctl snapshot save "$BACKUP_DIR/etcd-snapshot.db" 2>/dev/null || true
fi

# 备份 kubeadm 配置
cp /etc/kubernetes/kubeadm-config.yaml "$BACKUP_DIR/" 2>/dev/null || true
cp -r /etc/kubernetes/manifests "$BACKUP_DIR/" 2>/dev/null || true
cp -r /etc/kubernetes/pki "$BACKUP_DIR/" 2>/dev/null || true

echo "Backup completed: $BACKUP_DIR"

# ===== 下载新版本二进制 =====
KUBE_VERSION="%s"
BINARY_URL="{{ .packageMirror.url }}/kubernetes/bin/${KUBE_VERSION#v}"

curl -fSL -o /usr/bin/kubeadm "${BINARY_URL}/kubeadm"
curl -fSL -o /usr/bin/kubelet "${BINARY_URL}/kubelet"
curl -fSL -o /usr/bin/kubectl "${BINARY_URL}/kubectl"

chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl

# ===== 执行 kubeadm upgrade =====
kubeadm upgrade apply "${KUBE_VERSION}" -y --force

# ===== 重启 kubelet =====
systemctl daemon-reload
systemctl restart kubelet

# ===== 验证 =====
echo "=== Upgrade Verification ==="
echo "kubeadm: $(kubeadm version -o short)"
echo "kubelet: $(kubelet --version | awk '{print $2}')"
echo "kubelet status: $(systemctl is-active kubelet)"

# 版本一致性检查
KUBEADM_VER=$(kubeadm version -o short)
KUBELET_VER=$(kubelet --version | awk '{print $2}')
if [ "$KUBEADM_VER" != "$KUBELET_VER" ]; then
    echo "ERROR: Version mismatch!"
    exit 1
fi

echo "=== Upgrade completed successfully ==="
`, targetVersion)
}
```

#### 9.4 失败回滚到上一版本
**⚠️ 重要限制**: CAPI In-Place Update **当前不支持自动回滚** (proposal 明确列为 Non-Goal)。

**自行实现回滚方案**:

**方案 A: 升级前自动备份 + 手动回滚**
```yaml
# 升级前通过 BeforeClusterUpgrade Hook 触发备份
patches:
  - name: pre-upgrade-backup
    definitions:
      - selector:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
          kind: KubeadmConfigTemplate
          matchResources:
            machineDeploymentClass:
              names: ["*"]
        jsonPatches:
          - op: add
            path: /spec/template/spec/preKubeadmCommands/-
            value: |
              # 检查是否有升级标记
              if [ -f /tmp/.k8s-upgrade-in-progress ]; then
                # 这是升级后的第一次 reconcile，检查是否需要回滚
                if [ -f /tmp/.k8s-upgrade-failed ]; then
                  echo "Upgrade failed, rolling back..."
                  /opt/k8s-rollback.sh
                  exit 0
                fi
              fi
```

**方案 B: 自定义回滚 Controller**
```go
type RollbackReconciler struct {
    client.Client
}

func (r *RollbackReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    machine := &clusterv1.Machine{}
    r.Get(ctx, req.NamespacedName, machine)
    
    // 检查 Machine 更新状态
    for _, cond := range machine.Status.Conditions {
        if cond.Type == "Updating" && cond.Status == metav1.ConditionFalse {
            // 更新失败，触发回滚
            if err := r.triggerRollback(ctx, machine); err != nil {
                return ctrl.Result{RequeueAfter: 30 * time.Second}, err
            }
        }
    }
    
    return ctrl.Result{}, nil
}

func (r *RollbackReconciler) triggerRollback(ctx context.Context, machine *clusterv1.Machine) error {
    // 1. 获取备份信息
    backupDir := r.getLatestBackup(machine)
    
    // 2. SSH 连接执行回滚
    client := r.sshConnect(machine)
    
    rollbackScript := fmt.Sprintf(`#!/bin/bash
set -euo pipefail

BACKUP_DIR="%s"

# 恢复二进制
cp "$BACKUP_DIR/kubeadm" /usr/bin/kubeadm
cp "$BACKUP_DIR/kubelet" /usr/bin/kubelet
cp "$BACKUP_DIR/kubectl" /usr/bin/kubectl
chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl

# 恢复 etcd (如果有备份)
if [ -f "$BACKUP_DIR/etcd-snapshot.db" ]; then
    systemctl stop kubelet
    etcdctl snapshot restore "$BACKUP_DIR/etcd-snapshot.db" --data-dir /var/lib/etcd
fi

# 恢复 kubeadm 配置
if [ -d "$BACKUP_DIR/manifests" ]; then
    cp -r "$BACKUP_DIR/manifests"/* /etc/kubernetes/manifests/
fi
if [ -d "$BACKUP_DIR/pki" ]; then
    cp -r "$BACKUP_DIR/pki"/* /etc/kubernetes/pki/
fi

# 重启服务
systemctl daemon-reload
systemctl restart kubelet

# 清理标记
rm -f /tmp/.k8s-upgrade-failed

echo "Rollback completed from $BACKUP_DIR"
`, backupDir)
    
    return client.Run(rollbackScript)
}
```

**方案 C: 版本回退 (通过修改 KCP version 触发滚动升级)**
```yaml
# 升级失败时，将版本回退到旧版本
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: my-cluster-cp
spec:
  version: "v1.30.0"  # 从 v1.31.0 回退到 v1.30.0
```

#### 9.5 版本管理 CRD
```yaml
apiVersion: upgrade.cluster.x-k8s.io/v1alpha1
kind: VersionManifest
metadata:
  name: v1.31.0
spec:
  kubernetesVersion: "v1.31.0"
  components:
    # 控制面组件
    - name: kube-apiserver
      type: static-pod
      image: "{{ registry }}/kube-apiserver:v1.31.0"
    - name: kube-controller-manager
      type: static-pod
      image: "{{ registry }}/kube-controller-manager:v1.31.0"
    - name: kube-scheduler
      type: static-pod
      image: "{{ registry }}/kube-scheduler:v1.31.0"
    - name: etcd
      type: static-pod
      image: "{{ registry }}/etcd:3.5.15-0"
    # 核心插件
    - name: coredns
      type: deployment
      namespace: kube-system
      image: "{{ registry }}/coredns:1.11.1"
    - name: kube-proxy
      type: daemonset
      namespace: kube-system
      image: "{{ registry }}/kube-proxy:v1.31.0"
    - name: calico
      type: operator
      version: "v3.28.0"
    # 二进制组件 (支持原地升级)
    - name: kubeadm
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubeadm"
      checksum: "sha256:abc123..."
      inplaceUpgrade: true
    - name: kubelet
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubelet"
      checksum: "sha256:def456..."
      inplaceUpgrade: true
    - name: kubectl
      type: binary
      version: "v1.31.0"
      url: "{{ registry }}/kubernetes/bin/v1.31.0/kubectl"
      checksum: "sha256:ghi789..."
      inplaceUpgrade: true
    - name: containerd
      type: binary
      version: "1.7.20"
      url: "{{ registry }}/containerd/containerd-1.7.20-linux-amd64.tar.gz"
      checksum: "sha256:jkl012..."
      inplaceUpgrade: true
    - name: runc
      type: binary
      version: "1.1.13"
      url: "{{ registry }}/containerd/runc.amd64"
      checksum: "sha256:mno345..."
      inplaceUpgrade: true
    - name: cni-plugins
      type: binary
      version: "1.5.0"
      url: "{{ registry }}/cni-plugins/cni-plugins-linux-amd64-v1.5.0.tgz"
      checksum: "sha256:pqr678..."
      inplaceUpgrade: true
    - name: chrony
      type: binary
      version: "4.4"
      url: "{{ registry }}/ntp/chrony-4.4.tar.gz"
      checksum: "sha256:stu901..."
      inplaceUpgrade: true
```

#### 9.6 方案对比
| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **A. 包管理器安装** | 有镜像源/包管理源 | 标准化、易维护、支持自动依赖 | 需要配置源 |
| **B. 二进制直接安装** | 离线环境 | 不依赖包管理器 | 需要手动管理服务文件 |
| **C. CAPBM 预安装** | 大规模集群 | 与 CAPI 深度集成、可复用 | 开发工作量大 |

**推荐**:
- 中小规模: **方案 A** (包管理器)
- 离线环境: **方案 B** (二进制)
- 大规模 (>100节点): **方案 C** (CAPBM 预安装)

## 三、核心问答总结

| 需求 | kubeadm 是否支持 | 解决方案 |
|------|------------------|----------|
| etcd 外接配置 | ✅ 支持 | `clusterConfiguration.etcd.external` |
| etcd 标签指定节点 | ⚠️ 部分 | stacked: 原生; external: 需 Runtime Extension |
| API Server 标签指定节点 | ❌ 不支持 | 所有 control plane 节点都运行 API Server |
| API Server 扩缩容 | ✅ 支持 | 修改 KCP `spec.replicas` |
| API Server 指定 LB | ✅ 支持 | `controlPlaneEndpoint` |
| Scheduler 主备 | ✅ 支持 | leader election 默认启用 |
| Controller Manager 主备 | ✅ 支持 | leader election 默认启用 |
| 控制面组件分离部署 | ❌ 不支持 | 需自定义 Provider 或接受同节点 |
| kubelet 定制 | ✅ 支持 | `kubeletExtraArgs` + `kubeletConfig` |
| containerd 定制 | ✅ 支持 | `files` + `preKubeadmCommands` |
| 证书自动轮转 | ✅ 支持 | KCP `rolloutBefore.certificatesExpiryDays` |
| 二进制安装 | ✅ 支持 | `preKubeadmCommands` + `Files` |
| **组件原地升级** | ⚠️ Alpha 阶段 | In-Place Update Extension (feature gate) |
| **升级自动回滚** | ❌ 不支持 | 需自行实现 (备份 + 回滚脚本) |
