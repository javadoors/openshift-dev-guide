
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
