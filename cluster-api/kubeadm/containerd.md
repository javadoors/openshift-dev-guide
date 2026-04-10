
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

    
# 基于Cluster API安装Kubernetes集群与Containerd的设计方案
## 一、问题分析
### 核心矛盾
- **kubeadm职责边界**：kubeadm只负责Kubernetes集群的初始化和升级，不负责容器运行时的安装
- **Cluster API架构**：Cluster API通过多个Provider协作完成集群生命周期管理
- **容器运行时依赖**：Kubernetes节点必须先安装并配置好容器运行时，才能运行kubelet和容器
### 需要解决的问题
1. **时序问题**：containerd必须在kubelet启动前安装并运行
2. **配置问题**：containerd需要正确配置才能与Kubernetes集成
3. **版本兼容**：containerd版本需要与Kubernetes版本兼容
4. **生命周期管理**：containerd的升级、配置变更需要与集群升级协调
## 二、设计思路
### 方案选择：Infrastructure Provider扩展方案
我推荐使用**Infrastructure Provider扩展方案**，原因如下：
1. **职责清晰**：Infrastructure Provider负责节点的基础设施准备，containerd属于节点基础环境
2. **时序保证**：Infrastructure Provider在Bootstrap Provider之前执行，确保containerd先安装
3. **架构符合**：符合Cluster API的分层架构设计
4. **通用性强**：不依赖特定云平台的cloud-init功能
### 架构设计
```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster API Core                         │
│  (Cluster, Machine, KubeadmControlPlane, KubeadmConfig)    │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Infrastructure│   │   Bootstrap   │   │ Control Plane │
│   Provider    │   │   Provider    │   │   Provider    │
│               │   │   (Kubeadm)   │   │   (Kubeadm)   │
└───────────────┘   └───────────────┘   └───────────────┘
        │
        ├─ 创建节点资源（VM/物理机）
        ├─ 安装容器运行时
        ├─ 配置系统环境
        └─ 返回节点连接信息
```
## 三、详细设计方案
### 3.1 CRD设计
#### ContainerRuntimeConfig CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: containerruntimeconfigs.infrastructure.cluster.x-k8s.io
spec:
  group: infrastructure.cluster.x-k8s.io
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              type:
                type: string
                enum: [containerd, docker, cri-o]
                default: containerd
              version:
                type: string
                description: "Container runtime version, e.g., 1.7.0"
              config:
                type: object
                properties:
                  registryMirrors:
                    type: array
                    items:
                      type: string
                  insecureRegistries:
                    type: array
                    items:
                      type: string
                  sandboxImage:
                    type: string
                  systemCgroup:
                    type: boolean
                    default: true
              installConfig:
                type: object
                properties:
                  packageManager:
                    type: string
                    enum: [apt, yum, dnf, binary]
                  binaryDownloadURL:
                    type: string
                  preInstallScript:
                    type: string
                  postInstallScript:
                    type: string
          status:
            type: object
            properties:
              ready:
                type: boolean
              installedVersion:
                type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: string
                    status: string
                    reason: string
                    message: string
```
#### 扩展InfrastructureMachine CRD
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: GenericMachine
metadata:
  name: worker-node-1
spec:
  containerRuntimeRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: ContainerRuntimeConfig
    name: containerd-config
  providerSpec:
    # 云平台特定的配置
    instanceType: t3.medium
    imageId: ami-12345678
    # ... 其他基础设施配置
```
### 3.2 控制器设计
#### ContainerRuntimeConfig控制器
**职责**：
- 管理容器运行时配置的生命周期
- 验证配置的有效性
- 生成安装脚本

**核心逻辑**：
```go
type ContainerRuntimeConfigReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *ContainerRuntimeConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    config := &infrastructurev1beta1.ContainerRuntimeConfig{}
    if err := r.Get(ctx, req.NamespacedName, config); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 验证配置
    if err := r.validateConfig(config); err != nil {
        conditions.MarkFalse(config, "Valid", "InvalidConfig", err.Error())
        return ctrl.Result{}, r.Status().Update(ctx, config)
    }

    // 生成安装脚本
    installScript, err := r.generateInstallScript(config)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 存储安装脚本到Secret
    if err := r.storeInstallScript(ctx, config, installScript); err != nil {
        return ctrl.Result{}, err
    }

    conditions.MarkTrue(config, "Ready")
    config.Status.Ready = true
    return ctrl.Result{}, r.Status().Update(ctx, config)
}

func (r *ContainerRuntimeConfigReconciler) generateInstallScript(config *infrastructurev1beta1.ContainerRuntimeConfig) (string, error) {
    var script strings.Builder
    
    // 添加前置脚本
    if config.Spec.InstallConfig.PreInstallScript != "" {
        script.WriteString(config.Spec.InstallConfig.PreInstallScript)
        script.WriteString("\n")
    }

    // 根据包管理器生成安装命令
    switch config.Spec.InstallConfig.PackageManager {
    case "apt":
        script.WriteString(r.generateAptInstallScript(config))
    case "yum", "dnf":
        script.WriteString(r.generateYumInstallScript(config))
    case "binary":
        script.WriteString(r.generateBinaryInstallScript(config))
    }

    // 生成配置文件
    script.WriteString(r.generateConfigScript(config))

    // 启动服务
    script.WriteString("systemctl enable containerd\n")
    script.WriteString("systemctl start containerd\n")

    // 添加后置脚本
    if config.Spec.InstallConfig.PostInstallScript != "" {
        script.WriteString(config.Spec.InstallConfig.PostInstallScript)
        script.WriteString("\n")
    }

    return script.String(), nil
}
```
#### GenericMachine控制器
**职责**：
- 创建基础设施资源（VM/物理机）
- 注入容器运行时安装脚本
- 等待节点就绪

**核心逻辑**：
```go
func (r *GenericMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    machine := &infrastructurev1beta1.GenericMachine{}
    if err := r.Get(ctx, req.NamespacedName, machine); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 获取关联的Cluster和Machine
    cluster, err := r.getOwnerCluster(ctx, machine)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 获取容器运行时配置
    runtimeConfig, err := r.getContainerRuntimeConfig(ctx, machine)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 生成userdata（包含containerd安装）
    userData, err := r.generateUserData(ctx, machine, runtimeConfig)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 创建基础设施实例
    instanceID, err := r.createInstance(ctx, machine, cluster, userData)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 等待实例就绪
    if !r.isInstanceReady(ctx, instanceID) {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }

    // 更新状态
    machine.Status.Ready = true
    machine.Status.ProviderID = fmt.Sprintf("generic://%s", instanceID)
    return ctrl.Result{}, r.Status().Update(ctx, machine)
}

func (r *GenericMachineReconciler) generateUserData(ctx context.Context, 
    machine *infrastructurev1beta1.GenericMachine,
    runtimeConfig *infrastructurev1beta1.ContainerRuntimeConfig) (string, error) {
    
    // 获取安装脚本
    installScript, err := r.getInstallScript(ctx, runtimeConfig)
    if err != nil {
        return "", err
    }

    // 构建cloud-init userdata
    userData := fmt.Sprintf(`#!/bin/bash
set -e

# 安装容器运行时
%s

# 等待containerd就绪
while ! systemctl is-active --quiet containerd; do
    sleep 1
done

# 标记节点准备就绪
echo "Container runtime installed and ready"
`, installScript)

    return userData, nil
}
```
### 3.3 安装脚本生成策略
#### APT包管理器安装脚本
```bash
#!/bin/bash
set -e

# 安装依赖
apt-get update
apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加Docker官方GPG密钥
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 设置仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装containerd
apt-get update
apt-get install -y containerd.io={{ .Version }}
```

#### 二进制安装脚本

```bash
#!/bin/bash
set -e

CONTAINERD_VERSION="{{ .Version }}"
DOWNLOAD_URL="{{ .BinaryDownloadURL }}"

# 下载二进制文件
curl -fsSL "${DOWNLOAD_URL}" -o containerd.tar.gz
tar -xzf containerd.tar.gz -C /usr/local

# 创建systemd服务
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
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```
#### 配置文件生成脚本
```bash
#!/bin/bash
set -e

# 创建配置目录
mkdir -p /etc/containerd

# 生成默认配置
containerd config default > /etc/containerd/config.toml

# 自定义配置
cat > /etc/containerd/config.toml <<EOF
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "{{ .SandboxImage }}"
  
  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = {{ .SystemCgroup }}

[plugins."io.containerd.grpc.v1.cri".registry]
  {{- if .RegistryMirrors }}
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = [{{ range .RegistryMirrors }}"{{ . }}",{{ end }}]
  {{- end }}
  
  {{- if .InsecureRegistries }}
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    {{- range .InsecureRegistries }}
    [plugins."io.containerd.grpc.v1.cri".registry.configs."{{ . }}".tls]
      insecure_skip_verify = true
    {{- end }}
  {{- end }}
EOF
```
## 四、实现流程
### 4.1 集群创建流程
```
用户创建Cluster资源
        │
        ▼
┌───────────────────────────────────┐
│ 1. 创建ContainerRuntimeConfig     │
│    - 定义containerd版本和配置      │
│    - 控制器生成安装脚本            │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 2. 创建GenericMachine资源         │
│    - 引用ContainerRuntimeConfig   │
│    - 定义基础设施规格              │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 3. GenericMachine控制器执行        │
│    - 获取容器运行时配置            │
│    - 生成包含安装脚本的userdata    │
│    - 创建基础设施实例（VM）        │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 4. 节点启动执行userdata           │
│    - 安装containerd               │
│    - 配置containerd               │
│    - 启动containerd服务           │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│ 5. Bootstrap Provider执行         │
│    - kubeadm初始化节点            │
│    - kubelet连接到containerd      │
│    - 节点加入集群                  │
└───────────────────────────────────┘
        │
        ▼
    集群创建完成
```
### 4.2 完整示例
```yaml
# 1. 定义容器运行时配置
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: ContainerRuntimeConfig
metadata:
  name: containerd-config
  namespace: default
spec:
  type: containerd
  version: "1.7.2"
  config:
    registryMirrors:
      - https://mirror.gcr.io
      - https://registry.docker-cn.com
    sandboxImage: registry.k8s.io/pause:3.9
    systemCgroup: true
  installConfig:
    packageManager: apt
    preInstallScript: |
      apt-get update
      apt-get install -y curl wget
---
# 2. 创建Cluster
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    serviceDomain: "cluster.local"
    services:
      cidrBlocks: ["10.128.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: GenericCluster
    name: my-cluster
---
# 3. 创建GenericCluster（基础设施集群）
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: GenericCluster
metadata:
  name: my-cluster
  namespace: default
spec:
  controlPlaneEndpoint:
    host: "192.168.1.100"
    port: 6443
---
# 4. 创建KubeadmControlPlane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
  namespace: default
spec:
  replicas: 3
  version: "v1.28.0"
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: GenericMachine
      name: control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
    joinConfiguration:
      nodeRegistration:
        criSocket: unix:///var/run/containerd/containerd.sock
---
# 5. 创建MachineDeployment（工作节点）
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-cluster-workers
  namespace: default
spec:
  clusterName: my-cluster
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  template:
    spec:
      clusterName: my-cluster
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: worker-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: GenericMachine
        name: worker
---
# 6. KubeadmConfigTemplate
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: worker-config
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          criSocket: unix:///var/run/containerd/containerd.sock
          kubeletExtraArgs:
            cgroup-driver: systemd
```
## 五、关键设计要点
### 5.1 时序保证
**问题**：确保containerd在kubelet启动前安装完成

**解决方案**：
1. **Infrastructure Provider先执行**：在Machine的Infrastructure资源创建时安装containerd
2. **状态检查**：在userdata中添加containerd就绪检查
3. **Bootstrap Provider等待**：kubeadm会等待容器运行时socket可用

```bash
# 在userdata中添加就绪检查
while ! systemctl is-active --quiet containerd; do
    echo "Waiting for containerd to be ready..."
    sleep 2
done

while [ ! -S /var/run/containerd/containerd.sock ]; do
    echo "Waiting for containerd socket..."
    sleep 2
done
```
### 5.2 配置管理
**问题**：如何灵活配置containerd

**解决方案**：
1. **CRD定义配置**：通过ContainerRuntimeConfig CRD定义配置
2. **模板引擎**：使用Go template生成配置文件
3. **配置验证**：控制器验证配置的有效性
### 5.3 版本兼容性
**问题**：containerd版本与Kubernetes版本的兼容性

**解决方案**：
1. **版本矩阵**：维护containerd与Kubernetes的版本兼容矩阵
2. **Webhook验证**：通过ValidatingWebhook验证版本兼容性
3. **默认推荐**：提供推荐的版本组合

```go
var containerdK8sCompatibility = map[string][]string{
    "1.7.x": {"v1.27.x", "v1.28.x", "v1.29.x"},
    "1.6.x": {"v1.24.x", "v1.25.x", "v1.26.x"},
}
```
### 5.4 升级策略
**问题**：如何处理containerd的升级

**解决方案**：
1. **滚动升级**：通过MachineDeployment的滚动更新策略
2. **配置变更**：修改ContainerRuntimeConfig触发节点重建
3. **版本锁定**：支持指定containerd版本

```yaml
spec:
  version: "1.7.2"  # 修改版本触发升级
  upgradeStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```
### 5.5 错误处理
**问题**：安装失败如何处理

**解决方案**：
1. **重试机制**：控制器自动重试失败的安装
2. **状态报告**：通过Conditions报告详细错误信息
3. **回滚支持**：支持回滚到之前的配置版本

```go
// 错误重试逻辑
if err := r.installContainerRuntime(ctx, machine); err != nil {
    conditions.MarkFalse(machine, "ContainerRuntimeReady", "InstallFailed", err.Error())
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```
## 六、扩展性设计
### 6.1 支持多种容器运行时
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: ContainerRuntimeConfig
metadata:
  name: cri-o-config
spec:
  type: cri-o
  version: "1.28.0"
  config:
    # CRI-O特定配置
```
### 6.2 支持多种基础设施
```go
// 抽象基础设施接口
type InfrastructureProvider interface {
    CreateInstance(ctx context.Context, spec *InstanceSpec) (string, error)
    DeleteInstance(ctx context.Context, instanceID string) error
    GetInstanceStatus(ctx context.Context, instanceID string) (*InstanceStatus, error)
    InjectUserData(ctx context.Context, instanceID string, userData string) error
}

// AWS实现
type AWSProvider struct{}

// GCP实现
type GCPProvider struct{}

// vSphere实现
type VSphereProvider struct{}
```
### 6.3 支持自定义安装方式
```yaml
spec:
  installConfig:
    packageManager: binary
    binaryDownloadURL: "https://custom-repo/containerd-1.7.2.tar.gz"
    preInstallScript: |
      # 自定义前置脚本
      curl -fsSL https://custom-repo/ca.crt -o /etc/pki/ca.crt
    postInstallScript: |
      # 自定义后置脚本
      systemctl restart containerd
```
## 七、优势与限制
### 优势
1. **架构清晰**：符合Cluster API的分层架构
2. **职责明确**：Infrastructure Provider负责基础设施准备
3. **时序保证**：确保containerd在kubelet前安装
4. **配置灵活**：通过CRD灵活配置容器运行时
5. **可扩展性强**：支持多种容器运行时和基础设施
### 限制
1. **需要实现Infrastructure Provider**：需要开发GenericMachine控制器
2. **依赖cloud-init**：需要基础设施支持cloud-init或类似机制
3. **升级复杂**：containerd升级需要节点重建
## 八、总结
这个设计方案通过扩展Infrastructure Provider的方式，在节点创建阶段安装和配置containerd，确保容器运行时在kubelet启动前就绪。方案的核心优势是：

1. **符合Cluster API架构**：遵循Cluster API的分层设计理念
2. **时序保证**：通过Infrastructure Provider先执行的特性保证时序
3. **配置灵活**：通过CRD实现声明式配置管理
4. **可扩展**：支持多种容器运行时和基础设施平台

这个方案可以作为一个独立的Infrastructure Provider实现，也可以集成到现有的Provider中作为扩展功能。
