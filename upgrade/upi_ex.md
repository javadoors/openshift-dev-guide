# ComponentVersion 多安装方式重构的完整设计方案
基于对 OpenShift CVO 架构、ClusterVersion/ClusterOperator 模型以及 ComponentVersion/NodeConfig 概念的研究，以下是 **ComponentVersion 多安装方式重构**的完整设计方案。
## 一、原始 ComponentVersion 模型分析
参考 OpenShift CVO 的设计，原始 ComponentVersion 的核心职责是：

| 属性 | 说明 |
|------|------|
| `name` | 组件名称（如 etcd、apiserver、dns） |
| `version` | 组件当前版本 |
| `image` | 组件容器镜像 |
| `manifests` | 组件部署清单（CVO 通过 release payload 中的 manifest 部署） |

**原始模型的局限性：**
- 仅支持 manifest（k8s yaml）一种安装方式
- 无法表达脚本安装（如 kubeadm init、containerd 配置）
- 无法表达 Helm chart 安装
- 安装逻辑与版本信息耦合
## 二、重构后的 ComponentVersion CRD 设计
### 2.1 核心设计原则
1. **安装方式与版本信息解耦**：`spec.install` 独立于 `spec.version`，同一组件可切换安装方式
2. **策略模式（Strategy Pattern）**：通过 `install.type` 字段选择安装策略，各策略有独立的参数和状态
3. **状态追踪统一**：无论哪种安装方式，status 结构统一，便于上层编排器（CVO）聚合
4. **可扩展性**：通过 `type` 枚举支持未来新增安装方式
### 2.2 CRD 完整定义
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: componentversions.orchestration.bke.io
spec:
  group: orchestration.bke.io
  scope: Cluster
  names:
    plural: componentversions
    singular: componentversion
    kind: ComponentVersion
    shortNames: [cv]
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [componentName, version, install]
            properties:
              componentName:
                type: string
                description: "组件名称，如 etcd, kube-apiserver, coredns"
              version:
                type: string
                description: "组件目标版本，语义化版本号"
              scope:
                type: string
                enum: [Cluster, Node]
                description: "组件作用域：集群级或节点级"
              dependencies:
                type: array
                description: "前置依赖组件，DAG 调度依据"
                items:
                  type: object
                  required: [name]
                  properties:
                    name:
                      type: string
                    minVersion:
                      type: string
              install:
                type: object
                required: [type]
                properties:
                  type:
                    type: string
                    enum: [Manifest, InstallScript, HelmChart]
                    description: "安装策略类型"
                  manifest:
                    type: object
                    description: "Manifest 安装策略参数，type=Manifest 时必填"
                    properties:
                      manifests:
                        type: array
                        items:
                          type: object
                          required: [name, source]
                          properties:
                            name:
                              type: string
                              description: "manifest 资源名称标识"
                            source:
                              type: object
                              required: [type]
                              properties:
                                type:
                                  type: string
                                  enum: [Inline, ConfigMapRef, URIRref]
                                inline:
                                  type: string
                                  format: byte
                                  description: "内联 YAML 内容（base64）"
                                configMapRef:
                                  type: object
                                  required: [name, namespace]
                                  properties:
                                    name:
                                      type: string
                                    namespace:
                                      type: string
                                    key:
                                      type: string
                                      description: "ConfigMap 中存储 manifest 的 key"
                                uriRef:
                                  type: object
                                  required: [uri]
                                  properties:
                                    uri:
                                      type: string
                                      description: "远程 manifest URI，如 https://..."
                                    checksum:
                                      type: string
                                      description: "SHA256 校验和"
                            namespace:
                              type: string
                              description: "部署目标命名空间"
                      applyStrategy:
                        type: string
                        enum: [ServerSideApply, ThreeWayMerge, Replace]
                        default: ServerSideApply
                        description: "manifest 应用策略"
                      prune:
                        type: boolean
                        default: false
                        description: "是否清理不再存在的资源"
                      ownership:
                        type: object
                        properties:
                          labelKey:
                            type: string
                            default: "orchestration.bke.io/managed-by"
                          labelValue:
                            type: string
                  installScript:
                    type: object
                    description: "InstallScript 安装策略参数，type=InstallScript 时必填"
                    properties:
                      entryPoint:
                        type: string
                        description: "入口脚本命令，如 'kubeadm init', 'systemctl restart containerd'"
                      args:
                        type: array
                        items:
                          type: string
                      env:
                        type: array
                        items:
                          type: object
                          required: [name, value]
                          properties:
                            name:
                              type: string
                            value:
                              type: string
                            valueFrom:
                              type: object
                              properties:
                                secretKeyRef:
                                  type: object
                                  required: [name, key]
                                  properties:
                                    name: { type: string }
                                    key: { type: string }
                                    namespace: { type: string }
                      source:
                        type: object
                        required: [type]
                        properties:
                          type:
                            type: string
                            enum: [Inline, ConfigMapRef, ImageRef]
                          inline:
                            type: string
                            format: byte
                            description: "内联脚本内容（base64）"
                          configMapRef:
                            type: object
                            required: [name, namespace, key]
                            properties:
                              name: { type: string }
                              namespace: { type: string }
                              key: { type: string }
                          imageRef:
                            type: object
                            required: [image]
                            properties:
                              image:
                                type: string
                                description: "包含脚本的容器镜像，如 quay.io/bke/scripts:v1"
                              path:
                                type: string
                                description: "脚本在镜像中的路径，如 /scripts/install.sh"
                              imagePullSecret:
                                type: string
                      execution:
                        type: object
                        properties:
                          target:
                            type: string
                            enum: [Controller, NodeAgent, InitPod]
                            default: NodeAgent
                            description: "执行位置：控制器内 / 节点代理 / 初始化Pod"
                          nodeSelector:
                            type: object
                            additionalProperties:
                              type: string
                            description: "节点选择器，scope=Node 时使用"
                          timeout:
                            type: string
                            default: "300s"
                            description: "执行超时时间"
                          retryPolicy:
                            type: object
                            properties:
                              maxRetries:
                                type: integer
                                default: 3
                              retryDelay:
                                type: string
                                default: "10s"
                      validation:
                        type: object
                        description: "安装后验证"
                        properties:
                          command:
                            type: string
                            description: "验证命令，如 'etcdctl endpoint health'"
                          expectedOutput:
                            type: string
                            description: "期望输出包含的字符串"
                          timeout:
                            type: string
                            default: "60s"
                  helmChart:
                    type: object
                    description: "HelmChart 安装策略参数，type=HelmChart 时必填"
                    properties:
                      chartRef:
                        type: object
                        required: [name]
                        properties:
                          name:
                            type: string
                            description: "Chart 名称，如 coredns"
                          repo:
                            type: string
                            description: "Chart 仓库 URL"
                          version:
                            type: string
                            description: "Chart 版本"
                          chartPath:
                            type: string
                            description: "本地 Chart 路径（离线场景）"
                      releaseName:
                        type: string
                        description: "Helm Release 名称"
                      namespace:
                        type: string
                        description: "部署目标命名空间"
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
                        description: "Helm values 覆盖"
                      valuesFrom:
                        type: array
                        items:
                          type: object
                          required: [kind, name]
                          properties:
                            kind:
                              type: string
                              enum: [ConfigMap, Secret]
                            name:
                              type: string
                            namespace:
                              type: string
                            valuesKey:
                              type: string
                              description: "存储 values 的 key"
                      timeout:
                        type: string
                        default: "600s"
                      wait:
                        type: boolean
                        default: true
                        description: "是否等待所有 Pod 就绪"
                      atomic:
                        type: boolean
                        default: false
                        description: "失败时自动回滚"
              images:
                type: array
                description: "组件关联的容器镜像列表"
                items:
                  type: object
                  required: [name, image]
                  properties:
                    name:
                      type: string
                      description: "镜像用途标识，如 'etcd', 'setup'"
                    image:
                      type: string
                      description: "完整镜像地址，含 tag/digest"
              healthCheck:
                type: object
                description: "组件健康检查定义"
                properties:
                  type:
                    type: string
                    enum: [HTTPGet, TCPSocket, Command, ClusterOperator]
                  httpGet:
                    type: object
                    properties:
                      path: { type: string }
                      port: { type: integer }
                      scheme: { type: string, default: "HTTPS" }
                  tcpSocket:
                    type: object
                    properties:
                      port: { type: integer }
                  command:
                    type: object
                    properties:
                      command:
                        type: array
                        items:
                          type: string
                  clusterOperator:
                    type: object
                    properties:
                      name:
                        type: string
                      conditionType:
                        type: string
                        default: "Available"
                  interval:
                    type: string
                    default: "30s"
                  timeout:
                    type: string
                    default: "10s"
          status:
            type: object
            properties:
              observedVersion:
                type: string
                description: "实际观测到的版本"
              phase:
                type: string
                enum: [Pending, Installing, Verifying, Installed, Failed, Upgrading, RollingBack]
                description: "组件安装阶段"
              installType:
                type: string
                enum: [Manifest, InstallScript, HelmChart]
                description: "当前使用的安装类型"
              conditions:
                type: array
                items:
                  type: object
                  required: [type, status]
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                      enum: [True, False, Unknown]
                    reason:
                      type: string
                    message:
                      type: string
                    lastTransitionTime:
                      type: string
                      format: date-time
              installDetails:
                type: object
                description: "安装方式特定的状态详情"
                properties:
                  manifest:
                    type: object
                    properties:
                      appliedResources:
                        type: array
                        items:
                          type: object
                          properties:
                            group: { type: string }
                            version: { type: string }
                            kind: { type: string }
                            name: { type: string }
                            namespace: { type: string }
                            uid: { type: string }
                            hash: { type: string }
                            appliedAt: { type: string, format: date-time }
                      prunedResources:
                        type: array
                        items:
                          type: object
                          properties:
                            group: { type: string }
                            version: { type: string }
                            kind: { type: string }
                            name: { type: string }
                            namespace: { type: string }
                  installScript:
                    type: object
                    properties:
                      executionId:
                        type: string
                        description: "执行 ID，用于追踪命令状态"
                      targetNodes:
                        type: array
                        items:
                          type: object
                          properties:
                            nodeName: { type: string }
                            nodeIP: { type: string }
                            phase:
                              type: string
                              enum: [Pending, Running, Succeeded, Failed, Timeout]
                            exitCode: { type: integer }
                            stdout: { type: string }
                            stderr: { type: string }
                            startedAt: { type: string, format: date-time }
                            finishedAt: { type: string, format: date-time }
                            retryCount: { type: integer }
                      validationResult:
                        type: object
                        properties:
                          passed: { type: boolean }
                          output: { type: string }
                          checkedAt: { type: string, format: date-time }
                  helmChart:
                    type: object
                    properties:
                      releaseStatus:
                        type: string
                        description: "Helm Release 状态：deployed, failed, pending-install..."
                      revision:
                        type: integer
                      lastDeployed:
                        type: string
                        format: date-time
              lastTransitionTime:
                type: string
                format: date-time
              retryCount:
                type: integer
```
## 三、三种安装策略的详细设计
### 3.1 Manifest 策略（对标 OpenShift CVO 原生模式）
**设计思路：** 直接对标 OpenShift CVO 的 manifest 部署模式。CVO 从 release payload 中提取 manifest 并通过 `kubectl apply` 语义应用到集群。
```
┌─────────────────────────────────────────────────────────┐
│                  Manifest 安装流程                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ComponentVersion CR                                    │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────────┐                                       │
│  │ Source 解析  │  Inline / ConfigMap / URI             │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Manifest 渲染│  模板变量替换（版本、镜像等）         │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Apply 策略   │  ServerSideApply/3WayMerge/Replace    │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Ownership    │  标签标记 + 资源追踪                  │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Prune        │  清理不再存在的旧资源                 │
│  └──────────────┘                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```
**关键设计点：**

| 设计点 | 说明 |
|--------|------|
| **Source 多源支持** | Inline（适合小 manifest）、ConfigMapRef（适合动态更新）、URIRef（适合远程拉取） |
| **ApplyStrategy** | ServerSideApply 为默认，避免客户端合并冲突；ThreeWayMerge 兼容旧模式；Replace 用于 CRD 等不可 merge 场景 |
| **Ownership** | 通过 label 标记管理关系，支持 Prune 时精确识别 |
| **资源追踪** | status 中记录所有 appliedResources 的 GVK + hash，用于变更检测和回滚 |
### 3.2 InstallScript 策略
**设计思路：** 适配需要通过脚本执行的组件安装，如 kubeadm 初始化、containerd 配置、系统级包安装等。这是对 OpenShift MCO（Machine Config Operator）中 MachineConfigDaemon 执行脚本的抽象。
```
┌─────────────────────────────────────────────────────────┐
│               InstallScript 安装流程                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ComponentVersion CR                                    │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────────┐                                       │
│  │ Source 解析   │  Inline / ConfigMap / Image          │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ 执行目标选择  │  Controller / NodeAgent / InitPod    │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────────────────────────────┐               │
│  │         执行分发                      │              │
│  │  ┌─────────┐  ┌─────────┐           │                │
│  │  │Controller│  │NodeAgent│  │InitPod│                │
│  │  │ 直接执行 │  │下发命令 │  │Job/Pod│                │
│  │  └─────────┘  └─────────┘  └───────┘                 │
│  └──────────────────────────────────────┘               │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ 执行 + 重试   │  超时控制 + 重试策略                 │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ 验证         │  安装后执行验证命令                   │
│  └──────────────┘                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```
**三种执行目标对比：**

| 执行目标 | 适用场景 | 通信方式 | 生命周期 |
|----------|----------|----------|----------|
| **Controller** | 集群级操作（如创建 CRD、RBAC） | 直接调用 k8s API | 与 Controller 共存亡 |
| **NodeAgent** | 节点级操作（如 kubeadm、containerd、systemd） | 通过 Command CR 下发 | 异步执行，状态回传 |
| **InitPod** | 需要独立环境的操作（如数据迁移、复杂初始化） | 创建 Job/Pod | 一次性执行 |

**关键设计点：**

| 设计点 | 说明 |
|--------|------|
| **Source 三源支持** | Inline（短脚本）、ConfigMapRef（可动态更新）、ImageRef（复杂脚本打包为镜像） |
| **NodeSelector** | scope=Node 时，通过标签选择目标节点，支持滚动升级 |
| **Validation** | 安装后验证机制，确保脚本执行结果符合预期 |
| **状态回传** | NodeAgent 模式下，每个节点独立追踪执行状态（Pending→Running→Succeeded/Failed） |
### 3.3 HelmChart 策略
**设计思路：** 适配通过 Helm 管理的组件，如监控栈（Prometheus + Grafana）、日志采集（Fluent Bit）等。对标 OpenShift OLM（Operator Lifecycle Manager）的 Helm 集成模式。
```
┌─────────────────────────────────────────────────────────┐
│               HelmChart 安装流程                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ComponentVersion CR                                    │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────────┐                                       │
│  │ Chart 解析    │  repo URL / 本地路径 / OCI Registry  │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Values 合并   │  spec.values + valuesFrom → 最终值   │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Helm Install  │  helm upgrade --install              │
│  │  or Upgrade   │  --version <chartVersion>            │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Wait + Check │  --wait --timeout                     │
│  └──────┬───────┘                                       │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Atomic 回滚   │  失败时自动回滚（可选）              │
│  └──────────────┘                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```
**关键设计点：**

| 设计点 | 说明 |
|--------|------|
| **Chart 多源** | 支持远程仓库、本地路径、OCI Registry |
| **Values 分层** | spec.values（基础值）+ valuesFrom（ConfigMap/Secret 引用），后者覆盖前者 |
| **Atomic 模式** | 开启后 Helm 会在失败时自动回滚，确保集群状态一致 |
| **Release 追踪** | status 中记录 Helm Release 状态、revision、部署时间 |
## 四、NodeConfig CRD 设计
NodeConfig 与 ComponentVersion 配合，管理节点级组件的配置和状态：
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: nodeconfigs.orchestration.bke.io
spec:
  group: orchestration.bke.io
  scope: Cluster
  names:
    plural: nodeconfigs
    singular: nodeconfig
    kind: NodeConfig
    shortNames: [nc]
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [nodeName]
            properties:
              nodeName:
                type: string
              nodeIP:
                type: string
              roles:
                type: array
                items:
                  type: string
                  enum: [control-plane, worker, etcd]
              osInfo:
                type: object
                properties:
                  type: { type: string }
                  version: { type: string }
                  arch: { type: string }
              componentConfigs:
                type: array
                description: "节点级组件配置覆盖"
                items:
                  type: object
                  required: [componentName]
                  properties:
                    componentName:
                      type: string
                    configOverrides:
                      type: object
                      x-kubernetes-preserve-unknown-fields: true
                      description: "组件配置覆盖值"
                    nodeLabels:
                      type: object
                      additionalProperties:
                        type: string
                      description: "节点标签，用于 ComponentVersion 的 nodeSelector 匹配"
          status:
            type: object
            properties:
              componentVersions:
                type: array
                items:
                  type: object
                  properties:
                    componentName: { type: string }
                    currentVersion: { type: string }
                    targetVersion: { type: string }
                    phase:
                      type: string
                      enum: [Synced, OutOfSync, Updating, Failed]
                    lastUpdated: { type: string, format: date-time }
              nodeHealth:
                type: string
                enum: [Healthy, Degraded, Unreachable]
              lastHeartbeat:
                type: string
                format: date-time
```
## 五、Controller 协调逻辑
### 5.1 Reconcile 主流程
```
Reconcile(ComponentVersion)
    │
    ├─ 1. 解析 spec.install.type
    │
    ├─ 2. 根据 type 选择 Installer 策略
    │     ├─ ManifestInstaller
    │     ├─ InstallScriptInstaller
    │     └─ HelmChartInstaller
    │
    ├─ 3. 检查前置依赖 (spec.dependencies)
    │     └─ 查询依赖 ComponentVersion 的 status.phase == Installed
    │
    ├─ 4. 执行安装
    │     └─ installer.Install(ctx, cv)
    │
    ├─ 5. 执行验证
    │     └─ installer.Verify(ctx, cv)
    │
    ├─ 6. 更新 Status
    │     ├─ phase: Installing → Verifying → Installed / Failed
    │     └─ installDetails: 各策略特定状态
    │
    └─ 7. 健康检查
          └─ 根据 spec.healthCheck 持续检查
```
### 5.2 Installer 接口设计
```go
type Installer interface {
    Install(ctx context.Context, cv *ComponentVersion) error
    Verify(ctx context.Context, cv *ComponentVersion) (bool, error)
    Rollback(ctx context.Context, cv *ComponentVersion) error
    Status(ctx context.Context, cv *ComponentVersion) (*InstallStatus, error)
}

type InstallStatus struct {
    Phase       ComponentPhase
    Details     runtime.Object
    Conditions  []metav1.Condition
}
```
### 5.3 三种 Installer 实现
| Installer | Install 核心逻辑 | Verify 核心逻辑 | Rollback 核心逻辑 |
|-----------|-----------------|----------------|------------------|
| **ManifestInstaller** | 解析 Source → 渲染模板 → ApplyStrategy 应用 → 记录 appliedResources | 检查每个 appliedResource 的 status | 删除 appliedResources 或回滚到上一版本 hash |
| **InstallScriptInstaller** | 解析 Source → 选择执行目标 → 下发/执行脚本 → 追踪节点状态 | 执行 validation.command | 执行回滚脚本或重试 |
| **HelmChartInstaller** | 解析 ChartRef → 合并 Values → helm upgrade --install | 检查 Helm Release 状态 + --wait | helm rollback |
## 六、三种安装策略的典型使用场景
### 6.1 Manifest 方式 — CoreDNS 部署
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: ComponentVersion
metadata:
  name: coredns
spec:
  componentName: coredns
  version: "1.10.1"
  scope: Cluster
  install:
    type: Manifest
    manifest:
      manifests:
      - name: coredns-deployment
        source:
          type: ConfigMapRef
          configMapRef:
            name: coredns-manifests
            namespace: orchestration-system
            key: deployment.yaml
        namespace: kube-system
      - name: coredns-service
        source:
          type: ConfigMapRef
          configMapRef:
            name: coredns-manifests
            namespace: orchestration-system
            key: service.yaml
        namespace: kube-system
      applyStrategy: ServerSideApply
      prune: true
  images:
  - name: coredns
    image: registry.k8s.io/coredns/coredns:v1.10.1
  healthCheck:
    type: Command
    command:
      command: ["kubectl", "get", "pods", "-n", "kube-system", "-l", "k8s-app=kube-dns"]
    interval: 30s
```
### 6.2 InstallScript 方式 — Containerd 升级
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: ComponentVersion
metadata:
  name: containerd
spec:
  componentName: containerd
  version: "1.7.2"
  scope: Node
  dependencies:
  - name: kubelet
    minVersion: "1.27.0"
  install:
    type: InstallScript
    installScript:
      entryPoint: "/scripts/upgrade-containerd.sh"
      args: ["--version", "1.7.2", "--config", "/etc/containerd/config.toml"]
      env:
      - name: CONTAINERD_VERSION
        value: "1.7.2"
      source:
        type: ImageRef
        imageRef:
          image: quay.io/bke/scripts:v1.2.0
          path: /scripts/upgrade-containerd.sh
      execution:
        target: NodeAgent
        nodeSelector:
          node-role.kubernetes.io/worker: ""
        timeout: "600s"
        retryPolicy:
          maxRetries: 3
          retryDelay: "30s"
      validation:
        command: "containerd --version"
        expectedOutput: "1.7.2"
        timeout: "30s"
  healthCheck:
    type: Command
    command:
      command: ["systemctl", "is-active", "containerd"]
    interval: 60s
```
### 6.3 HelmChart 方式 — 监控栈部署
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: ComponentVersion
metadata:
  name: monitoring-stack
spec:
  componentName: monitoring
  version: "45.0.0"
  scope: Cluster
  dependencies:
  - name: kube-apiserver
    minVersion: "1.26.0"
  install:
    type: HelmChart
    helmChart:
      chartRef:
        name: kube-prometheus-stack
        repo: https://prometheus-community.github.io/helm-charts
        version: "45.0.0"
      releaseName: monitoring
      namespace: monitoring
      values:
        prometheus:
          prometheusSpec:
            retention: 15d
            resources:
              requests:
                cpu: 500m
                memory: 2Gi
        grafana:
          enabled: true
          adminPassword: admin
      valuesFrom:
      - kind: Secret
        name: monitoring-values
        namespace: monitoring
        valuesKey: overrides.yaml
      timeout: "900s"
      wait: true
      atomic: true
  healthCheck:
    type: HTTPGet
    httpGet:
      path: /-/healthy
      port: 9090
      scheme: HTTP
    interval: 30s
```
## 七、状态机设计
三种安装策略共享统一的状态机：
```
                    ┌──────────┐
                    │ Pending  │ ← 初始状态/依赖未满足
                    └────┬─────┘
                         │ 依赖满足
                         ▼
                    ┌──────────┐
          ┌────────│Installing │────────────┐
          │        └────┬─────┘            │
          │             │                  │
          │    安装完成  │     安装失败     │
          │             ▼                  ▼
          │       ┌──────────┐       ┌──────────┐
          │       │Verifying │       │  Failed  │
          │       └────┬─────┘       └────┬─────┘
          │            │                   │
          │   验证通过 │    验证失败       │ 手动触发
          │            ▼                   │
          │       ┌──────────┐             │
          │       │ Installed│◄────────────┘
          │       └────┬─────┘    回滚成功
          │            │
          │  版本变更   │
          │            ▼
          │       ┌──────────┐
          └──────►│Upgrading │
                  └────┬─────┘
                       │ 升级失败
                       ▼
                  ┌──────────┐
                  │RollingBack│
                  └────┬─────┘
                       │ 回滚完成
                       ▼
                  ┌──────────┐
                  │ Installed│ (回滚到旧版本)
                  └──────────┘
```
## 八、与 NodeConfig 的协作关系
```
┌─────────────────────────────────────────────────────────────┐
│                    编排层协作关系                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ComponentVersion (集群级定义)                               │
│  ┌─────────────────────────────────────────┐                │
│  │ spec.componentName: containerd          │                │
│  │ spec.version: 1.7.2                     │                │
│  │ spec.scope: Node                        │                │
│  │ spec.install.type: InstallScript        │                │
│  │ spec.install.installScript.execution:   │                │
│  │   target: NodeAgent                     │                │
│  │   nodeSelector:                         │                │
│  │     node-role: worker                   │──────┐         │
│  └─────────────────────────────────────────┘      │         │
│                                                    │ 匹配    │
│  NodeConfig (节点级配置)                           │         │
│  ┌─────────────────────────────────────────┐      │         │
│  │ spec.nodeName: worker-01                │      │         │
│  │ spec.roles: [worker]                    │◄─────┘         │
│  │ spec.componentConfigs:                  │                │
│  │ - componentName: containerd             │                │
│  │   configOverrides:                      │                │
│  │     configPath: /etc/containerd/custom  │                │
│  │   nodeLabels:                           │                │
│  │     node-role: worker                   │                │
│  └─────────────────────────────────────────┘                │
│                                                             │
│  执行时：ComponentVersion 的 nodeSelector 匹配 NodeConfig   │
│  的 nodeLabels，确定目标节点列表                             │
│  NodeConfig 的 configOverrides 提供节点级配置覆盖            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
## 九、设计总结
### 9.1 三种安装策略对比
| 维度 | Manifest | InstallScript | HelmChart |
|------|----------|---------------|-----------|
| **适用组件** | k8s 原生资源（Deployment、Service、CRD） | 系统级组件（kubeadm、containerd、systemd） | 复杂应用栈（监控、日志、Ingress） |
| **执行位置** | Controller → API Server | Controller / NodeAgent / InitPod | Controller → Helm SDK |
| **版本管理** | manifest hash + GVK 追踪 | 脚本执行 ID + 节点状态 | Helm Release revision |
| **回滚能力** | 删除/替换 appliedResources | 重试/执行回滚脚本 | helm rollback |
| **离线支持** | ConfigMap 内嵌 | ImageRef 打包 | chartPath 本地路径 |
| **状态粒度** | 资源级别 | 节点级别 | Release 级别 |
### 9.2 关键设计决策
1. **`spec.install` 下三种策略互斥**：通过 `type` 字段选择，同一 ComponentVersion 只能使用一种安装方式，避免混合导致状态混乱
2. **`status.installDetails` 分层记录**：各策略有独立的状态结构，但共享顶层 phase/conditions，便于上层编排器统一消费
3. **Source 多源设计**：Inline/ConfigMap/URI/Image 四种来源覆盖在线、离线、动态更新场景
4. **NodeConfig 解耦节点级配置**：ComponentVersion 定义"安装什么"，NodeConfig 定义"在哪些节点、用什么配置"，两者通过 nodeSelector/nodeLabels 关联
5. **HealthCheck 独立于安装策略**：无论哪种安装方式，健康检查定义统一，确保组件可用性监控的一致性
        
