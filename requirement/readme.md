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

# 基于 Cluster-API 的裸金属集群管理方案设计
## 一、总体架构
```
┌─────────────────────────────────────────────────────────────┐
│                    Management Cluster                       │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │   CAPI      │  │ CAPBM        │  │  Runtime Extension  │ │
│  │  Core       │  │ (Bare Metal) │  │  (Addon Manager)    │ │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬──────────┘ │
│         │                │                      │            │
│  ┌──────▼────────────────▼──────────────────────▼──────────┐ │
│  │              ClusterClass + Topology                    │ │
│  │  ┌────────────┐  ┌─────────────┐  ┌──────────────────┐  │ │
│  │  │ Kubeadm    │  │ Machine     │  │ Lifecycle Hooks  │  │ │
│  │  │ControlPlane│  │Deployment   │  │ & Patches        │  │ │
│  │  └────────────┘  └─────────────┘  └──────────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   Workload Cluster (Bare Metal)             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Control Plane│  │   Workers    │  │    Add-ons        │  │
│  │ etcd/k8s     │  │  (labeled)   │  │ calico/coredns/   │  │
│  │              │  │              │  │ kube-proxy + apps │  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 二、需求逐项分析与技术方案
### 需求 1: 机器列表管理（主机名、IP、用户名、密码）
**可行性**: ✅ 完全可行

**技术选型**: 开发自定义 Infrastructure Provider `CAPBM` (Cluster API Provider Bare Metal)

**设计方案**:
```go
// BareMetalMachine CRD 示例
type BareMetalMachineSpec struct {
    HostName     string              `json:"hostName"`
    IPAddress    string              `json:"ipAddress"`
    SSHPort      int                 `json:"sshPort,omitempty"`
    Credentials  SecretReference     `json:"credentials"`  // 引用包含用户名/密码的Secret
    Power        PowerConfig         `json:"power,omitempty"`  // IPMI/Redfish电源管理
}

type BareMetalMachineStatus struct {
    ProviderID   string              `json:"providerID,omitempty"`  // format: baremetal://<hostname>
    Ready        bool                `json:"ready"`
    Addresses    []MachineAddress    `json:"addresses,omitempty"`
}
```

**关键设计点**:
- 使用 SSH 连接已安装OS的机器
- 通过 `ProviderID` 格式 `baremetal://<hostname>` 唯一标识机器
- Credentials 存储为 Kubernetes Secret，避免明文暴露
- 可选集成 IPMI/Redfish 实现电源管理（重启、关机）

### 需求 2: 外部服务集成（NFS、NTP、镜像仓库、二进制安装源、chart仓库、外部LB）
**可行性**: ✅ 完全可行

**技术方案**: 通过 ClusterClass Variables + Bootstrap Configuration 注入
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
              insecure: { type: boolean }
          packageMirror:
            type: object
            properties:
              aptUrl: { type: string }
              yumBaseUrl: { type: string }
          helmRegistry:
            type: object
            properties:
              endpoint: { type: string }
          externalLoadBalancer:
            type: object
            properties:
              endpoint: { type: string }
              port: { type: integer }
```

**在 KubeadmConfig 中使用 preKubeadmCommands 配置**:
```yaml
preKubeadmCommands:
  - |
    # 配置NTP
    cat > /etc/chrony.conf <<EOF
    {{ range .externalServices.ntp.servers }}
    server {{ . }} iburst
    {{ end }}
    EOF
    systemctl restart chronyd

    # 配置镜像仓库
    cat > /etc/containerd/config.d/registry.toml <<EOF
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["{{ .externalServices.registry.endpoint }}"]
    EOF

    # 挂载NFS共享
    mkdir -p /mnt/shared-data
    mount -t nfs {{ .externalServices.nfs.server }}:{{ .externalServices.nfs.shares[0] }} /mnt/shared-data
```

### 需求 3: K8s 控制面配置

#### 3.1 etcd 支持外接配置
**可行性**: ✅ 原生支持

**技术方案**: 使用 `ClusterConfiguration.Etcd.External`
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    etcd:
      external:
        endpoints:
          - "https://etcd-1:2379"
          - "https://etcd-2:2379"
          - "https://etcd-3:2379"
        caFile: "/etc/kubernetes/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
```

#### 3.2 etcd 通过标签指定节点并自动化安装
**可行性**: ✅ 通过 KubeadmControlPlane + 标签选择实现

**方案 A: 托管 etcd（Stacked）** - 推荐用于大多数场景
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: my-cluster-cp
spec:
  replicas: 3
  machineTemplate:
    metadata:
      labels:
        node-role.kubernetes.io/etcd: ""
    spec:
      infrastructureRef: ...
  kubeadmConfigSpec:
    clusterConfiguration:
      etcd:
        local:
          dataDir: /var/lib/etcd
          extraArgs:
            - name: listen-metrics-urls
              value: "http://0.0.0.0:2381"
```

**方案 B: 分离 etcd 节点** - 需要自定义 ControlPlane Provider

> ⚠️ **技术限制说明**: 标准的 KubeadmControlPlane 将 etcd 与控制面节点绑定。若需要独立的 etcd 节点池（与控制面节点分离），有两种方案：
>
> 1. **External etcd + MachineDeployment**: 使用 External etcd 配置，通过 MachineDeployment 管理独立的 etcd 节点，使用 Runtime Extension 在 etcd 节点上安装 etcd 集群
> 2. **自定义 ControlPlane Provider**: 开发扩展的 ControlPlane Provider 支持分离部署

**推荐方案**: 使用 Runtime Extension + Lifecycle Hook 实现 etcd 节点的独立管理
```yaml
# 通过 MachineDeployment 管理 etcd 专用节点
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  name: etcd-nodes
spec:
  replicas: 3
  selector:
    matchLabels:
      role: etcd
  template:
    spec:
      clusterName: my-cluster
      version: v1.31.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
          kind: KubeadmConfig
          name: etcd-bootstrap
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: BareMetalMachineTemplate
        name: etcd-machine-template
```

#### 3.3 API Server 标签指定节点、扩缩容、指定负载均衡器
**可行性**: ✅ 完全支持
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  replicas: 3  # 支持动态扩缩容
  kubeadmConfigSpec:
    clusterConfiguration:
      controlPlaneEndpoint: "{{ .externalServices.externalLoadBalancer.endpoint }}:{{ .externalServices.externalLoadBalancer.port }}"
      apiServer:
        extraArgs:
          - name: advertise-address
            value: "${LOCAL_IP}"
          - name: audit-log-path
            value: /var/log/kubernetes/audit.log
        certSANs:
          - "{{ .externalServices.externalLoadBalancer.endpoint }}"
```
**扩缩容**: 直接修改 `spec.replicas` 或通过 HPA-like 控制器自动调整

#### 3.4 Scheduler 和 Controller-Manager 主备模式
**可行性**: ✅ 通过 kubeadm 原生支持
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    scheduler:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
    controllerManager:
      extraArgs:
        - name: leader-elect
          value: "true"
        - name: leader-elect-resource-lock
          value: "leases"
```
> **说明**: kubeadm 部署的 scheduler 和 controller-manager 默认启用 leader election，多副本时自动实现主备模式

#### 3.5 Kubelet 定制化配置
**可行性**: ✅ 通过 `NodeRegistrationOptions.KubeletExtraArgs` + `Files`
```yaml
kubeadmConfigSpec:
  initConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        - name: kube-reserved
          value: "cpu=500m,memory=512Mi"
        - name: system-reserved
          value: "cpu=200m,memory=256Mi"
        - name: eviction-hard
          value: "memory.available<500Mi,nodefs.available<10%"
  # 针对特定节点的差异化配置通过 ClusterClass Patch 实现
```

**节点差异化配置** - 使用 ClusterClass Patches:
```yaml
# ClusterClass Patch - 根据标签应用不同配置
patches:
  - name: kubelet-custom-config
    definitions:
      - selector:
          apiVersion: controlplane.cluster.x-k8s.io/v1beta2
          kind: KubeadmControlPlaneTemplate
          matchResources:
            controlPlane: true
        jsonPatches:
          - op: add
            path: /spec/template/spec/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs
            valueFrom:
              variable: "customKubeletArgs"
```

#### 3.6 Containerd 定制化配置
**可行性**: ✅ 通过 `preKubeadmCommands` + `Files`
```yaml
kubeadmConfigSpec:
  files:
    - path: /etc/containerd/config.toml
      content: |
        version = 2
        [plugins]
          [plugins."io.containerd.grpc.v1.cri"]
            [plugins."io.containerd.grpc.v1.cri".containerd]
              default_runtime_name = "runc"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                runtime_type = "io.containerd.runc.v2"
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                  SystemdCgroup = true
  preKubeadmCommands:
    - systemctl restart containerd
```

### 需求 4: 扩展机制安装组件（Calico、CoreDNS、Kube-proxy、其他应用）
**可行性**: ✅ 完全可行

#### 4.1 Calico / CoreDNS / Kube-proxy
**方案 A: 使用 KubeadmControlPlane 内置管理**（CoreDNS、Kube-proxy）
```yaml
# 跳过内置的 CoreDNS/Kube-proxy 管理，使用自定义版本
metadata:
  annotations:
    controlplane.cluster.x-k8s.io/skip-coredns: "true"
    controlplane.cluster.x-k8s.io/skip-kube-proxy: "true"
```

**方案 B: 使用 Runtime Extension + AfterControlPlaneInitialized Hook**（推荐）
```go
// Runtime Extension 实现
type AddonInstaller struct{}

func (a *AddonInstaller) AfterControlPlaneInitialized(
    ctx context.Context,
    req *runtimehooksv1.AfterControlPlaneInitializedRequest,
    resp *runtimehooksv1.AfterControlPlaneInitializedResponse,
) {
    // 获取 workload cluster client
    cl, err := a.getWorkloadClusterClient(req.Cluster)
    
    // 按依赖顺序安装
    addons := []Addon{
        {Name: "calico", Manifest: calicoManifests, Priority: 1},
        {Name: "kube-proxy", Manifest: kubeProxyManifests, Priority: 2},
        {Name: "coredns", Manifest: corednsManifests, Priority: 3},
    }
    
    for _, addon := range addons {
        if err := a.installAddon(cl, addon); err != nil {
            resp.Status = runtimehooksv1.ResponseStatusFailure
            resp.Message = fmt.Sprintf("Failed to install %s: %v", addon.Name, err)
            return
        }
    }
    
    resp.Status = runtimehooksv1.ResponseStatusSuccess
}
```

#### 4.2 应用拓扑化安装（依赖关系）

**技术方案**: Runtime Extension + GeneratePatches Hook + 自定义 Controller
```yaml
# ClusterClass 中定义应用及其依赖
variables:
  - name: addons
    schema:
      openAPIV3Schema:
        type: array
        items:
          type: object
          properties:
            name: { type: string }
            chart:
              type: object
              properties:
                repo: { type: string }
                name: { type: string }
                version: { type: string }
            dependsOn:
              type: array
              items: { type: string }
            values:
              type: object
              x-kubernetes-preserve-unknown-fields: true
```

**拓扑安装 Controller 设计**:
```go
// AddonTopologyReconciler 实现依赖感知的安装
type AddonTopologyReconciler struct {
    client.Client
}

func (r *AddonTopologyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    cluster := &clusterv1.Cluster{}
    r.Get(ctx, req.NamespacedName, cluster)
    
    // 构建依赖图
    graph := r.buildDependencyGraph(cluster.Spec.Topology.Variables)
    
    // 拓扑排序
    order, err := graph.TopologicalSort()
    if err != nil {
        return ctrl.Result{}, fmt.Errorf("circular dependency detected: %v", err)
    }
    
    // 按顺序安装
    for _, addon := range order {
        if !r.isDependencyReady(addon.DependsOn) {
            // 依赖未就绪，等待
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
        r.installAddon(ctx, cluster, addon)
    }
    
    return ctrl.Result{}, nil
}
```

**或使用 Lifecycle Hooks 实现**:
```yaml
# 使用 BeforeClusterCreate / AfterControlPlaneInitialized 钩子
apiVersion: runtime.cluster.x-k8s.io/v1alpha1
kind: ExtensionConfig
metadata:
  name: addon-installer
spec:
  clientConfig:
    service:
      name: addon-installer-service
      namespace: capi-system
      port: 443
  namespaceSelector: {}
```

### 需求 5: 证书体系自动轮转

**可行性**: ✅ 原生支持 + 扩展

**方案 A: Kubeadm 内置证书轮转**（控制面证书）
```yaml
kubeadmConfigSpec:
  clusterConfiguration:
    certificateValidityPeriodDays: 365  # 证书有效期
    caCertificateValidityPeriodDays: 3650  # CA证书有效期
```

**KubeadmControlPlane 自动轮转触发**:
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
spec:
  rolloutBefore:
    certificatesExpiryDays: 30  # 证书到期前30天触发轮转
```

**方案 B: 使用 cert-manager**（推荐用于完整证书管理）
```yaml
# 安装 cert-manager 作为 addon
addons:
  - name: cert-manager
    chart:
      repo: https://charts.jetstack.io
      name: cert-manager
      version: v1.14.0
    values:
      installCRDs: true
      prometheus:
        enabled: false

# 使用 Certificate CR 管理证书
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: apiserver-serving
  namespace: kube-system
spec:
  secretName: apiserver-serving-tls
  issuerRef:
    name: cluster-ca
    kind: ClusterIssuer
  dnsNames:
    - kubernetes.default.svc
    - "{{ .externalServices.externalLoadBalancer.endpoint }}"
  duration: 8760h  # 1 year
  renewBefore: 720h  # 30 days
```

**方案 C: 自定义证书轮转 Controller**
```go
// CertificateRotationReconciler
type CertificateRotationReconciler struct {
    client.Client
}

func (r *CertificateRotationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    kcp := &controlplanev1.KubeadmControlPlane{}
    r.Get(ctx, req.NamespacedName, kcp)
    
    // 检查证书到期时间
    certs := r.getCertificates(kcp)
    for _, cert := range certs {
        if cert.NotAfter.Sub(time.Now()) < 30*24*time.Hour {
            // 触发证书轮转
            // 1. 生成新证书
            // 2. 更新 Secret
            // 3. 触发 KCP rollout（通过 annotations）
            kcp.Annotations["controlplane.cluster.x-k8s.io/trigger-rollout"] = time.Now().Format(time.RFC3339)
            r.Update(ctx, kcp)
        }
    }
    
    return ctrl.Result{RequeueAfter: 24 * time.Hour}, nil
}
```

## 三、完整架构组件清单

| 组件 | 类型 | 说明 |
|------|------|------|
| **CAPBM** | Infrastructure Provider | 裸金属机器管理，SSH连接 |
| **Kubeadm Bootstrap** | Bootstrap Provider | 内置，无需开发 |
| **KubeadmControlPlane** | Control Plane Provider | 内置，无需开发 |
| **Addon Manager** | Runtime Extension | 扩展组件安装与依赖管理 |
| **ClusterClass** | 模板 | 集群拓扑定义 |
| **Cert Manager** | 可选组件 | 证书自动轮转 |

## 四、关键实现文件结构
```
capbm/
├── api/
│   └── v1beta2/
│       ├── baremetalmachine_types.go      # 裸金属机器CRD
│       ├── baremetalcluster_types.go      # 裸金属集群CRD
│       └── baremetalmachinetemplate_types.go
├── controllers/
│   ├── baremetalmachine_controller.go     # 机器生命周期管理
│   ├── baremetalcluster_controller.go     # 集群网络/LB管理
│   └── helpers/
│       ├── ssh_client.go                  # SSH连接管理
│       └── ipmi_client.go                 # IPMI电源管理（可选）
├── internal/
│   └── provisioning/
│       ├── kubeadm_init.go                # kubeadm init执行
│       └── kubeadm_join.go                # kubeadm join执行
└── main.go

addon-manager/
├── controllers/
│   ├── addon_topology_reconciler.go       # 依赖拓扑安装
│   └── addon_status_reconciler.go         # 状态监控
├── runtime/
│   └── hooks/
│       ├── lifecycle_hooks.go             # 生命周期钩子
│       └── topology_patches.go            # 拓扑补丁
└── main.go
```

## 五、风险与挑战
| 风险项 | 影响 | 缓解措施 |
|--------|------|----------|
| 裸金属机器状态不可控 | 高 | 集成 IPMI/Redfish 电源管理 |
| etcd 分离部署复杂性 | 中 | 优先使用 stacked etcd，分离部署需充分测试 |
| 证书轮转期间服务中断 | 高 | 使用滚动更新策略，确保 APIServer 可用性 |
| 网络配置复杂性 | 中 | 提供完善的网络预检脚本 |

## 六、推荐实施路径
1. **Phase 1**: 开发 CAPBM Infrastructure Provider（基础机器管理）
2. **Phase 2**: 定义 ClusterClass 模板（kubeadm 配置、外部服务集成）
3. **Phase 3**: 开发 Addon Manager Runtime Extension（组件安装）
4. **Phase 4**: 实现证书轮转机制（cert-manager 集成）
5. **Phase 5**: 完善拓扑化安装与依赖管理
6. **Phase 6**: 端到端测试与优化
