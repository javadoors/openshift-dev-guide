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
7. 在大规模集群下，api-server/controller-manager/scheduler需要分别部署在不同节点上
8. 分析使用的开源组件及其作用与限制
9. 在OS上安装kubelet/kubectl/containerd/ntp等二进制组件的设计

# 基于 Cluster-API 的裸金属集群全生命周期管理方案
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
