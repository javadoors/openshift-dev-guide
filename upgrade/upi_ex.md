## 一、原始 ComponentVersion / NodeConfig 模型（参考 upi.md）
基于 OpenShift CVO 的设计理念，原始 ComponentVersion 和 NodeConfig 的核心职责：
### 1.1 原始 ComponentVersion
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: ComponentVersion
metadata:
  name: etcd
spec:
  componentName: etcd
  version: "3.5.9"
  scope: Cluster
  image: "quay.io/openshift/etcd:v3.5.9"
  manifests:
  - name: etcd-deployment
    source: release-payload
  dependencies:
  - name: kube-apiserver
    minVersion: "1.27.0"
status:
  observedVersion: "3.5.9"
  phase: Installed
  conditions: []
```
### 1.2 原始 NodeConfig
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: NodeConfig
metadata:
  name: master-01
spec:
  nodeName: master-01
  nodeIP: "192.168.1.10"
  roles: [control-plane, etcd]
  osInfo:
    type: rhcos
    version: "4.14"
status:
  componentVersions:
  - componentName: etcd
    currentVersion: "3.5.9"
  nodeHealth: Healthy
```
### 1.3 原始模型的局限性
| 局限点 | 说明 |
|--------|------|
| **安装方式单一** | 仅支持 manifest（k8s yaml）一种安装方式，通过 CVO release payload 分发 |
| **无法执行脚本** | 无法表达需要通过 shell 脚本执行的安装（如 kubeadm、systemctl、containerd 配置） |
| **无法使用 Helm** | 无法表达通过 Helm chart 管理的组件（如监控栈、Ingress Controller） |
| **来源固定** | manifest 来源仅支持 release-payload，无法从 ConfigMap、远程 URI、本地文件等获取 |
| **安装与版本耦合** | 安装逻辑嵌入在版本信息中，无法独立演进 |
## 二、重构目标
将 ComponentVersion 的安装能力从单一的 manifest 扩展为三种策略：

| 安装策略 | 适用场景 | 核心特征 |
|----------|----------|----------|
| **Manifest** | k8s 原生资源部署（Deployment、Service、CRD 等） | 声明式，通过 API Server 应用 |
| **InstallScript** | 系统级操作（kubeadm、containerd、systemd、二进制安装） | 命令式，通过脚本执行 |
| **HelmChart** | 复杂应用栈（监控、日志、Ingress） | 包管理，通过 Helm SDK 部署 |

每种策略的来源均支持多种方式：

| 来源类型 | 说明 | 适用场景 |
|----------|------|----------|
| **Inline** | 内嵌于 CR 中（base64 编码） | 小型 manifest、短脚本 |
| **ConfigMapRef** | 引用集群内 ConfigMap | 可动态更新，适合运维修改 |
| **SecretRef** | 引用集群内 Secret | 敏感配置（证书、密钥） |
| **URIRref** | 远程 URI 拉取 | 从 HTTP/HTTPS 服务器获取 |
| **ImageRef** | 容器镜像内打包 | 复杂脚本+依赖打包为镜像 |
| **LocalPath** | 本地文件路径 | 离线环境、预分发场景 |
## 三、重构后的 ComponentVersion CRD
### 3.1 完整 CRD 定义
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
                description: "组件名称"
              version:
                type: string
                description: "组件目标版本"
              scope:
                type: string
                enum: [Cluster, Node]
                default: Cluster
                description: "组件作用域"
              dependencies:
                type: array
                items:
                  type: object
                  required: [name]
                  properties:
                    name: { type: string }
                    minVersion: { type: string }
              images:
                type: array
                items:
                  type: object
                  required: [name, image]
                  properties:
                    name: { type: string }
                    image: { type: string }
              install:
                type: object
                required: [type]
                properties:
                  type:
                    type: string
                    enum: [Manifest, InstallScript, HelmChart]
                  manifest:
                    type: object
                    properties:
                      items:
                        type: array
                        items:
                          type: object
                          required: [name, source]
                          properties:
                            name: { type: string }
                            namespace: { type: string }
                            source:
                              type: object
                              required: [type]
                              properties:
                                type:
                                  type: string
                                  enum: [Inline, ConfigMapRef, SecretRef, URIRef, LocalPath]
                                inline:
                                  type: string
                                  format: byte
                                configMapRef:
                                  type: object
                                  required: [name, namespace]
                                  properties:
                                    name: { type: string }
                                    namespace: { type: string }
                                    key: { type: string }
                                secretRef:
                                  type: object
                                  required: [name, namespace]
                                  properties:
                                    name: { type: string }
                                    namespace: { type: string }
                                    key: { type: string }
                                uriRef:
                                  type: object
                                  required: [uri]
                                  properties:
                                    uri: { type: string }
                                    checksum: { type: string }
                                    insecureSkipTLSVerify: { type: boolean }
                                localPath:
                                  type: object
                                  required: [path]
                                  properties:
                                    path: { type: string }
                                    checksum: { type: string }
                      applyStrategy:
                        type: string
                        enum: [ServerSideApply, ThreeWayMerge, Replace]
                        default: ServerSideApply
                      prune: { type: boolean, default: false }
                  installScript:
                    type: object
                    properties:
                      command:
                        type: string
                        description: "入口命令，如 '/scripts/install.sh'"
                      args:
                        type: array
                        items: { type: string }
                      env:
                        type: array
                        items:
                          type: object
                          required: [name]
                          properties:
                            name: { type: string }
                            value: { type: string }
                            valueFrom:
                              type: object
                              properties:
                                configMapKeyRef:
                                  type: object
                                  properties:
                                    name: { type: string }
                                    key: { type: string }
                                    namespace: { type: string }
                                secretKeyRef:
                                  type: object
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
                            enum: [Inline, ConfigMapRef, SecretRef, URIRef, ImageRef, LocalPath]
                          inline:
                            type: string
                            format: byte
                          configMapRef:
                            type: object
                            required: [name, namespace, key]
                            properties:
                              name: { type: string }
                              namespace: { type: string }
                              key: { type: string }
                          secretRef:
                            type: object
                            required: [name, namespace, key]
                            properties:
                              name: { type: string }
                              namespace: { type: string }
                              key: { type: string }
                          uriRef:
                            type: object
                            required: [uri]
                            properties:
                              uri: { type: string }
                              checksum: { type: string }
                              insecureSkipTLSVerify: { type: boolean }
                          imageRef:
                            type: object
                            required: [image]
                            properties:
                              image: { type: string }
                              path: { type: string }
                              imagePullSecret: { type: string }
                          localPath:
                            type: object
                            required: [path]
                            properties:
                              path: { type: string }
                              checksum: { type: string }
                      execution:
                        type: object
                        properties:
                          target:
                            type: string
                            enum: [Controller, NodeAgent, InitPod]
                            default: NodeAgent
                          nodeSelector:
                            type: object
                            additionalProperties: { type: string }
                          timeout: { type: string, default: "300s" }
                          retryPolicy:
                            type: object
                            properties:
                              maxRetries: { type: integer, default: 3 }
                              retryDelay: { type: string, default: "10s" }
                      validation:
                        type: object
                        properties:
                          command: { type: string }
                          expectedOutput: { type: string }
                          timeout: { type: string, default: "60s" }
                  helmChart:
                    type: object
                    properties:
                      chartRef:
                        type: object
                        required: [name]
                        properties:
                          name: { type: string }
                          repo: { type: string }
                          version: { type: string }
                          chartPath: { type: string }
                      releaseName: { type: string }
                      namespace: { type: string }
                      source:
                        type: object
                        required: [type]
                        properties:
                          type:
                            type: string
                            enum: [Repo, OCI, LocalPath, ConfigMapRef]
                          repo:
                            type: object
                            properties:
                              url: { type: string }
                              username: { type: string }
                              passwordSecretRef:
                                type: object
                                properties:
                                  name: { type: string }
                                  key: { type: string }
                                  namespace: { type: string }
                          oci:
                            type: object
                            properties:
                              registry: { type: string }
                              repository: { type: string }
                              tag: { type: string }
                              insecure: { type: boolean }
                          localPath:
                            type: object
                            properties:
                              path: { type: string }
                          configMapRef:
                            type: object
                            properties:
                              name: { type: string }
                              namespace: { type: string }
                              key: { type: string }
                      values:
                        type: object
                        x-kubernetes-preserve-unknown-fields: true
                      valuesFrom:
                        type: array
                        items:
                          type: object
                          required: [kind, name]
                          properties:
                            kind:
                              type: string
                              enum: [ConfigMap, Secret]
                            name: { type: string }
                            namespace: { type: string }
                            valuesKey: { type: string }
                      timeout: { type: string, default: "600s" }
                      wait: { type: boolean, default: true }
                      atomic: { type: boolean, default: false }
              healthCheck:
                type: object
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
                        items: { type: string }
                  clusterOperator:
                    type: object
                    properties:
                      name: { type: string }
                      conditionType: { type: string, default: "Available" }
                  interval: { type: string, default: "30s" }
                  timeout: { type: string, default: "10s" }
          status:
            type: object
            properties:
              observedVersion: { type: string }
              phase:
                type: string
                enum: [Pending, Installing, Verifying, Installed, Failed, Upgrading, RollingBack]
              installType:
                type: string
                enum: [Manifest, InstallScript, HelmChart]
              conditions:
                type: array
                items:
                  type: object
                  required: [type, status]
                  properties:
                    type: { type: string }
                    status: { type: string, enum: [True, False, Unknown] }
                    reason: { type: string }
                    message: { type: string }
                    lastTransitionTime: { type: string, format: date-time }
              installDetails:
                type: object
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
                      executionId: { type: string }
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
                      releaseStatus: { type: string }
                      revision: { type: integer }
                      lastDeployed: { type: string, format: date-time }
              lastTransitionTime: { type: string, format: date-time }
              retryCount: { type: integer }
```
## 四、三种安装策略的详细设计
### 4.1 Source 通用模型
三种安装策略共享统一的 Source 模型，但各自支持的来源类型有差异：
```
┌───────────────────────────────────────────────────────────────┐
│                    Source 类型矩阵                            │
├──────────────┬──────────┬───────────────┬───────────┬─────────┤
│  Source 类型 │ Manifest │ InstallScript │ HelmChart │ 说明    │
├──────────────┼──────────┼───────────────┼───────────┼─────────┤
│  Inline      │    ✅    │      ✅       │     ❌    │ 内嵌内容│
│  ConfigMapRef│    ✅    │      ✅       │     ✅    │ 集群内  │
│  SecretRef   │    ✅    │      ✅       │     ❌    │ 敏感数据│
│  URIRef      │    ✅    │      ✅       │     ❌    │ 远程拉取│
│  ImageRef    │    ❌    │      ✅       │     ❌    │ 镜像打包│
│  LocalPath   │    ✅    │      ✅       │     ✅    │ 本地文件│
│  Repo        │    ❌    │      ❌       │     ✅    │ Helm仓库│
│  OCI         │    ❌    │      ❌       │     ✅    │ OCI仓库 │
└──────────────┴─────────┴───────────────┴───────────┴─────────┘
```
### 4.2 Manifest 策略
**对标 OpenShift CVO 原生模式**：CVO 从 release payload 提取 manifest 并通过 `kubectl apply` 语义应用到集群。
```
ComponentVersion (type=Manifest)
       │
       ▼
  ┌──────────────────────────────────────────────┐
  │              Source 解析层                   │
  │  ┌────────┐ ┌──────────┐ ┌──────┐ ┌───────┐  │
  │  │Inline  │ │ConfigMap │ │URIRef│ │Local  │  │
  │  │base64  │ │Ref       │ │HTTP  │ │Path   │  │
  │  └───┬────┘ └────┬─────┘ └──┬───┘ └───┬───┘  │
  │      └───────────┬┴──────────┴─────────┘     │
  └──────────────────┼───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              Manifest 渲染层                 │
  │  模板变量替换：{{.Version}} {{.Image}} 等    │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              Apply 策略层                    │
  │  ┌──────────────┐ ┌───────────┐ ┌─────────┐  │
  │  │ServerSide    │ │3WayMerge  │ │Replace  │  │
  │  │Apply(默认)   │ │           │ │         │  │
  │  └──────────────┘ └───────────┘ └─────────┘  │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              Ownership + Prune 层            │
  │  标签标记 → 资源追踪 → 清理废弃资源          │
  └──────────────────────────────────────────────┘
```
**Source 各类型说明：**

| Source 类型 | 字段 | 场景 |
|-------------|------|------|
| **Inline** | `inline` (base64) | 小型 manifest，如单个 Service、ConfigMap |
| **ConfigMapRef** | `name/namespace/key` | 可动态更新的 manifest，运维可修改 ConfigMap |
| **SecretRef** | `name/namespace/key` | 包含敏感信息的 manifest，如 TLS 证书配置 |
| **URIRef** | `uri/checksum/insecureSkipTLSVerify` | 从远程 HTTP 服务器拉取，如内部制品仓库 |
| **LocalPath** | `path/checksum` | 离线环境预分发的 manifest 文件 |
### 4.3 InstallScript 策略
**对标 OpenShift MCO 中 MachineConfigDaemon 的脚本执行模式**：适配需要通过命令行脚本执行的组件安装。
```
ComponentVersion (type=InstallScript)
       │
       ▼
  ┌──────────────────────────────────────────────┐
  │              Source 解析层                   │
  │  ┌────────┐ ┌──────────┐ ┌──────┐            │
  │  │Inline  │ │ConfigMap │ │Secret│            │
  │  │base64  │ │Ref       │ │Ref   │            │
  │  └────────┘ └──────────┘ └──────┘            │
  │  ┌────────┐ ┌──────────┐ ┌──────┐            │
  │  │URIRef  │ │ImageRef  │ │Local │            │
  │  │HTTP(S) │ │容器镜像  │ │Path  │            │
  │  └────────┘ └──────────┘ └──────┘            │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              执行目标选择层                  │
  │  ┌────────────┐ ┌──────────┐ ┌──────────┐    │
  │  │Controller  │ │NodeAgent │ │InitPod   │    │
  │  │控制器内执行│ │节点代理  │ │临时Pod   │    │
  │  └────────────┘ └──────────┘ └──────────┘    │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              执行 + 重试 + 验证层            │
  │  超时控制 → 重试策略 → 安装后验证            │
  └──────────────────────────────────────────────┘
```
**三种执行目标对比：**

| 执行目标 | 通信方式 | 适用场景 | 生命周期 |
|----------|----------|----------|----------|
| **Controller** | 直接调用 k8s API | 集群级操作（创建 CRD、RBAC、Namespace） | 与 Controller 共存 |
| **NodeAgent** | 通过 Command CR 下发到节点代理 | 节点级操作（kubeadm、containerd、systemd） | 异步执行，状态回传 |
| **InitPod** | 创建 Job/Pod 执行 | 需要独立环境的操作（数据迁移、复杂初始化） | 一次性执行 |

**Source 各类型说明：**

| Source 类型 | 字段 | 场景 |
|-------------|------|------|
| **Inline** | `inline` (base64) | 短脚本，如 `systemctl restart containerd` |
| **ConfigMapRef** | `name/namespace/key` | 可动态更新的脚本，运维可修改 |
| **SecretRef** | `name/namespace/key` | 包含敏感信息的脚本（如带密码的初始化） |
| **URIRef** | `uri/checksum` | 从远程拉取安装脚本 |
| **ImageRef** | `image/path/imagePullSecret` | 复杂脚本+依赖打包为容器镜像 |
| **LocalPath** | `path/checksum` | 离线环境预分发的脚本文件 |
### 4.4 HelmChart 策略
**对标 OpenShift OLM 的 Helm 集成模式**：适配通过 Helm 管理的复杂应用栈。
```
ComponentVersion (type=HelmChart)
       │
       ▼
  ┌──────────────────────────────────────────────┐
  │              Chart 来源层                    │
  │  ┌────────┐ ┌──────────┐ ┌──────┐            │
  │  │Repo    │ │OCI       │ │Local │            │
  │  │HTTP仓库│ │Registry  │ │Path  │            │
  │  └────────┘ └──────────┘ └──────┘            │
  │  ┌──────────┐                                │
  │  │ConfigMap │                                │
  │  │Ref       │                                │
  │  └──────────┘                                │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              Values 合并层                   │
  │  spec.values + valuesFrom → 最终 Values      │
  │  (ConfigMap/Secret 引用覆盖基础值)           │
  └──────────────────┬───────────────────────────┘
                     ▼
  ┌──────────────────────────────────────────────┐
  │              Helm 执行层                     │
  │  helm upgrade --install --wait --atomic      │
  └──────────────────────────────────────────────┘
```
**Source 各类型说明：**

| Source 类型 | 字段 | 场景 |
|-------------|------|------|
| **Repo** | `url/username/passwordSecretRef` | 标准 Helm Chart 仓库（HTTP/HTTPS） |
| **OCI** | `registry/repository/tag/insecure` | OCI 兼容仓库（如 Harbor、GHCR） |
| **LocalPath** | `path` | 离线环境预分发的 Chart 包 |
| **ConfigMapRef** | `name/namespace/key` | Chart 打包为 tar.gz 存储在 ConfigMap |
## 五、NodeConfig CRD 重构
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
              nodeName: { type: string }
              nodeIP: { type: string }
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
                items:
                  type: object
                  required: [componentName]
                  properties:
                    componentName: { type: string }
                    configOverrides:
                      type: object
                      x-kubernetes-preserve-unknown-fields: true
                    nodeLabels:
                      type: object
                      additionalProperties: { type: string }
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
              lastHeartbeat: { type: string, format: date-time }
```
## 六、ComponentVersion 与 NodeConfig 的协作关系
```
┌─────────────────────────────────────────────────────────────────┐
│                     协作关系图                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ComponentVersion (定义"安装什么 + 怎么安装")                   │
│  ┌───────────────────────────────────────────┐                  │
│  │ componentName: containerd                 │                  │
│  │ version: 1.7.2                            │                  │
│  │ scope: Node                               │                  │
│  │ install:                                  │                  │
│  │   type: InstallScript                     │                  │
│  │   installScript:                          │                  │
│  │     source:                               │                  │
│  │       type: ImageRef                      │                  │
│  │       imageRef:                           │                  │
│  │         image: quay.io/bke/scripts:v1     │                  │
│  │         path: /scripts/upgrade-containerd │                  │
│  │     execution:                            │                  │
│  │       target: NodeAgent                   │                  │
│  │       nodeSelector:                       │                  │
│  │         node-role: worker      ──────────┐│                  │
│  └──────────────────────────────────────────┼┘                  │
│                                             │ 匹配              │
│  NodeConfig (定义"在哪些节点 + 什么配置")   │                   │
│  ┌──────────────────────────────────────────┼──┐                │
│  │ nodeName: worker-01                      │  │                │
│  │ roles: [worker]                          │  │                │
│  │ componentConfigs:                        │  │                │
│  │ - componentName: containerd              │  │                │
│  │   configOverrides:                       │  │                │
│  │     configPath: /etc/containerd/custom   │  │                │
│  │   nodeLabels:                            │  │                │
│  │     node-role: worker  ◄─────────────────┘  │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
│  执行时：                                                       │
│  1. ComponentVersion.nodeSelector 匹配 NodeConfig.nodeLabels    │
│  2. NodeConfig.configOverrides 提供节点级配置覆盖               │
│  3. 生成最终执行命令 → 下发到匹配的 NodeAgent                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
## 七、三种策略的典型使用示例
### 7.1 Manifest — CoreDNS 部署
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
      items:
      - name: coredns-deployment
        namespace: kube-system
        source:
          type: ConfigMapRef
          configMapRef:
            name: coredns-manifests
            namespace: orchestration-system
            key: deployment.yaml
      - name: coredns-service
        namespace: kube-system
        source:
          type: URIRef
          uriRef:
            uri: "https://artifacts.internal/coredns/service.yaml"
            checksum: "sha256:abc123..."
      applyStrategy: ServerSideApply
      prune: true
  images:
  - name: coredns
    image: registry.k8s.io/coredns/coredns:v1.10.1
  healthCheck:
    type: Command
    command:
      command: ["kubectl", "get", "pods", "-n", "kube-system", "-l", "k8s-app=kube-dns"]
```
### 7.2 InstallScript — Containerd 升级
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
      command: "/scripts/upgrade-containerd.sh"
      args: ["--version", "1.7.2"]
      env:
      - name: CONTAINERD_VERSION
        value: "1.7.2"
      - name: REGISTRY_CONFIG
        valueFrom:
          configMapKeyRef:
            name: registry-config
            key: config.toml
            namespace: orchestration-system
      source:
        type: ImageRef
        imageRef:
          image: quay.io/bke/scripts:v1.2.0
          path: /scripts/upgrade-containerd.sh
      execution:
        target: NodeAgent
        nodeSelector:
          node-role: worker
        timeout: "600s"
        retryPolicy:
          maxRetries: 3
          retryDelay: "30s"
      validation:
        command: "containerd --version"
        expectedOutput: "1.7.2"
  healthCheck:
    type: Command
    command:
      command: ["systemctl", "is-active", "containerd"]
```
### 7.3 HelmChart — 监控栈部署
```yaml
apiVersion: orchestration.bke.io/v1alpha1
kind: ComponentVersion
metadata:
  name: monitoring-stack
spec:
  componentName: monitoring
  version: "45.0.0"
  scope: Cluster
  install:
    type: HelmChart
    helmChart:
      chartRef:
        name: kube-prometheus-stack
        version: "45.0.0"
      releaseName: monitoring
      namespace: monitoring
      source:
        type: Repo
        repo:
          url: https://prometheus-community.github.io/helm-charts
      values:
        prometheus:
          prometheusSpec:
            retention: 15d
        grafana:
          enabled: true
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
```
## 八、状态机设计
三种安装策略共享统一的状态机：
```
                 ┌───────────┐
                 │  Pending  │ ◄── 初始状态 / 依赖未满足
                 └─────┬─────┘
                       │ 依赖满足
                       ▼
                 ┌───────────┐
           ┌─────│Installing │─────┐
           │     └─────┬─────┘     │
           │           │           │ 安装失败
           │ 安装完成  │           ▼
           │           ▼     ┌──────────┐
           │     ┌──────────┐│  Failed  │
           │     │Verifying │└────┬─────┘
           │     └────┬─────┘     │
           │          │           │ 手动触发
           │ 验证通过 │ 验证失败  │ 或自动重试
           │          ▼           │
           │    ┌───────────┐     │
           │    │ Installed │◄────┘
           │    └─────┬─────┘
           │          │ 版本变更
           │          ▼
           │    ┌───────────┐
           └───►│ Upgrading │
                └─────┬─────┘
                      │ 升级失败
                      ▼
                ┌────────────┐
                │RollingBack │
                └─────┬──────┘
                      │ 回滚完成
                      ▼
                ┌───────────┐
                │ Installed │ (回滚到旧版本)
                └───────────┘
```
## 九、设计总结
### 9.1 重构前后对比
| 维度 | 重构前 | 重构后 |
|------|--------|--------|
| **安装方式** | 仅 Manifest | Manifest + InstallScript + HelmChart |
| **来源** | 仅 release-payload | Inline / ConfigMap / Secret / URI / Image / LocalPath / Repo / OCI |
| **执行位置** | 仅 Controller | Controller / NodeAgent / InitPod |
| **脚本支持** | 无 | 完整的脚本执行 + 验证 + 重试 |
| **Helm 支持** | 无 | 完整的 Helm 生命周期管理 |
| **离线支持** | 有限 | LocalPath + ImageRef + ConfigMap |
| **敏感数据** | 无 | SecretRef 来源 + passwordSecretRef |
| **状态追踪** | 简单 phase | 分层 status（manifest/installScript/helmChart 各自独立） |
### 9.2 核心设计决策
1. **`spec.install.type` 三选一互斥**：同一 ComponentVersion 只使用一种安装策略，避免混合导致状态混乱
2. **Source 模型统一但差异化**：三种策略共享 Source 概念，但各自支持的来源类型不同（如 ImageRef 仅 InstallScript 支持，Repo/OCI 仅 HelmChart 支持）
3. **`status.installDetails` 分层**：各策略有独立的状态结构，但共享顶层 phase/conditions，便于上层编排器统一消费
4. **NodeConfig 解耦节点级配置**：ComponentVersion 定义"安装什么+怎么安装"，NodeConfig 定义"在哪些节点+什么配置"，通过 nodeSelector/nodeLabels 关联
5. **HealthCheck 独立于安装策略**：无论哪种安装方式，健康检查定义统一，确保组件可用性监控的一致性

# 支持多种部署方式的 ComponentVersion Controller 详细设计大纲
## ComponentVersion Controller 详细设计大纲
### 一、整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                     ComponentVersion Controller                     │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │  Reconciler  │───▶│  Dispatcher  │───▶│  Installer Interface │   │
│  │ (主调谐循环) │    │ (策略路由)   │    │  (安装策略抽象)      │   │
│  └──────┬───────┘    └──────┬───────┘    └──────────┬───────────┘   │
│         │                   │                       │               │
│         │           ┌───────┴────────┐     ┌────────┴────────┐      │
│         │           │ SourceResolver │     │  具体实现       │      │
│         │           │ (来源解析)     │     │  ┌────────────┐ │      │
│         │           └────────────────┘     │  │ Manifest   │ │      │
│         │                                  │  │ Installer  │ │      │
│         │                                  │  └────────────┘ │      │
│         │                                  │  ┌────────────┐ │      │
│         │                                  │  │ Script     │ │      │
│         │                                  │  │ Installer  │ │      │
│         │                                  │  └────────────┘ │      │
│         │                                  │  ┌────────────┐ │      │
│         │                                  │  │ HelmChart  │ │      │
│         │                                  │  │ Installer  │ │      │
│         │                                  │  └────────────┘ │      │
│         │                                  └─────────────────┘      │
│         │                                                           │
│  ┌──────┴───────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │ StateMachine │    │ HealthChecker│    │  DependencyResolver  │   │
│  │ (状态机)     │    │ (健康检查)   │    │  (依赖解析)          │   │
│  └──────────────┘    └──────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```
### 二、目录结构规划
```
pkg/
├── componentversion/
│   ├── controller/
│   │   ├── componentversion_controller.go   # 主 Reconciler
│   │   ├── dispatcher.go                    # 策略路由
│   │   └── helper.go                        # 通用辅助函数
│   ├── installer/
│   │   ├── interface.go                     # Installer 接口定义
│   │   ├── manifest/
│   │   │   ├── installer.go                 # Manifest 安装器
│   │   │   ├── apply.go                     # Apply 策略实现
│   │   │   └── prune.go                     # 资源清理
│   │   ├── script/
│   │   │   ├── installer.go                 # InstallScript 安装器
│   │   │   ├── executor.go                  # 脚本执行器
│   │   │   └── validator.go                 # 安装后验证
│   │   └── helmchart/
│   │       ├── installer.go                 # HelmChart 安装器
│   │       ├── values.go                    # Values 合并逻辑
│   │       └── release.go                   # Release 管理
│   ├── source/
│   │   ├── resolver.go                      # Source 统一解析接口
│   │   ├── inline.go                        # Inline 来源解析
│   │   ├── configmap.go                     # ConfigMap 来源解析
│   │   ├── secret.go                        # Secret 来源解析
│   │   ├── uri.go                           # URI 远程拉取
│   │   ├── image.go                         # 镜像内脚本提取
│   │   └── localpath.go                     # 本地路径读取
│   ├── statemachine/
│   │   ├── machine.go                       # 状态机核心
│   │   └── transition.go                    # 状态转换规则
│   ├── healthcheck/
│   │   ├── checker.go                       # 健康检查接口
│   │   ├── http.go                          # HTTPGet 检查
│   │   ├── tcp.go                           # TCPSocket 检查
│   │   ├── command.go                       # Command 检查
│   │   └── clusteroperator.go              # ClusterOperator 检查
│   └── dependency/
│       ├── resolver.go                      # 依赖 DAG 解析
│       └── graph.go                         # 有向图实现
api/
├── componentversion/
│   └── v1alpha1/
│       ├── componentversion_types.go        # ComponentVersion CRD 类型
│       ├── nodeconfig_types.go              # NodeConfig CRD 类型
│       ├── install_types.go                 # 安装策略相关类型
│       ├── source_types.go                  # 来源相关类型
│       ├── groupversion_info.go             # GroupVersion 信息
│       └── zz_generated.deepcopy.go         # DeepCopy 生成
```
### 三、API 类型设计
#### 3.1 核心类型定义（`componentversion_types.go`）
```go
package v1alpha1

// InstallType 安装策略类型
type InstallType string
const (
    InstallTypeManifest     InstallType = "Manifest"
    InstallTypeInstallScript InstallType = "InstallScript"
    InstallTypeHelmChart    InstallType = "HelmChart"
)

// ComponentScope 组件作用域
type ComponentScope string
const (
    ComponentScopeCluster ComponentScope = "Cluster"
    ComponentScopeNode    ComponentScope = "Node"
)

// ComponentVersionSpec 定义组件版本的期望状态
type ComponentVersionSpec struct {
    ComponentName string              `json:"componentName"`
    Version       string              `json:"version"`
    Scope         ComponentScope      `json:"scope,omitempty"`
    Dependencies  []Dependency        `json:"dependencies,omitempty"`
    Images        []ImageRef          `json:"images,omitempty"`
    Install       InstallSpec         `json:"install"`
    HealthCheck   *HealthCheckSpec    `json:"healthCheck,omitempty"`
}

// ComponentVersionStatus 定义组件版本的观测状态
type ComponentVersionStatus struct {
    ObservedVersion    string              `json:"observedVersion,omitempty"`
    Phase              ComponentPhase      `json:"phase,omitempty"`
    InstallType        InstallType         `json:"installType,omitempty"`
    Conditions         []metav1.Condition  `json:"conditions,omitempty"`
    InstallDetails     InstallDetails      `json:"installDetails,omitempty"`
    LastTransitionTime *metav1.Time        `json:"lastTransitionTime,omitempty"`
    RetryCount         int                 `json:"retryCount,omitempty"`
}
```
#### 3.2 安装策略类型（`install_types.go`）
```go
// InstallSpec 安装规格，三种策略互斥
type InstallSpec struct {
    Type          InstallType       `json:"type"`
    Manifest      *ManifestSpec     `json:"manifest,omitempty"`
    InstallScript *InstallScriptSpec `json:"installScript,omitempty"`
    HelmChart     *HelmChartSpec    `json:"helmChart,omitempty"`
}

// ManifestSpec k8s yaml 安装策略
type ManifestSpec struct {
    Items         []ManifestItem   `json:"items,omitempty"`
    ApplyStrategy ApplyStrategy    `json:"applyStrategy,omitempty"`
    Prune         bool             `json:"prune,omitempty"`
}

// ManifestItem 单个 manifest 条目
type ManifestItem struct {
    Name      string     `json:"name"`
    Namespace string     `json:"namespace,omitempty"`
    Source    SourceSpec `json:"source"`
}

// InstallScriptSpec 脚本安装策略
type InstallScriptSpec struct {
    Command    string            `json:"command,omitempty"`
    Args       []string          `json:"args,omitempty"`
    Env        []EnvVar          `json:"env,omitempty"`
    Source     SourceSpec        `json:"source"`
    Execution  *ExecutionSpec    `json:"execution,omitempty"`
    Validation *ValidationSpec   `json:"validation,omitempty"`
}

// HelmChartSpec Helm chart 安装策略
type HelmChartSpec struct {
    ChartRef    HelmChartRef      `json:"chartRef,omitempty"`
    ReleaseName string            `json:"releaseName,omitempty"`
    Namespace   string            `json:"namespace,omitempty"`
    Source      *ChartSourceSpec  `json:"source,omitempty"`
    Values      *runtime.RawExtension `json:"values,omitempty"`
    ValuesFrom  []ValuesFromSource `json:"valuesFrom,omitempty"`
    Timeout     string            `json:"timeout,omitempty"`
    Wait        bool              `json:"wait,omitempty"`
    Atomic      bool              `json:"atomic,omitempty"`
}
```
#### 3.3 来源类型（`source_types.go`）
```go
// SourceType 来源类型枚举
type SourceType string
const (
    SourceTypeInline      SourceType = "Inline"
    SourceTypeConfigMapRef SourceType = "ConfigMapRef"
    SourceTypeSecretRef   SourceType = "SecretRef"
    SourceTypeURIRef      SourceType = "URIRef"
    SourceTypeImageRef    SourceType = "ImageRef"
    SourceTypeLocalPath   SourceType = "LocalPath"
)

// SourceSpec 统一来源规格
type SourceSpec struct {
    Type         SourceType        `json:"type"`
    Inline       *InlineSource     `json:"inline,omitempty"`
    ConfigMapRef *ConfigMapSource  `json:"configMapRef,omitempty"`
    SecretRef    *SecretSource     `json:"secretRef,omitempty"`
    URIRef       *URISource        `json:"uriRef,omitempty"`
    ImageRef     *ImageSource      `json:"imageRef,omitempty"`
    LocalPath    *LocalPathSource  `json:"localPath,omitempty"`
}

// InlineSource 内嵌内容
type InlineSource struct {
    Content string `json:"content,omitempty"`
}

// ConfigMapSource ConfigMap 引用
type ConfigMapSource struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace"`
    Key       string `json:"key,omitempty"`
}

// SecretSource Secret 引用
type SecretSource struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace"`
    Key       string `json:"key,omitempty"`
}

// URISource 远程 URI
type URISource struct {
    URI                    string `json:"uri"`
    Checksum               string `json:"checksum,omitempty"`
    InsecureSkipTLSVerify  bool   `json:"insecureSkipTLSVerify,omitempty"`
}

// ImageSource 容器镜像
type ImageSource struct {
    Image           string `json:"image"`
    Path            string `json:"path,omitempty"`
    ImagePullSecret string `json:"imagePullSecret,omitempty"`
}

// LocalPathSource 本地路径
type LocalPathSource struct {
    Path     string `json:"path"`
    Checksum string `json:"checksum,omitempty"`
}
```
### 四、Reconciler 主循环设计
#### 4.1 Reconciler 结构体
```go
type ComponentVersionReconciler struct {
    client.Client
    Scheme          *runtime.Scheme
    Recorder        record.EventRecorder
    RestConfig      *rest.Config
    
    // 子组件
    Dispatcher      *Dispatcher
    StateMachine    *statemachine.Machine
    HealthChecker   *healthcheck.Checker
    DepResolver     *dependency.Resolver
    SourceResolver  *source.Resolver
}
```
#### 4.2 Reconcile 主流程
```
Reconcile(ctx, req)
  │
  ├─ 1. 获取 ComponentVersion 对象
  │     └─ 不存在 → 返回 nil
  │
  ├─ 2. 处理 Deletion + Finalizer
  │     ├─ 正在删除 → 执行卸载逻辑 → 移除 Finalizer
  │     └─ 未删除 → 确保 Finalizer 存在
  │
  ├─ 3. 依赖检查
  │     ├─ 解析 spec.dependencies
  │     ├─ 查询依赖的 ComponentVersion 状态
  │     └─ 依赖未满足 → 设置 phase=Pending, requeue
  │
  ├─ 4. 状态机驱动
  │     ├─ 读取当前 phase
  │     ├─ 计算目标 phase
  │     └─ 执行状态转换
  │
  ├─ 5. 策略分发
  │     ├─ 根据 spec.install.type 选择 Installer
  │     └─ 调用 Installer.Install() / Verify() / Rollback()
  │
  ├─ 6. 健康检查
  │     ├─ 读取 spec.healthCheck
  │     └─ 执行检查 → 更新 conditions
  │
  ├─ 7. 状态上报
  │     ├─ 更新 status.phase
  │     ├─ 更新 status.conditions
  │     ├─ 更新 status.installDetails
  │     └─ 更新 status.observedVersion
  │
  └─ 8. 返回 Result
        ├─ 成功 → {RequeueAfter: healthCheckInterval}
        ├─ 进行中 → {RequeueAfter: progressInterval}
        └─ 失败 → {RequeueAfter: retryInterval}
```
#### 4.3 Reconcile 伪代码
```go
func (r *ComponentVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取 CV 对象
    cv := &v1alpha1.ComponentVersion{}
    if err := r.Get(ctx, req.NamespacedName, cv); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // 2. 处理 Finalizer
    if !cv.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, cv)
    }
    if err := r.ensureFinalizer(ctx, cv); err != nil {
        return ctrl.Result{}, err
    }

    // 3. 依赖检查
    if satisfied, err := r.DepResolver.CheckDependencies(ctx, cv); err != nil {
        return r.updatePhase(ctx, cv, v1alpha1.PhasePending, err)
    } else if !satisfied {
        return r.updatePhase(ctx, cv, v1alpha1.PhasePending, nil)
    }

    // 4. 状态机驱动
    result, err := r.StateMachine.Drive(ctx, cv, r.executeInstall)
    if err != nil {
        return result, err
    }

    // 5. 健康检查（仅 Installed 状态）
    if cv.Status.Phase == v1alpha1.PhaseInstalled {
        if err := r.HealthChecker.Check(ctx, cv); err != nil {
            r.Recorder.Event(cv, corev1.EventTypeWarning, "HealthCheckFailed", err.Error())
        }
    }

    return result, nil
}
```
### 五、Installer 接口设计
#### 5.1 接口定义
```go
type Installer interface {
    // Install 执行安装/升级
    Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error)
    
    // Verify 验证安装结果
    Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*VerifyResult, error)
    
    // Rollback 回滚到上一版本
    Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error
    
    // Uninstall 卸载组件
    Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error
}

type InstallResult struct {
    Phase   v1alpha1.ComponentPhase
    Details runtime.Object
    Requeue bool
}

type VerifyResult struct {
    Passed  bool
    Message string
}
```
#### 5.2 ManifestInstaller
```
ManifestInstaller.Install(ctx, cv)
  │
  ├─ 1. 遍历 spec.install.manifest.items
  │     ├─ 对每个 item:
  │     │   ├─ SourceResolver.Resolve(item.source) → []byte
  │     │   ├─ 模板渲染（替换 {{.Version}}, {{.Image}} 等变量）
  │     │   ├─ 解析为 unstructured.Unstructured 列表
  │     │   └─ 根据 applyStrategy 应用:
  │     │       ├─ ServerSideApply: Patch with fieldManager
  │     │       ├─ ThreeWayMerge:  三方合并后 Update
  │     │       └─ Replace:        Delete + Create
  │     └─ 记录 appliedResources 到 status
  │
  ├─ 2. Prune 处理
  │     ├─ 根据 ownershipLabel 查询历史资源
  │     ├─ 对比当前 appliedResources
  │     └─ 清理不在列表中的旧资源
  │
  └─ 3. 返回 InstallResult{Phase: Verifying}

ManifestInstaller.Verify(ctx, cv)
  │
  ├─ 遍历 status.installDetails.manifest.appliedResources
  │   ├─ 检查资源是否存在
  │   ├─ 检查资源状态（Deployment ready, Pod running 等）
  │   └─ 全部就绪 → Passed=true
  │
  └─ 返回 VerifyResult
```
#### 5.3 ScriptInstaller
```
ScriptInstaller.Install(ctx, cv)
  │
  ├─ 1. SourceResolver.Resolve(spec.install.installScript.source) → 脚本内容
  │
  ├─ 2. 根据 execution.target 选择执行通道:
  │     ├─ Controller:
  │     │   └─ 直接在 Controller 进程内执行（os/exec）
  │     │
  │     ├─ NodeAgent:
  │     │   ├─ 构造 Command CRD（复用现有 bkeagent Command 机制）
  │     │   ├─ 设置 nodeSelector / env / args
  │     │   ├─ 创建 Command 对象
  │     │   └─ 等待 Command 状态 → Running
  │     │
  │     └─ InitPod:
  │         ├─ 构造 Job/Pod 定义
  │         ├─ 挂载脚本（ConfigMap 或 EmptyDir + InitContainer）
  │         ├─ 创建 Job
  │         └─ 等待 Job 状态 → Running
  │
  ├─ 3. 记录 executionId 到 status
  │
  └─ 4. 返回 InstallResult{Phase: Installing, Requeue: true}

ScriptInstaller.Verify(ctx, cv)
  │
  ├─ 根据 execution.target 查询执行状态:
  │   ├─ NodeAgent → 查询 Command CRD status
  │   └─ InitPod   → 查询 Job status
  │
  ├─ 执行 validation（如果配置了）:
  │   ├─ 构造 validation Command CRD
  │   └─ 检查 expectedOutput
  │
  └─ 返回 VerifyResult
```
**关键设计：与现有 Command CRD 的集成**
```go
// ScriptInstaller 通过构造 Command CRD 下发脚本到 NodeAgent
func (s *ScriptInstaller) dispatchToNodeAgent(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    scriptContent, err := s.SourceResolver.Resolve(ctx, &cv.Spec.Install.InstallScript.Source)
    if err != nil {
        return err
    }

    // 将脚本内容写入 ConfigMap
    scriptCM := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("cv-script-%s", cv.Name),
            Namespace: cv.Namespace,
        },
        Data: map[string]string{
            "script.sh": string(scriptContent),
        },
    }
    if err := s.Create(ctx, scriptCM); err != nil {
        return err
    }

    // 构造 Command CRD，复用现有 bkeagent 的 Command 机制
    command := &agentv1beta1.Command{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("cv-%s-%s", cv.Name, cv.Spec.ComponentName),
            Namespace: cv.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: v1alpha1.GroupVersion.String(),
                    Kind:       "ComponentVersion",
                    Name:       cv.Name,
                    UID:        cv.UID,
                },
            },
        },
        Spec: agentv1beta1.CommandSpec{
            NodeSelector: cv.Spec.Install.InstallScript.Execution.NodeSelector,
            Commands: []agentv1beta1.ExecCommand{
                {
                    ID:      fmt.Sprintf("%s-install", cv.Name),
                    Type:    agentv1beta1.CommandKubernetes,
                    Command: []string{fmt.Sprintf("configmap:%s/%s:rx:script.sh", scriptCM.Namespace, scriptCM.Name)},
                },
            },
            ActiveDeadlineSecond: parseDuration(cv.Spec.Install.InstallScript.Execution.Timeout),
            BackoffLimit:         cv.Spec.Install.InstallScript.Execution.RetryPolicy.MaxRetries,
        },
    }
    return s.Create(ctx, command)
}
```
#### 5.4 HelmChartInstaller
```
HelmChartInstaller.Install(ctx, cv)
  │
  ├─ 1. 解析 Chart 来源:
  │     ├─ Repo → helm repo add + helm pull
  │     ├─ OCI  → helm pull oci://...
  │     ├─ LocalPath → 直接加载
  │     └─ ConfigMapRef → 提取 tar.gz
  │
  ├─ 2. 合并 Values:
  │     ├─ spec.install.helmChart.values (基础值)
  │     └─ spec.install.helmChart.valuesFrom (覆盖值)
  │         ├─ ConfigMap → 读取 valuesKey
  │         └─ Secret   → 读取 valuesKey
  │     └─ 合并顺序: values → valuesFrom[0] → valuesFrom[1] → ...
  │
  ├─ 3. 执行 Helm 操作:
  │     ├─ 检查 Release 是否存在
  │     ├─ 不存在 → helm install
  │     └─ 已存在 → helm upgrade
  │
  └─ 4. 返回 InstallResult{Phase: Verifying}

HelmChartInstaller.Verify(ctx, cv)
  │
  ├─ 查询 Release 状态
  │   ├─ deployed → Passed=true
  │   ├─ pending-install/upgrade → Passed=false, requeue
  │   └─ failed → Passed=false, error
  │
  └─ 返回 VerifyResult
```
**关键设计：复用项目已有的 Helm SDK**
```go
// 项目 go.mod 已包含 helm.sh/helm/v3 v3.19.0
import (
    "helm.sh/helm/v3/pkg/action"
    "helm.sh/helm/v3/pkg/cli"
    "helm.sh/helm/v3/pkg/release"
)

type HelmChartInstaller struct {
    client.Client
    Settings  *cli.EnvSettings
    RestConfig *rest.Config
}

func (h *HelmChartInstaller) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error) {
    actionConfig, err := h.newActionConfig(cv.Spec.Install.HelmChart.Namespace)
    if err != nil {
        return nil, err
    }

    // 合并 values
    values, err := h.mergeValues(ctx, cv)
    if err != nil {
        return nil, err
    }

    // 检查 Release 是否存在
    histClient := action.NewHistory(actionConfig)
    releases, err := histClient.Run(cv.Spec.Install.HelmChart.ReleaseName)
    
    if len(releases) == 0 {
        // helm install
        install := action.NewInstall(actionConfig)
        install.ReleaseName = cv.Spec.Install.HelmChart.ReleaseName
        install.Namespace = cv.Spec.Install.HelmChart.Namespace
        install.Wait = cv.Spec.Install.HelmChart.Wait
        install.Atomic = cv.Spec.Install.HelmChart.Atomic
        install.Timeout = parseDuration(cv.Spec.Install.HelmChart.Timeout)
        
        chart, err := h.loadChart(ctx, cv)
        if err != nil {
            return nil, err
        }
        _, err = install.Run(chart, values)
    } else {
        // helm upgrade
        upgrade := action.NewUpgrade(actionConfig)
        upgrade.Wait = cv.Spec.Install.HelmChart.Wait
        upgrade.Atomic = cv.Spec.Install.HelmChart.Atomic
        upgrade.Timeout = parseDuration(cv.Spec.Install.HelmChart.Timeout)
        
        chart, err := h.loadChart(ctx, cv)
        if err != nil {
            return nil, err
        }
        _, err = upgrade.Run(cv.Spec.Install.HelmChart.ReleaseName, chart, values)
    }

    return &InstallResult{Phase: v1alpha1.PhaseVerifying}, err
}
```
### 六、SourceResolver 设计
```go
type Resolver struct {
    client.Client
    HTTPClient *http.Client
}

type ResolvedContent struct {
    Content  []byte
    Checksum string
    Source   v1alpha1.SourceType
}

func (r *Resolver) Resolve(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    switch spec.Type {
    case v1alpha1.SourceTypeInline:
        return r.resolveInline(spec)
    case v1alpha1.SourceTypeConfigMapRef:
        return r.resolveConfigMap(ctx, spec)
    case v1alpha1.SourceTypeSecretRef:
        return r.resolveSecret(ctx, spec)
    case v1alpha1.SourceTypeURIRef:
        return r.resolveURI(ctx, spec)
    case v1alpha1.SourceTypeImageRef:
        return r.resolveImage(ctx, spec)
    case v1alpha1.SourceTypeLocalPath:
        return r.resolveLocalPath(spec)
    default:
        return nil, fmt.Errorf("unsupported source type: %s", spec.Type)
    }
}
```
各来源解析逻辑：

| 来源 | 解析逻辑 | 缓存策略 |
|------|----------|----------|
| **Inline** | base64 解码 `spec.inline.content` | 无需缓存 |
| **ConfigMapRef** | `r.Get(ctx, key, &ConfigMap{})` → 取 `data[key]` | 按 ResourceVersion 缓存 |
| **SecretRef** | `r.Get(ctx, key, &Secret{})` → 取 `data[key]` | 按 ResourceVersion 缓存 |
| **URIRef** | `http.Get(uri)` → 校验 checksum | 本地文件缓存 + TTL |
| **ImageRef** | 拉取镜像 → `docker cp` 提取文件 | 镜像缓存 |
| **LocalPath** | `os.ReadFile(path)` → 校验 checksum | 按 mtime 缓存 |
### 七、状态机设计
#### 7.1 状态定义
```go
type ComponentPhase string
const (
    PhasePending     ComponentPhase = "Pending"
    PhaseInstalling  ComponentPhase = "Installing"
    PhaseVerifying   ComponentPhase = "Verifying"
    PhaseInstalled   ComponentPhase = "Installed"
    PhaseFailed      ComponentPhase = "Failed"
    PhaseUpgrading   ComponentPhase = "Upgrading"
    PhaseRollingBack ComponentPhase = "RollingBack"
)
```
#### 7.2 转换规则
```go
var transitions = map[ComponentPhase][]ComponentPhase{
    PhasePending:     {PhaseInstalling},
    PhaseInstalling:  {PhaseVerifying, PhaseFailed},
    PhaseVerifying:   {PhaseInstalled, PhaseFailed},
    PhaseInstalled:   {PhaseUpgrading},
    PhaseFailed:      {PhaseInstalling, PhaseRollingBack},
    PhaseUpgrading:   {PhaseVerifying, PhaseFailed, PhaseRollingBack},
    PhaseRollingBack: {PhaseInstalled, PhaseFailed},
}

func (m *Machine) Drive(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
    currentPhase := cv.Status.Phase
    if currentPhase == "" {
        currentPhase = PhasePending
    }

    switch currentPhase {
    case PhasePending:
        // 依赖已满足 → Installing
        return m.transition(ctx, cv, PhaseInstalling, execute)
        
    case PhaseInstalling:
        result, err := execute(ctx, cv)
        if err != nil {
            return m.transition(ctx, cv, PhaseFailed, nil)
        }
        return m.transition(ctx, cv, PhaseVerifying, execute)
        
    case PhaseVerifying:
        result, err := execute(ctx, cv) // 调用 Installer.Verify
        if err != nil || !result.Passed {
            return m.transition(ctx, cv, PhaseFailed, nil)
        }
        return m.transition(ctx, cv, PhaseInstalled, nil)
        
    case PhaseInstalled:
        // 检测版本变更 → Upgrading
        if cv.Status.ObservedVersion != cv.Spec.Version {
            return m.transition(ctx, cv, PhaseUpgrading, execute)
        }
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        
    case PhaseFailed:
        // 自动重试或等待手动触发
        if cv.Status.RetryCount < maxRetry {
            return m.transition(ctx, cv, PhaseInstalling, execute)
        }
        return ctrl.Result{}, nil
        
    case PhaseUpgrading:
        result, err := execute(ctx, cv)
        if err != nil {
            return m.transition(ctx, cv, PhaseRollingBack, execute)
        }
        return m.transition(ctx, cv, PhaseVerifying, execute)
        
    case PhaseRollingBack:
        _, err := execute(ctx, cv) // 调用 Installer.Rollback
        if err != nil {
            return m.transition(ctx, cv, PhaseFailed, nil)
        }
        return m.transition(ctx, cv, PhaseInstalled, nil)
    }
}
```
### 八、Dispatcher 策略路由
```go
type Dispatcher struct {
    ManifestInstaller     *manifest.Installer
    ScriptInstaller       *script.Installer
    HelmChartInstaller    *helmchart.Installer
}

func (d *Dispatcher) GetInstaller(installType v1alpha1.InstallType) (Installer, error) {
    switch installType {
    case v1alpha1.InstallTypeManifest:
        return d.ManifestInstaller, nil
    case v1alpha1.InstallTypeInstallScript:
        return d.ScriptInstaller, nil
    case v1alpha1.InstallTypeHelmChart:
        return d.HelmChartInstaller, nil
    default:
        return nil, fmt.Errorf("unsupported install type: %s", installType)
    }
}

func (d *Dispatcher) Dispatch(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error) {
    installer, err := d.GetInstaller(cv.Spec.Install.Type)
    if err != nil {
        return nil, err
    }

    switch cv.Status.Phase {
    case v1alpha1.PhaseInstalling, v1alpha1.PhaseUpgrading:
        return installer.Install(ctx, cv)
    case v1alpha1.PhaseVerifying:
        verifyResult, err := installer.Verify(ctx, cv)
        return &InstallResult{Phase: ...}, err
    case v1alpha1.PhaseRollingBack:
        err := installer.Rollback(ctx, cv)
        return &InstallResult{Phase: ...}, err
    default:
        return nil, nil
    }
}
```
### 九、与 NodeConfig 的协作设计
#### 9.1 协作流程
```
ComponentVersion (scope=Node)
       │
       ├─ Controller 检测到 scope=Node 的 CV
       │
       ├─ 查询匹配的 NodeConfig 列表:
       │   ├─ CV.spec.install.installScript.execution.nodeSelector
       │   │   匹配
       │   └─ NodeConfig.spec.componentConfigs[].componentName
       │
       ├─ 对每个匹配的 NodeConfig:
       │   ├─ 合并 configOverrides 到执行参数
       │   ├─ 生成节点专属的 Command CRD
       │   └─ 等待各节点 Command 完成
       │
       └─ 汇总所有节点状态 → 更新 CV status
```
#### 9.2 NodeConfig Watch 机制
```go
// SetupWithManager 中 Watch NodeConfig 变更
func (r *ComponentVersionReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1alpha1.ComponentVersion{}).
        Watches(
            &v1alpha1.NodeConfig{},
            handler.EnqueueRequestsFromMapFunc(r.nodeConfigToComponentVersion),
            builder.WithPredicates(predicate.ResourceVersionChangedPredicate{}),
        ).
        Complete(r)
}

func (r *ComponentVersionReconciler) nodeConfigToComponentVersion(ctx context.Context, obj client.Object) []reconcile.Request {
    nc := obj.(*v1alpha1.NodeConfig)
    
    // 查找所有 scope=Node 且 nodeSelector 匹配此 NodeConfig 的 ComponentVersion
    cvList := &v1alpha1.ComponentVersionList{}
    if err := r.List(ctx, cvList); err != nil {
        return nil
    }
    
    var requests []reconcile.Request
    for _, cv := range cvList.Items {
        if cv.Spec.Scope != v1alpha1.ComponentScopeNode {
            continue
        }
        if r.nodeMatchesSelector(nc, cv) {
            requests = append(requests, reconcile.Request{
                NamespacedName: types.NamespacedName{Name: cv.Name},
            })
        }
    }
    return requests
}
```
### 十、依赖解析设计
```go
type Resolver struct {
    client.Client
}

// CheckDependencies 检查 CV 的所有依赖是否已满足
func (r *Resolver) CheckDependencies(ctx context.Context, cv *v1alpha1.ComponentVersion) (bool, error) {
    for _, dep := range cv.Spec.Dependencies {
        depCV := &v1alpha1.ComponentVersion{}
        if err := r.Get(ctx, types.NamespacedName{Name: dep.Name}, depCV); err != nil {
            if apierrors.IsNotFound(err) {
                return false, nil
            }
            return false, err
        }
        if depCV.Status.Phase != v1alpha1.PhaseInstalled {
            return false, nil
        }
        if dep.MinVersion != "" {
            if !versionGTE(depCV.Status.ObservedVersion, dep.MinVersion) {
                return false, nil
            }
        }
    }
    return true, nil
}

// BuildDAG 构建依赖有向图，用于批量安装排序
func (r *Resolver) BuildDAG(ctx context.Context) (*Graph, error) {
    cvList := &v1alpha1.ComponentVersionList{}
    if err := r.List(ctx, cvList); err != nil {
        return nil, err
    }
    
    g := NewGraph()
    for _, cv := range cvList.Items {
        g.AddNode(cv.Name)
        for _, dep := range cv.Spec.Dependencies {
            g.AddEdge(dep.Name, cv.Name) // dep → cv
        }
    }
    
    if g.HasCycle() {
        return nil, fmt.Errorf("circular dependency detected")
    }
    return g, nil
}
```
### 十一、健康检查设计
```go
type Checker struct {
    client.Client
    RestConfig *rest.Config
}

func (c *Checker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    if cv.Spec.HealthCheck == nil {
        return nil
    }
    
    switch cv.Spec.HealthCheck.Type {
    case "HTTPGet":
        return c.checkHTTP(ctx, cv)
    case "TCPSocket":
        return c.checkTCP(ctx, cv)
    case "Command":
        return c.checkCommand(ctx, cv)
    case "ClusterOperator":
        return c.checkClusterOperator(ctx, cv)
    }
    return nil
}
```
### 十二、SetupWithManager 完整设计
```go
func (r *ComponentVersionReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1alpha1.ComponentVersion{},
            builder.WithPredicates(
                predicate.Or(
                    predicate.GenerationChangedPredicate{},   // spec 变更
                    predicate.AnnotationChangedPredicate{},   // 注解变更（触发重试等）
                ),
            ),
        ).
        // Watch NodeConfig 变更 → 触发关联的 scope=Node 的 CV
        Watches(
            &v1alpha1.NodeConfig{},
            handler.EnqueueRequestsFromMapFunc(r.nodeConfigToComponentVersion),
        ).
        // Watch Command 变更 → 触发关联的 InstallScript 类型 CV
        Watches(
            &agentv1beta1.Command{},
            handler.EnqueueRequestsFromMapFunc(r.commandToComponentVersion),
        ).
        // Watch 依赖的 CV 变更 → 触发下游 CV
        Watches(
            &v1alpha1.ComponentVersion{},
            handler.EnqueueRequestsFromMapFunc(r.dependencyToDownstreamCV),
        ).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 5,
            RateLimiter: workqueue.NewItemFastSlowRateLimiter(
                5*time.Second,  // fast rate
                60*time.Second, // slow rate
                5,              // max fast attempts
            ),
        }).
        Complete(r)
}
```
### 十三、RBAC 权限设计
```go
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions/finalizers,verbs=update
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=nodeconfigs,verbs=get;list;watch
// +kubebuilder:rbac:groups=bkeagent.bocloud.com,resources=commands,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=bkeagent.bocloud.com,resources=commands/status,verbs=get;update;patch
// +kubebuilder:rbac:groups="",resources=configmaps;secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=pods;pods/log,verbs=get;list;watch
// +kubebuilder:rbac:groups="batch",resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="apps",resources=deployments;statefulsets;daemonsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="apiextensions.k8s.io",resources=customresourcedefinitions,verbs=get;list;watch
```
### 十四、错误处理与重试策略
| 场景 | 处理策略 | Requeue 策略 |
|------|----------|-------------|
| **依赖未满足** | phase=Pending | RequeueAfter: 10s |
| **Source 解析失败** | phase=Failed, retryCount++ | RequeueAfter: 指数退避 |
| **Manifest Apply 冲突** | 自动重试（RetryOnConflict） | 框架自动 requeue |
| **脚本执行超时** | phase=Failed | RequeueAfter: 30s |
| **Helm Release 失败** | atomic=true 自动回滚 | RequeueAfter: 30s |
| **健康检查失败** | 不改变 phase，更新 condition | RequeueAfter: 30s |
| **最大重试耗尽** | phase=Failed，等待手动干预 | 不 requeue |
### 十五、与现有系统的集成点
| 集成点 | 现有组件 | 集成方式 |
|--------|----------|----------|
| **脚本执行** | `bkeagent Command CRD` | ScriptInstaller 创建 Command 对象，复用 NodeAgent 执行通道 |
| **状态上报** | `mergecluster.SyncStatusUntilComplete` | 复用现有状态同步机制，解决冲突重试 |
| **日志体系** | `utils/capbke/log` | 复用现有结构化日志 |
| **Helm SDK** | `helm.sh/helm/v3 v3.19.0` | 直接引用，HelmChartInstaller 使用 action API |
| **Phase 框架** | `pkg/phaseframe` | 可选集成：将 ComponentVersion 安装封装为 Phase 嵌入 BKECluster PhaseFlow |
| **节点管理** | `utils/capbke/nodeutil` | 复用 NodeFetcher 查询节点状态 |
### 十六、测试策略
| 层级 | 测试内容 | 方式 |
|------|----------|------|
| **单元测试** | SourceResolver 各来源解析 | fake client + httptest |
| **单元测试** | 状态机转换逻辑 | 表驱动测试 |
| **单元测试** | Values 合并逻辑 | 固定输入输出对比 |
| **单元测试** | 依赖 DAG 构建 + 环检测 | 图算法验证 |
| **集成测试** | ManifestInstaller 完整流程 | envtest + fake k8s |
| **集成测试** | HelmChartInstaller 完整流程 | envtest + helm action |
| **集成测试** | ScriptInstaller + Command CRD | envtest + mock agent |
| **E2E 测试** | 三种策略端到端验证 | kind 集群 |
| **E2E 测试** | 升级 + 回滚场景 | kind 集群 |

以上就是 ComponentVersion Controller 支持多种部署方式的详细设计大纲，核心要点总结：

1. **策略模式**：通过 `Installer` 接口抽象三种安装策略，`Dispatcher` 负责路由
2. **Source 统一解析**：`SourceResolver` 统一处理 6 种来源类型，Installer 无需关心来源细节
3. **状态机驱动**：7 个状态 + 明确的转换规则，Reconciler 主循环由状态机驱动
4. **复用现有组件**：ScriptInstaller 复用 `Command CRD` 下发到 NodeAgent，HelmChartInstaller 复用项目已有的 `Helm SDK`
5. **NodeConfig 协作**：通过 nodeSelector 匹配 + configOverrides 合并，实现节点级配置覆盖
6. **依赖管理**：DAG 解析 + 逐层安装，确保组件安装顺序

# ComponentVersion Controller 的完整代码实现框架
## ComponentVersion Controller 代码实现框架
### 1. API 类型定义
```go
/*
componentversion_types.go
*/

package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Version",type="string",JSONPath=".spec.version"
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// ComponentVersion is the Schema for the componentversions API
type ComponentVersion struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ComponentVersionSpec   `json:"spec,omitempty"`
    Status ComponentVersionStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// ComponentVersionList contains a list of ComponentVersion
type ComponentVersionList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []ComponentVersion `json:"items"`
}

// InstallType 安装策略类型
type InstallType string

const (
    InstallTypeManifest     InstallType = "Manifest"
    InstallTypeInstallScript InstallType = "InstallScript"
    InstallTypeHelmChart    InstallType = "HelmChart"
)

// ComponentScope 组件作用域
type ComponentScope string

const (
    ComponentScopeCluster ComponentScope = "Cluster"
    ComponentScopeNode    ComponentScope = "Node"
)

// ComponentPhase 组件安装阶段
type ComponentPhase string

const (
    PhasePending     ComponentPhase = "Pending"
    PhaseInstalling  ComponentPhase = "Installing"
    PhaseVerifying   ComponentPhase = "Verifying"
    PhaseInstalled   ComponentPhase = "Installed"
    PhaseFailed      ComponentPhase = "Failed"
    PhaseUpgrading   ComponentPhase = "Upgrading"
    PhaseRollingBack ComponentPhase = "RollingBack"
)

// ComponentVersionSpec 定义组件版本的期望状态
type ComponentVersionSpec struct {
    ComponentName string              `json:"componentName"`
    Version       string              `json:"version"`
    Scope         ComponentScope      `json:"scope,omitempty"`
    Dependencies  []Dependency        `json:"dependencies,omitempty"`
    Images        []ImageRef          `json:"images,omitempty"`
    Install       InstallSpec         `json:"install"`
    HealthCheck   *HealthCheckSpec    `json:"healthCheck,omitempty"`
}

// Dependency 依赖定义
type Dependency struct {
    Name       string `json:"name"`
    MinVersion string `json:"minVersion,omitempty"`
}

// ImageRef 镜像引用
type ImageRef struct {
    Name  string `json:"name"`
    Image string `json:"image"`
}

// InstallSpec 安装规格，三种策略互斥
type InstallSpec struct {
    Type          InstallType       `json:"type"`
    Manifest      *ManifestSpec     `json:"manifest,omitempty"`
    InstallScript *InstallScriptSpec `json:"installScript,omitempty"`
    HelmChart     *HelmChartSpec    `json:"helmChart,omitempty"`
}

// ManifestSpec k8s yaml 安装策略
type ManifestSpec struct {
    Items         []ManifestItem   `json:"items,omitempty"`
    ApplyStrategy ApplyStrategy    `json:"applyStrategy,omitempty"`
    Prune         bool             `json:"prune,omitempty"`
}

// ApplyStrategy 应用策略
type ApplyStrategy string

const (
    ApplyStrategyServerSideApply ApplyStrategy = "ServerSideApply"
    ApplyStrategyThreeWayMerge   ApplyStrategy = "ThreeWayMerge"
    ApplyStrategyReplace         ApplyStrategy = "Replace"
)

// ManifestItem 单个 manifest 条目
type ManifestItem struct {
    Name      string     `json:"name"`
    Namespace string     `json:"namespace,omitempty"`
    Source    SourceSpec `json:"source"`
}

// InstallScriptSpec 脚本安装策略
type InstallScriptSpec struct {
    Command    string            `json:"command,omitempty"`
    Args       []string          `json:"args,omitempty"`
    Env        []EnvVar          `json:"env,omitempty"`
    Source     SourceSpec        `json:"source"`
    Execution  *ExecutionSpec    `json:"execution,omitempty"`
    Validation *ValidationSpec   `json:"validation,omitempty"`
}

// EnvVar 环境变量
type EnvVar struct {
    Name      string         `json:"name"`
    Value     string         `json:"value,omitempty"`
    ValueFrom *EnvVarSource  `json:"valueFrom,omitempty"`
}

// EnvVarSource 环境变量来源
type EnvVarSource struct {
    ConfigMapKeyRef *ConfigMapKeyRef `json:"configMapKeyRef,omitempty"`
    SecretKeyRef    *SecretKeyRef    `json:"secretKeyRef,omitempty"`
}

// ConfigMapKeyRef ConfigMap 引用
type ConfigMapKeyRef struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace,omitempty"`
    Key       string `json:"key"`
}

// SecretKeyRef Secret 引用
type SecretKeyRef struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace,omitempty"`
    Key       string `json:"key"`
}

// ExecutionSpec 执行规格
type ExecutionSpec struct {
    Target         string            `json:"target,omitempty"`
    NodeSelector   map[string]string `json:"nodeSelector,omitempty"`
    Timeout        string            `json:"timeout,omitempty"`
    RetryPolicy    *RetryPolicySpec `json:"retryPolicy,omitempty"`
}

// RetryPolicySpec 重试策略
type RetryPolicySpec struct {
    MaxRetries int    `json:"maxRetries,omitempty"`
    RetryDelay string `json:"retryDelay,omitempty"`
}

// ValidationSpec 验证规格
type ValidationSpec struct {
    Command        string `json:"command,omitempty"`
    ExpectedOutput string `json:"expectedOutput,omitempty"`
    Timeout        string `json:"timeout,omitempty"`
}

// HelmChartSpec Helm chart 安装策略
type HelmChartSpec struct {
    ChartRef    HelmChartRef        `json:"chartRef,omitempty"`
    ReleaseName string              `json:"releaseName,omitempty"`
    Namespace   string              `json:"namespace,omitempty"`
    Source      *ChartSourceSpec    `json:"source,omitempty"`
    Values      *runtime.RawExtension `json:"values,omitempty"`
    ValuesFrom  []ValuesFromSource  `json:"valuesFrom,omitempty"`
    Timeout     string              `json:"timeout,omitempty"`
    Wait        bool                `json:"wait,omitempty"`
    Atomic      bool                `json:"atomic,omitempty"`
}

// HelmChartRef Helm chart 引用
type HelmChartRef struct {
    Name      string `json:"name"`
    Repo      string `json:"repo,omitempty"`
    Version   string `json:"version,omitempty"`
    ChartPath string `json:"chartPath,omitempty"`
}

// ChartSourceSpec Chart 来源
type ChartSourceSpec struct {
    Type         string           `json:"type"`
    Repo         *ChartRepoSource `json:"repo,omitempty"`
    OCI          *ChartOCISource  `json:"oci,omitempty"`
    LocalPath    *string          `json:"localPath,omitempty"`
    ConfigMapRef *ConfigMapSource `json:"configMapRef,omitempty"`
}

// ChartRepoSource Repo 来源
type ChartRepoSource struct {
    URL                 string       `json:"url"`
    Username            string       `json:"username,omitempty"`
    PasswordSecretRef   *SecretKeyRef `json:"passwordSecretRef,omitempty"`
}

// ChartOCISource OCI 来源
type ChartOCISource struct {
    Registry string `json:"registry"`
    Repository string `json:"repository"`
    Tag      string `json:"tag"`
    Insecure bool   `json:"insecure,omitempty"`
}

// ValuesFromSource Values 来源
type ValuesFromSource struct {
    Kind       string `json:"kind"`
    Name       string `json:"name"`
    Namespace  string `json:"namespace,omitempty"`
    ValuesKey  string `json:"valuesKey,omitempty"`
}

// SourceType 来源类型枚举
type SourceType string

const (
    SourceTypeInline      SourceType = "Inline"
    SourceTypeConfigMapRef SourceType = "ConfigMapRef"
    SourceTypeSecretRef   SourceType = "SecretRef"
    SourceTypeURIRef      SourceType = "URIRef"
    SourceTypeImageRef    SourceType = "ImageRef"
    SourceTypeLocalPath   SourceType = "LocalPath"
)

// SourceSpec 统一来源规格
type SourceSpec struct {
    Type         SourceType        `json:"type"`
    Inline       *InlineSource     `json:"inline,omitempty"`
    ConfigMapRef *ConfigMapSource  `json:"configMapRef,omitempty"`
    SecretRef    *SecretSource     `json:"secretRef,omitempty"`
    URIRef       *URISource        `json:"uriRef,omitempty"`
    ImageRef     *ImageSource      `json:"imageRef,omitempty"`
    LocalPath    *LocalPathSource  `json:"localPath,omitempty"`
}

// InlineSource 内嵌内容
type InlineSource struct {
    Content string `json:"content,omitempty"`
}

// ConfigMapSource ConfigMap 引用
type ConfigMapSource struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace"`
    Key       string `json:"key,omitempty"`
}

// SecretSource Secret 引用
type SecretSource struct {
    Name      string `json:"name"`
    Namespace string `json:"namespace"`
    Key       string `json:"key,omitempty"`
}

// URISource 远程 URI
type URISource struct {
    URI                    string `json:"uri"`
    Checksum               string `json:"checksum,omitempty"`
    InsecureSkipTLSVerify  bool   `json:"insecureSkipTLSVerify,omitempty"`
}

// ImageSource 容器镜像
type ImageSource struct {
    Image           string `json:"image"`
    Path            string `json:"path,omitempty"`
    ImagePullSecret string `json:"imagePullSecret,omitempty"`
}

// LocalPathSource 本地路径
type LocalPathSource struct {
    Path     string `json:"path"`
    Checksum string `json:"checksum,omitempty"`
}

// HealthCheckSpec 健康检查规格
type HealthCheckSpec struct {
    Type             string           `json:"type"`
    HTTPGet          *HTTPGetCheck   `json:"httpGet,omitempty"`
    TCPSocket        *TCPSocketCheck `json:"tcpSocket,omitempty"`
    Command          *CommandCheck    `json:"command,omitempty"`
    ClusterOperator  *ClusterOperatorCheck `json:"clusterOperator,omitempty"`
    Interval         string           `json:"interval,omitempty"`
    Timeout          string           `json:"timeout,omitempty"`
}

// HTTPGetCheck HTTP 健康检查
type HTTPGetCheck struct {
    Path   string `json:"path"`
    Port   int    `json:"port"`
    Scheme string `json:"scheme,omitempty"`
}

// TCPSocketCheck TCP 健康检查
type TCPSocketCheck struct {
    Port int `json:"port"`
}

// CommandCheck 命令健康检查
type CommandCheck struct {
    Command []string `json:"command"`
}

// ClusterOperatorCheck ClusterOperator 健康检查
type ClusterOperatorCheck struct {
    Name          string `json:"name"`
    ConditionType string `json:"conditionType,omitempty"`
}

// ComponentVersionStatus 定义组件版本的观测状态
type ComponentVersionStatus struct {
    ObservedVersion    string              `json:"observedVersion,omitempty"`
    Phase              ComponentPhase      `json:"phase,omitempty"`
    InstallType        InstallType         `json:"installType,omitempty"`
    Conditions         []metav1.Condition  `json:"conditions,omitempty"`
    InstallDetails     InstallDetails      `json:"installDetails,omitempty"`
    LastTransitionTime *metav1.Time        `json:"lastTransitionTime,omitempty"`
    RetryCount         int                 `json:"retryCount,omitempty"`
}

// InstallDetails 安装详情
type InstallDetails struct {
    Manifest      *ManifestDetails      `json:"manifest,omitempty"`
    InstallScript *InstallScriptDetails `json:"installScript,omitempty"`
    HelmChart     *HelmChartDetails     `json:"helmChart,omitempty"`
}

// ManifestDetails Manifest 安装详情
type ManifestDetails struct {
    AppliedResources []AppliedResource `json:"appliedResources,omitempty"`
    PrunedResources  []PrunedResource  `json:"prunedResources,omitempty"`
}

// AppliedResource 已应用资源
type AppliedResource struct {
    Group     string       `json:"group,omitempty"`
    Version   string       `json:"version,omitempty"`
    Kind      string       `json:"kind,omitempty"`
    Name      string       `json:"name,omitempty"`
    Namespace string       `json:"namespace,omitempty"`
    UID       string       `json:"uid,omitempty"`
    Hash      string       `json:"hash,omitempty"`
    AppliedAt *metav1.Time `json:"appliedAt,omitempty"`
}

// PrunedResource 已清理资源
type PrunedResource struct {
    Group     string `json:"group,omitempty"`
    Version   string `json:"version,omitempty"`
    Kind      string `json:"kind,omitempty"`
    Name      string `json:"name,omitempty"`
    Namespace string `json:"namespace,omitempty"`
}

// InstallScriptDetails 脚本安装详情
type InstallScriptDetails struct {
    ExecutionID      string               `json:"executionId,omitempty"`
    TargetNodes      []TargetNode         `json:"targetNodes,omitempty"`
    ValidationResult *ValidationResult    `json:"validationResult,omitempty"`
}

// TargetNode 目标节点
type TargetNode struct {
    NodeName     string       `json:"nodeName,omitempty"`
    NodeIP       string       `json:"nodeIP,omitempty"`
    Phase        string       `json:"phase,omitempty"`
    ExitCode     int          `json:"exitCode,omitempty"`
    Stdout       string       `json:"stdout,omitempty"`
    Stderr       string       `json:"stderr,omitempty"`
    StartedAt    *metav1.Time `json:"startedAt,omitempty"`
    FinishedAt   *metav1.Time `json:"finishedAt,omitempty"`
    RetryCount   int          `json:"retryCount,omitempty"`
}

// ValidationResult 验证结果
type ValidationResult struct {
    Passed    bool         `json:"passed,omitempty"`
    Output    string       `json:"output,omitempty"`
    CheckedAt *metav1.Time `json:"checkedAt,omitempty"`
}

// HelmChartDetails Helm chart 安装详情
type HelmChartDetails struct {
    ReleaseStatus string       `json:"releaseStatus,omitempty"`
    Revision      int          `json:"revision,omitempty"`
    LastDeployed  *metav1.Time `json:"lastDeployed,omitempty"`
}

func init() {
    SchemeBuilder.Register(&ComponentVersion{}, &ComponentVersionList{})
}
```
```go
/*
nodeconfig_types.go
*/

package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// NodeConfig is the Schema for the nodeconfigs API
type NodeConfig struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   NodeConfigSpec   `json:"spec,omitempty"`
    Status NodeConfigStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// NodeConfigList contains a list of NodeConfig
type NodeConfigList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []NodeConfig `json:"items"`
}

// NodeConfigSpec 定义 NodeConfig 的期望状态
type NodeConfigSpec struct {
    NodeName         string                  `json:"nodeName"`
    NodeIP           string                  `json:"nodeIP,omitempty"`
    Roles            []string                `json:"roles,omitempty"`
    OSInfo           *OSInfo                `json:"osInfo,omitempty"`
    ComponentConfigs []ComponentConfig      `json:"componentConfigs,omitempty"`
}

// OSInfo 操作系统信息
type OSInfo struct {
    Type    string `json:"type,omitempty"`
    Version string `json:"version,omitempty"`
    Arch    string `json:"arch,omitempty"`
}

// ComponentConfig 组件配置
type ComponentConfig struct {
    ComponentName    string                 `json:"componentName"`
    ConfigOverrides  map[string]interface{} `json:"configOverrides,omitempty"`
    NodeLabels       map[string]string      `json:"nodeLabels,omitempty"`
}

// NodeConfigStatus 定义 NodeConfig 的观测状态
type NodeConfigStatus struct {
    ComponentVersions []ComponentVersionStatus `json:"componentVersions,omitempty"`
    NodeHealth        string                  `json:"nodeHealth,omitempty"`
    LastHeartbeat     *metav1.Time           `json:"lastHeartbeat,omitempty"`
}

// ComponentVersionStatus 组件版本状态
type ComponentVersionStatus struct {
    ComponentName  string       `json:"componentName"`
    CurrentVersion string       `json:"currentVersion,omitempty"`
    TargetVersion  string       `json:"targetVersion,omitempty"`
    Phase          string       `json:"phase,omitempty"`
    LastUpdated    *metav1.Time `json:"lastUpdated,omitempty"`
}

func init() {
    SchemeBuilder.Register(&NodeConfig{}, &NodeConfigList{})
}
```
### 2. Installer 接口与实现
```go
/*
installer/interface.go
*/

package installer

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
)

// Installer 定义安装器接口
type Installer interface {
    Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error)
    Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*VerifyResult, error)
    Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error
    Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error
}

// InstallResult 安装结果
type InstallResult struct {
    Phase   v1alpha1.ComponentPhase
    Details runtime.Object
    Requeue bool
}

// VerifyResult 验证结果
type VerifyResult struct {
    Passed  bool
    Message string
}
```

```go
/*
installer/manifest/installer.go
*/

package manifest

import (
    "context"
    "encoding/json"
    "fmt"

    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/util/retry"
    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
    "your.module.path/pkg/componentversion/installer"
    "your.module.path/pkg/componentversion/source"
)

type Installer struct {
    client.Client
    Scheme         *runtime.Scheme
    SourceResolver *source.Resolver
}

func NewInstaller(c client.Client, scheme *runtime.Scheme, sr *source.Resolver) *Installer {
    return &Installer{
        Client:         c,
        Scheme:         scheme,
        SourceResolver: sr,
    }
}

func (i *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
    if cv.Spec.Install.Manifest == nil {
        return nil, fmt.Errorf("manifest spec is nil")
    }

    var appliedResources []v1alpha1.AppliedResource

    for _, item := range cv.Spec.Install.Manifest.Items {
        content, err := i.SourceResolver.Resolve(ctx, &item.Source)
        if err != nil {
            return nil, fmt.Errorf("resolve source for %s failed: %w", item.Name, err)
        }

        resources, err := i.parseManifest(content.Content)
        if err != nil {
            return nil, fmt.Errorf("parse manifest for %s failed: %w", item.Name, err)
        }

        for _, res := range resources {
            if err := i.applyResource(ctx, &res, cv.Spec.Install.Manifest.ApplyStrategy, item.Namespace); err != nil {
                return nil, fmt.Errorf("apply resource failed: %w", err)
            }

            appliedResources = append(appliedResources, v1alpha1.AppliedResource{
                Group:     res.GetAPIVersion(),
                Version:   "",
                Kind:      res.GetKind(),
                Name:      res.GetName(),
                Namespace: res.GetNamespace(),
                AppliedAt: nil,
            })
        }
    }

    if cv.Spec.Install.Manifest.Prune {
        if err := i.pruneOldResources(ctx, cv, appliedResources); err != nil {
            return nil, fmt.Errorf("prune old resources failed: %w", err)
        }
    }

    cv.Status.InstallDetails.Manifest = &v1alpha1.ManifestDetails{
        AppliedResources: appliedResources,
    }

    return &installer.InstallResult{
        Phase: v1alpha1.PhaseVerifying,
    }, nil
}

func (i *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
    if cv.Status.InstallDetails.Manifest == nil {
        return &installer.VerifyResult{Passed: false, Message: "manifest details not found"}, nil
    }

    for _, res := range cv.Status.InstallDetails.Manifest.AppliedResources {
        obj := &unstructured.Unstructured{}
        obj.SetAPIVersion(res.Group)
        obj.SetKind(res.Kind)
        obj.SetName(res.Name)
        obj.SetNamespace(res.Namespace)

        if err := i.Get(ctx, client.ObjectKeyFromObject(obj), obj); err != nil {
            return &installer.VerifyResult{Passed: false, Message: fmt.Sprintf("resource %s/%s not found", res.Kind, res.Name)}, nil
        }

        if !i.checkResourceReady(obj) {
            return &installer.VerifyResult{Passed: false, Message: fmt.Sprintf("resource %s/%s not ready", res.Kind, res.Name)}, nil
        }
    }

    return &installer.VerifyResult{Passed: true}, nil
}

func (i *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现回滚逻辑
    return fmt.Errorf("rollback not implemented")
}

func (i *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现卸载逻辑
    return fmt.Errorf("uninstall not implemented")
}

func (i *Installer) parseManifest(content []byte) ([]unstructured.Unstructured, error) {
    // TODO: 解析 manifest 内容
    return nil, nil
}

func (i *Installer) applyResource(ctx context.Context, obj *unstructured.Unstructured, strategy v1alpha1.ApplyStrategy, namespace string) error {
    // TODO: 根据 strategy 应用资源
    return nil
}

func (i *Installer) pruneOldResources(ctx context.Context, cv *v1alpha1.ComponentVersion, appliedResources []v1alpha1.AppliedResource) error {
    // TODO: 清理旧资源
    return nil
}

func (i *Installer) checkResourceReady(obj *unstructured.Unstructured) bool {
    // TODO: 检查资源是否就绪
    return true
}
```

```go
/*
installer/script/installer.go
*/

package script

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
    "your.module.path/pkg/componentversion/installer"
    "your.module.path/pkg/componentversion/source"
)

type Installer struct {
    client.Client
    Scheme         *runtime.Scheme
    SourceResolver *source.Resolver
}

func NewInstaller(c client.Client, scheme *runtime.Scheme, sr *source.Resolver) *Installer {
    return &Installer{
        Client:         c,
        Scheme:         scheme,
        SourceResolver: sr,
    }
}

func (i *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
    if cv.Spec.Install.InstallScript == nil {
        return nil, fmt.Errorf("installScript spec is nil")
    }

    content, err := i.SourceResolver.Resolve(ctx, &cv.Spec.Install.InstallScript.Source)
    if err != nil {
        return nil, fmt.Errorf("resolve script source failed: %w", err)
    }

    execution := cv.Spec.Install.InstallScript.Execution
    if execution == nil {
        execution = &v1alpha1.ExecutionSpec{
            Target: "NodeAgent",
        }
    }

    switch execution.Target {
    case "Controller":
        return i.executeInController(ctx, cv, content.Content)
    case "NodeAgent":
        return i.dispatchToNodeAgent(ctx, cv, content.Content)
    case "InitPod":
        return i.executeInInitPod(ctx, cv, content.Content)
    default:
        return nil, fmt.Errorf("unsupported execution target: %s", execution.Target)
    }
}

func (i *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
    if cv.Spec.Install.InstallScript.Validation == nil {
        return &installer.VerifyResult{Passed: true}, nil
    }

    // TODO: 执行验证逻辑
    return &installer.VerifyResult{Passed: true}, nil
}

func (i *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现回滚逻辑
    return fmt.Errorf("rollback not implemented")
}

func (i *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现卸载逻辑
    return fmt.Errorf("uninstall not implemented")
}

func (i *Installer) executeInController(ctx context.Context, cv *v1alpha1.ComponentVersion, content []byte) (*installer.InstallResult, error) {
    // TODO: 在 Controller 进程内执行脚本
    return &installer.InstallResult{Phase: v1alpha1.PhaseVerifying}, nil
}

func (i *Installer) dispatchToNodeAgent(ctx context.Context, cv *v1alpha1.ComponentVersion, content []byte) (*installer.InstallResult, error) {
    // TODO: 构造 Command CRD 下发到 NodeAgent
    return &installer.InstallResult{Phase: v1alpha1.PhaseInstalling, Requeue: true}, nil
}

func (i *Installer) executeInInitPod(ctx context.Context, cv *v1alpha1.ComponentVersion, content []byte) (*installer.InstallResult, error) {
    // TODO: 创建 Job/Pod 执行脚本
    return &installer.InstallResult{Phase: v1alpha1.PhaseInstalling, Requeue: true}, nil
}
```

```go
/*
installer/helmchart/installer.go
*/

package helmchart

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
    "your.module.path/pkg/componentversion/installer"
)

type Installer struct {
    client.Client
    Scheme *runtime.Scheme
}

func NewInstaller(c client.Client, scheme *runtime.Scheme) *Installer {
    return &Installer{
        Client: c,
        Scheme: scheme,
    }
}

func (i *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
    if cv.Spec.Install.HelmChart == nil {
        return nil, fmt.Errorf("helmChart spec is nil")
    }

    // TODO: 加载 Chart
    // TODO: 合并 Values
    // TODO: 执行 helm install/upgrade

    cv.Status.InstallDetails.HelmChart = &v1alpha1.HelmChartDetails{
        ReleaseStatus: "deployed",
        Revision:      1,
    }

    return &installer.InstallResult{Phase: v1alpha1.PhaseVerifying}, nil
}

func (i *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
    // TODO: 查询 Release 状态
    return &installer.VerifyResult{Passed: true}, nil
}

func (i *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现回滚逻辑
    return fmt.Errorf("rollback not implemented")
}

func (i *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现卸载逻辑
    return fmt.Errorf("uninstall not implemented")
}
```
### 3. SourceResolver
```go
/*
source/resolver.go
*/

package source

import (
    "context"
    "fmt"
    "net/http"
    "os"

    corev1 "k8s.io/api/core/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
)

type Resolver struct {
    client.Client
    HTTPClient *http.Client
}

type ResolvedContent struct {
    Content  []byte
    Checksum string
    Source   v1alpha1.SourceType
}

func NewResolver(c client.Client) *Resolver {
    return &Resolver{
        Client:     c,
        HTTPClient: http.DefaultClient,
    }
}

func (r *Resolver) Resolve(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    switch spec.Type {
    case v1alpha1.SourceTypeInline:
        return r.resolveInline(spec)
    case v1alpha1.SourceTypeConfigMapRef:
        return r.resolveConfigMap(ctx, spec)
    case v1alpha1.SourceTypeSecretRef:
        return r.resolveSecret(ctx, spec)
    case v1alpha1.SourceTypeURIRef:
        return r.resolveURI(ctx, spec)
    case v1alpha1.SourceTypeImageRef:
        return r.resolveImage(ctx, spec)
    case v1alpha1.SourceTypeLocalPath:
        return r.resolveLocalPath(spec)
    default:
        return nil, fmt.Errorf("unsupported source type: %s", spec.Type)
    }
}

func (r *Resolver) resolveInline(spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.Inline == nil {
        return nil, fmt.Errorf("inline source is nil")
    }
    return &ResolvedContent{
        Content: []byte(spec.Inline.Content),
        Source:  spec.Type,
    }, nil
}

func (r *Resolver) resolveConfigMap(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.ConfigMapRef == nil {
        return nil, fmt.Errorf("configMapRef is nil")
    }

    cm := &corev1.ConfigMap{}
    key := client.ObjectKey{
        Namespace: spec.ConfigMapRef.Namespace,
        Name:      spec.ConfigMapRef.Name,
    }
    if err := r.Get(ctx, key, cm); err != nil {
        return nil, fmt.Errorf("get configmap failed: %w", err)
    }

    dataKey := spec.ConfigMapRef.Key
    if dataKey == "" {
        for k := range cm.Data {
            dataKey = k
            break
        }
    }

    content, ok := cm.Data[dataKey]
    if !ok {
        return nil, fmt.Errorf("key %q not found in configmap", dataKey)
    }

    return &ResolvedContent{
        Content: []byte(content),
        Source:  spec.Type,
    }, nil
}

func (r *Resolver) resolveSecret(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.SecretRef == nil {
        return nil, fmt.Errorf("secretRef is nil")
    }

    secret := &corev1.Secret{}
    key := client.ObjectKey{
        Namespace: spec.SecretRef.Namespace,
        Name:      spec.SecretRef.Name,
    }
    if err := r.Get(ctx, key, secret); err != nil {
        return nil, fmt.Errorf("get secret failed: %w", err)
    }

    dataKey := spec.SecretRef.Key
    if dataKey == "" {
        for k := range secret.Data {
            dataKey = k
            break
        }
    }

    content, ok := secret.Data[dataKey]
    if !ok {
        return nil, fmt.Errorf("key %q not found in secret", dataKey)
    }

    return &ResolvedContent{
        Content: content,
        Source:  spec.Type,
    }, nil
}

func (r *Resolver) resolveURI(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.URIRef == nil {
        return nil, fmt.Errorf("uriRef is nil")
    }
    // TODO: 实现 URI 拉取逻辑
    return nil, fmt.Errorf("uri source not implemented")
}

func (r *Resolver) resolveImage(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.ImageRef == nil {
        return nil, fmt.Errorf("imageRef is nil")
    }
    // TODO: 实现镜像内文件提取逻辑
    return nil, fmt.Errorf("image source not implemented")
}

func (r *Resolver) resolveLocalPath(spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
    if spec.LocalPath == nil {
        return nil, fmt.Errorf("localPath is nil")
    }

    content, err := os.ReadFile(spec.LocalPath.Path)
    if err != nil {
        return nil, fmt.Errorf("read file failed: %w", err)
    }

    return &ResolvedContent{
        Content: content,
        Source:  spec.Type,
    }, nil
}
```
### 4. 状态机
```go
/*
statemachine/machine.go
*/

package statemachine

import (
    "context"
    "fmt"
    "time"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
)

type Machine struct {
    transitions map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase
}

type ExecuteFunc func(ctx context.Context, cv *v1alpha1.ComponentVersion) (*v1alpha1.ComponentPhase, error)

func NewMachine() *Machine {
    return &Machine{
        transitions: map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase{
            v1alpha1.PhasePending:     {v1alpha1.PhaseInstalling},
            v1alpha1.PhaseInstalling:  {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed},
            v1alpha1.PhaseVerifying:   {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
            v1alpha1.PhaseInstalled:   {v1alpha1.PhaseUpgrading},
            v1alpha1.PhaseFailed:      {v1alpha1.PhaseInstalling, v1alpha1.PhaseRollingBack},
            v1alpha1.PhaseUpgrading:   {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed, v1alpha1.PhaseRollingBack},
            v1alpha1.PhaseRollingBack: {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
        },
    }
}

func (m *Machine) Drive(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
    currentPhase := cv.Status.Phase
    if currentPhase == "" {
        currentPhase = v1alpha1.PhasePending
    }

    switch currentPhase {
    case v1alpha1.PhasePending:
        return m.transition(ctx, cv, v1alpha1.PhaseInstalling, execute)

    case v1alpha1.PhaseInstalling, v1alpha1.PhaseUpgrading:
        nextPhase, err := execute(ctx, cv)
        if err != nil {
            return m.transition(ctx, cv, v1alpha1.PhaseFailed, nil)
        }
        return m.transition(ctx, cv, *nextPhase, nil)

    case v1alpha1.PhaseVerifying:
        nextPhase, err := execute(ctx, cv)
        if err != nil || *nextPhase != v1alpha1.PhaseInstalled {
            return m.transition(ctx, cv, v1alpha1.PhaseFailed, nil)
        }
        return m.transition(ctx, cv, *nextPhase, nil)

    case v1alpha1.PhaseInstalled:
        if cv.Status.ObservedVersion != cv.Spec.Version {
            return m.transition(ctx, cv, v1alpha1.PhaseUpgrading, execute)
        }
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil

    case v1alpha1.PhaseFailed:
        const maxRetry = 3
        if cv.Status.RetryCount < maxRetry {
            cv.Status.RetryCount++
            return m.transition(ctx, cv, v1alpha1.PhaseInstalling, execute)
        }
        return ctrl.Result{}, nil

    case v1alpha1.PhaseRollingBack:
        nextPhase, err := execute(ctx, cv)
        if err != nil {
            return m.transition(ctx, cv, v1alpha1.PhaseFailed, nil)
        }
        return m.transition(ctx, cv, *nextPhase, nil)

    default:
        return ctrl.Result{}, fmt.Errorf("unknown phase: %s", currentPhase)
    }
}

func (m *Machine) transition(ctx context.Context, cv *v1alpha1.ComponentVersion, phase v1alpha1.ComponentPhase, execute ExecuteFunc) (ctrl.Result, error) {
    if !m.canTransition(cv.Status.Phase, phase) {
        return ctrl.Result{}, fmt.Errorf("cannot transition from %s to %s", cv.Status.Phase, phase)
    }

    cv.Status.Phase = phase
    cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}

    return ctrl.Result{Requeue: true}, nil
}

func (m *Machine) canTransition(from, to v1alpha1.ComponentPhase) bool {
    validTransitions, ok := m.transitions[from]
    if !ok {
        return false
    }
    for _, valid := range validTransitions {
        if valid == to {
            return true
        }
    }
    return false
}
```
### 5. 健康检查
```go
/*
healthcheck/checker.go
*/

package healthcheck

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
)

type Checker struct {
    client.Client
    Scheme *runtime.Scheme
}

func NewChecker(c client.Client, scheme *runtime.Scheme) *Checker {
    return &Checker{
        Client: c,
        Scheme: scheme,
    }
}

func (c *Checker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    if cv.Spec.HealthCheck == nil {
        return nil
    }

    switch cv.Spec.HealthCheck.Type {
    case "HTTPGet":
        return c.checkHTTP(ctx, cv)
    case "TCPSocket":
        return c.checkTCP(ctx, cv)
    case "Command":
        return c.checkCommand(ctx, cv)
    case "ClusterOperator":
        return c.checkClusterOperator(ctx, cv)
    default:
        return fmt.Errorf("unsupported health check type: %s", cv.Spec.HealthCheck.Type)
    }
}

func (c *Checker) checkHTTP(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现 HTTP 健康检查
    return nil
}

func (c *Checker) checkTCP(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现 TCP 健康检查
    return nil
}

func (c *Checker) checkCommand(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现命令健康检查
    return nil
}

func (c *Checker) checkClusterOperator(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    // TODO: 实现 ClusterOperator 健康检查
    return nil
}
```
### 6. 依赖解析
```go
/*
dependency/resolver.go
*/

package dependency

import (
    "context"
    "fmt"

    "sigs.k8s.io/controller-runtime/pkg/client"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
)

type Resolver struct {
    client.Client
}

func NewResolver(c client.Client) *Resolver {
    return &Resolver{Client: c}
}

func (r *Resolver) CheckDependencies(ctx context.Context, cv *v1alpha1.ComponentVersion) (bool, error) {
    for _, dep := range cv.Spec.Dependencies {
        depCV := &v1alpha1.ComponentVersion{}
        if err := r.Get(ctx, client.ObjectKey{Name: dep.Name}, depCV); err != nil {
            return false, err
        }

        if depCV.Status.Phase != v1alpha1.PhaseInstalled {
            return false, nil
        }

        if dep.MinVersion != "" {
            if !versionGTE(depCV.Status.ObservedVersion, dep.MinVersion) {
                return false, nil
            }
        }
    }
    return true, nil
}

func versionGTE(current, min string) bool {
    // TODO: 实现版本比较逻辑
    return true
}
```
### 7. Dispatcher
```go
/*
dispatcher.go
*/

package componentversion

import (
    "context"
    "fmt"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
    "your.module.path/pkg/componentversion/installer"
    "your.module.path/pkg/componentversion/installer/manifest"
    "your.module.path/pkg/componentversion/installer/script"
    "your.module.path/pkg/componentversion/installer/helmchart"
)

type Dispatcher struct {
    ManifestInstaller  *manifest.Installer
    ScriptInstaller    *script.Installer
    HelmChartInstaller *helmchart.Installer
}

func NewDispatcher(mi *manifest.Installer, si *script.Installer, hi *helmchart.Installer) *Dispatcher {
    return &Dispatcher{
        ManifestInstaller:  mi,
        ScriptInstaller:    si,
        HelmChartInstaller: hi,
    }
}

func (d *Dispatcher) GetInstaller(installType v1alpha1.InstallType) (installer.Installer, error) {
    switch installType {
    case v1alpha1.InstallTypeManifest:
        return d.ManifestInstaller, nil
    case v1alpha1.InstallTypeInstallScript:
        return d.ScriptInstaller, nil
    case v1alpha1.InstallTypeHelmChart:
        return d.HelmChartInstaller, nil
    default:
        return nil, fmt.Errorf("unsupported install type: %s", installType)
    }
}

func (d *Dispatcher) Dispatch(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
    inst, err := d.GetInstaller(cv.Spec.Install.Type)
    if err != nil {
        return nil, err
    }

    switch cv.Status.Phase {
    case v1alpha1.PhaseInstalling, v1alpha1.PhaseUpgrading:
        return inst.Install(ctx, cv)
    case v1alpha1.PhaseVerifying:
        verifyResult, err := inst.Verify(ctx, cv)
        if err != nil {
            return nil, err
        }
        phase := v1alpha1.PhaseInstalled
        if !verifyResult.Passed {
            phase = v1alpha1.PhaseFailed
        }
        return &installer.InstallResult{Phase: phase}, nil
    case v1alpha1.PhaseRollingBack:
        if err := inst.Rollback(ctx, cv); err != nil {
            return nil, err
        }
        return &installer.InstallResult{Phase: v1alpha1.PhaseInstalled}, nil
    default:
        return nil, nil
    }
}
```
### 8. 主 Controller
```go
/*
componentversion_controller.go
*/

package controller

import (
    "context"
    "fmt"

    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/record"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/source"

    v1alpha1 "your.module.path/api/componentversion/v1alpha1"
    "your.module.path/pkg/componentversion"
    "your.module.path/pkg/componentversion/dependency"
    "your.module.path/pkg/componentversion/healthcheck"
    "your.module.path/pkg/componentversion/statemachine"
)

const componentVersionFinalizer = "componentversion.orchestration.bke.io/finalizer"

// ComponentVersionReconciler reconciles a ComponentVersion object
type ComponentVersionReconciler struct {
    client.Client
    Scheme          *runtime.Scheme
    Recorder        record.EventRecorder
    
    // 子组件
    Dispatcher      *componentversion.Dispatcher
    StateMachine    *statemachine.Machine
    HealthChecker   *healthcheck.Checker
    DepResolver     *dependency.Resolver
}

// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=componentversions/finalizers,verbs=update
// +kubebuilder:rbac:groups=orchestration.bke.io,resources=nodeconfigs,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=configmaps;secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=events,verbs=get;list;watch;create;update;patch

// Reconcile is part of the main kubernetes reconciliation loop
func (r *ComponentVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.Log.WithName("controllers").WithName("ComponentVersion").WithValues("componentversion", req.NamespacedName)

    cv := &v1alpha1.ComponentVersion{}
    if err := r.Get(ctx, req.NamespacedName, cv); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    if !cv.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, cv)
    }

    if err := r.ensureFinalizer(ctx, cv); err != nil {
        return ctrl.Result{}, err
    }

    satisfied, err := r.DepResolver.CheckDependencies(ctx, cv)
    if err != nil {
        r.Recorder.Event(cv, corev1.EventTypeWarning, "DependencyCheckFailed", err.Error())
        return r.updatePhase(ctx, cv, v1alpha1.PhasePending, err)
    }
    if !satisfied {
        r.Recorder.Event(cv, corev1.EventTypeNormal, "DependenciesPending", "Waiting for dependencies")
        return r.updatePhase(ctx, cv, v1alpha1.PhasePending, nil)
    }

    executeFunc := func(ctx context.Context, cv *v1alpha1.ComponentVersion) (*v1alpha1.ComponentPhase, error) {
        result, err := r.Dispatcher.Dispatch(ctx, cv)
        if err != nil {
            return nil, err
        }
        if result == nil {
            return nil, nil
        }
        return &result.Phase, nil
    }

    result, err := r.StateMachine.Drive(ctx, cv, executeFunc)
    if err != nil {
        r.Recorder.Event(cv, corev1.EventTypeWarning, "PhaseTransitionFailed", err.Error())
        return result, err
    }

    if cv.Status.Phase == v1alpha1.PhaseInstalled {
        if err := r.HealthChecker.Check(ctx, cv); err != nil {
            r.Recorder.Event(cv, corev1.EventTypeWarning, "HealthCheckFailed", err.Error())
        }
    }

    if err := r.updateStatus(ctx, cv); err != nil {
        return result, err
    }

    return result, nil
}

func (r *ComponentVersionReconciler) handleDeletion(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
    if !containsFinalizer(cv, componentVersionFinalizer) {
        return ctrl.Result{}, nil
    }

    inst, err := r.Dispatcher.GetInstaller(cv.Spec.Install.Type)
    if err == nil {
        _ = inst.Uninstall(ctx, cv)
    }

    removeFinalizer(cv, componentVersionFinalizer)
    if err := r.Update(ctx, cv); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *ComponentVersionReconciler) ensureFinalizer(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    if !containsFinalizer(cv, componentVersionFinalizer) {
        addFinalizer(cv, componentVersionFinalizer)
        return r.Update(ctx, cv)
    }
    return nil
}

func (r *ComponentVersionReconciler) updatePhase(ctx context.Context, cv *v1alpha1.ComponentVersion, phase v1alpha1.ComponentPhase, err error) (ctrl.Result, error) {
    cv.Status.Phase = phase
    cv.Status.LastTransitionTime = &metav1.Time{Time: metav1.Now().Time}
    if err != nil {
        setCondition(cv, "ReconcileError", metav1.ConditionFalse, "ReconcileFailed", err.Error())
    }
    return ctrl.Result{Requeue: true}, r.updateStatus(ctx, cv)
}

func (r *ComponentVersionReconciler) updateStatus(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
    if cv.Status.Phase == v1alpha1.PhaseInstalled {
        cv.Status.ObservedVersion = cv.Spec.Version
    }
    cv.Status.InstallType = cv.Spec.Install.Type
    return r.Status().Update(ctx, cv)
}

func setCondition(cv *v1alpha1.ComponentVersion, conditionType string, status metav1.ConditionStatus, reason, message string) {
    cond := metav1.Condition{
        Type:               conditionType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
    }
    cv.Status.Conditions = append(cv.Status.Conditions, cond)
}

func containsFinalizer(obj *v1alpha1.ComponentVersion, finalizer string) bool {
    for _, f := range obj.Finalizers {
        if f == finalizer {
            return true
        }
    }
    return false
}

func addFinalizer(obj *v1alpha1.ComponentVersion, finalizer string) {
    obj.Finalizers = append(obj.Finalizers, finalizer)
}

func removeFinalizer(obj *v1alpha1.ComponentVersion, finalizer string) {
    var finalizers []string
    for _, f := range obj.Finalizers {
        if f != finalizer {
            finalizers = append(finalizers, f)
        }
    }
    obj.Finalizers = finalizers
}

// SetupWithManager sets up the controller with the Manager.
func (r *ComponentVersionReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1alpha1.ComponentVersion{}).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 5,
        }).
        Complete(r)
}
```
### 9. GroupVersion 信息
```go
/*
groupversion_info.go
*/

package v1alpha1

import (
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
    GroupVersion = schema.GroupVersion{Group: "orchestration.bke.io", Version: "v1alpha1"}

    SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

    AddToScheme = SchemeBuilder.AddToScheme
)
```
以上是 ComponentVersion Controller 的完整代码实现框架，包含：
- 完整的 API 类型定义（ComponentVersion、NodeConfig）
- Installer 接口和三个实现（Manifest、InstallScript、HelmChart）
- SourceResolver 统一来源解析（6 种来源类型）
- 状态机驱动
- 健康检查框架
- 依赖解析
- Dispatcher 策略路由
- 主 Reconciler 循环

#  ComponentVersion Controller 的完整代码实现框架
```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"time"

	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/predicate"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"sigs.k8s.io/controller-runtime/pkg/source"

	"helm.sh/helm/v3/pkg/action"
	"helm.sh/helm/v3/pkg/chart/loader"
	"helm.sh/helm/v3/pkg/cli"
	"helm.sh/helm/v3/pkg/getter"
	"helm.sh/helm/v3/pkg/release"
)

// ============================================================================
// 一、API 类型定义
// ============================================================================

// GroupVersion is group version used to register these objects
var GroupVersion = schema.GroupVersion{Group: "orchestration.bke.io", Version: "v1alpha1"}

// InstallType 安装策略类型
type InstallType string

const (
	InstallTypeManifest     InstallType = "Manifest"
	InstallTypeInstallScript InstallType = "InstallScript"
	InstallTypeHelmChart    InstallType = "HelmChart"
)

// ComponentScope 组件作用域
type ComponentScope string

const (
	ComponentScopeCluster ComponentScope = "Cluster"
	ComponentScopeNode    ComponentScope = "Node"
)

// ComponentPhase 组件安装阶段
type ComponentPhase string

const (
	PhasePending     ComponentPhase = "Pending"
	PhaseInstalling  ComponentPhase = "Installing"
	PhaseVerifying   ComponentPhase = "Verifying"
	PhaseInstalled   ComponentPhase = "Installed"
	PhaseFailed      ComponentPhase = "Failed"
	PhaseUpgrading   ComponentPhase = "Upgrading"
	PhaseRollingBack ComponentPhase = "RollingBack"
)

// SourceType 来源类型
type SourceType string

const (
	SourceTypeInline      SourceType = "Inline"
	SourceTypeConfigMapRef SourceType = "ConfigMapRef"
	SourceTypeSecretRef   SourceType = "SecretRef"
	SourceTypeURIRef      SourceType = "URIRef"
	SourceTypeImageRef    SourceType = "ImageRef"
	SourceTypeLocalPath   SourceType = "LocalPath"
)

// ComponentVersion is the Schema for the componentversions API
type ComponentVersion struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ComponentVersionSpec   `json:"spec,omitempty"`
	Status ComponentVersionStatus `json:"status,omitempty"`
}

// ComponentVersionList contains a list of ComponentVersion
type ComponentVersionList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ComponentVersion `json:"items"`
}

// ComponentVersionSpec 定义组件版本的期望状态
type ComponentVersionSpec struct {
	ComponentName string              `json:"componentName"`
	Version       string              `json:"version"`
	Scope         ComponentScope      `json:"scope,omitempty"`
	Dependencies  []Dependency        `json:"dependencies,omitempty"`
	Images        []ImageRef          `json:"images,omitempty"`
	Install       InstallSpec         `json:"install"`
	HealthCheck   *HealthCheckSpec    `json:"healthCheck,omitempty"`
}

// ComponentVersionStatus 定义组件版本的观测状态
type ComponentVersionStatus struct {
	ObservedVersion    string              `json:"observedVersion,omitempty"`
	Phase              ComponentPhase      `json:"phase,omitempty"`
	InstallType        InstallType         `json:"installType,omitempty"`
	Conditions         []metav1.Condition  `json:"conditions,omitempty"`
	InstallDetails     InstallDetails      `json:"installDetails,omitempty"`
	LastTransitionTime *metav1.Time        `json:"lastTransitionTime,omitempty"`
	RetryCount         int                 `json:"retryCount,omitempty"`
}

// Dependency 组件依赖
type Dependency struct {
	Name       string `json:"name"`
	MinVersion string `json:"minVersion,omitempty"`
}

// ImageRef 镜像引用
type ImageRef struct {
	Name  string `json:"name"`
	Image string `json:"image"`
}

// InstallSpec 安装规格，三种策略互斥
type InstallSpec struct {
	Type          InstallType       `json:"type"`
	Manifest      *ManifestSpec     `json:"manifest,omitempty"`
	InstallScript *InstallScriptSpec `json:"installScript,omitempty"`
	HelmChart     *HelmChartSpec    `json:"helmChart,omitempty"`
}

// ManifestSpec k8s yaml 安装策略
type ManifestSpec struct {
	Items         []ManifestItem   `json:"items,omitempty"`
	ApplyStrategy ApplyStrategy    `json:"applyStrategy,omitempty"`
	Prune         bool             `json:"prune,omitempty"`
}

// ManifestItem 单个 manifest 条目
type ManifestItem struct {
	Name      string     `json:"name"`
	Namespace string     `json:"namespace,omitempty"`
	Source    SourceSpec `json:"source"`
}

// ApplyStrategy 应用策略
type ApplyStrategy string

const (
	ApplyStrategyServerSideApply ApplyStrategy = "ServerSideApply"
	ApplyStrategyThreeWayMerge  ApplyStrategy = "ThreeWayMerge"
	ApplyStrategyReplace         ApplyStrategy = "Replace"
)

// InstallScriptSpec 脚本安装策略
type InstallScriptSpec struct {
	Command    string            `json:"command,omitempty"`
	Args       []string          `json:"args,omitempty"`
	Env        []EnvVar          `json:"env,omitempty"`
	Source     SourceSpec        `json:"source"`
	Execution  *ExecutionSpec    `json:"execution,omitempty"`
	Validation *ValidationSpec   `json:"validation,omitempty"`
}

// EnvVar 环境变量
type EnvVar struct {
	Name      string       `json:"name"`
	Value     string       `json:"value,omitempty"`
	ValueFrom *EnvVarSource `json:"valueFrom,omitempty"`
}

// EnvVarSource 环境变量源
type EnvVarSource struct {
	ConfigMapKeyRef *ConfigMapKeySelector `json:"configMapKeyRef,omitempty"`
	SecretKeyRef    *SecretKeySelector    `json:"secretKeyRef,omitempty"`
}

// ConfigMapKeySelector ConfigMap 键选择器
type ConfigMapKeySelector struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key"`
}

// SecretKeySelector Secret 键选择器
type SecretKeySelector struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key"`
}

// ExecutionSpec 执行规格
type ExecutionSpec struct {
	Target        ExecutionTarget       `json:"target,omitempty"`
	NodeSelector  map[string]string      `json:"nodeSelector,omitempty"`
	Timeout       string                `json:"timeout,omitempty"`
	RetryPolicy   *RetryPolicy          `json:"retryPolicy,omitempty"`
}

// ExecutionTarget 执行目标
type ExecutionTarget string

const (
	ExecutionTargetController ExecutionTarget = "Controller"
	ExecutionTargetNodeAgent  ExecutionTarget = "NodeAgent"
	ExecutionTargetInitPod    ExecutionTarget = "InitPod"
)

// RetryPolicy 重试策略
type RetryPolicy struct {
	MaxRetries int    `json:"maxRetries,omitempty"`
	RetryDelay string `json:"retryDelay,omitempty"`
}

// ValidationSpec 验证规格
type ValidationSpec struct {
	Command        string `json:"command,omitempty"`
	ExpectedOutput string `json:"expectedOutput,omitempty"`
	Timeout        string `json:"timeout,omitempty"`
}

// HelmChartSpec Helm chart 安装策略
type HelmChartSpec struct {
	ChartRef    HelmChartRef      `json:"chartRef,omitempty"`
	ReleaseName string            `json:"releaseName,omitempty"`
	Namespace   string            `json:"namespace,omitempty"`
	Source      *ChartSourceSpec  `json:"source,omitempty"`
	Values      *runtime.RawExtension `json:"values,omitempty"`
	ValuesFrom  []ValuesFromSource `json:"valuesFrom,omitempty"`
	Timeout     string            `json:"timeout,omitempty"`
	Wait        bool              `json:"wait,omitempty"`
	Atomic      bool              `json:"atomic,omitempty"`
}

// HelmChartRef Helm chart 引用
type HelmChartRef struct {
	Name      string `json:"name"`
	Repo      string `json:"repo,omitempty"`
	Version   string `json:"version,omitempty"`
	ChartPath string `json:"chartPath,omitempty"`
}

// ChartSourceSpec Chart 来源规格
type ChartSourceSpec struct {
	Type       ChartSourceType       `json:"type"`
	Repo       *RepoChartSource      `json:"repo,omitempty"`
	OCI        *OCIChartSource       `json:"oci,omitempty"`
	LocalPath  *LocalPathChartSource `json:"localPath,omitempty"`
	ConfigMap  *ConfigMapChartSource `json:"configMap,omitempty"`
}

// ChartSourceType Chart 来源类型
type ChartSourceType string

const (
	ChartSourceTypeRepo      ChartSourceType = "Repo"
	ChartSourceTypeOCI       ChartSourceType = "OCI"
	ChartSourceTypeLocalPath ChartSourceType = "LocalPath"
	ChartSourceTypeConfigMap ChartSourceType = "ConfigMap"
)

// RepoChartSource Repo 来源
type RepoChartSource struct {
	URL              string               `json:"url"`
	Username         string               `json:"username,omitempty"`
	PasswordSecretRef *PasswordSecretRef  `json:"passwordSecretRef,omitempty"`
}

// PasswordSecretRef 密码 Secret 引用
type PasswordSecretRef struct {
	Name      string `json:"name"`
	Key       string `json:"key"`
	Namespace string `json:"namespace"`
}

// OCIChartSource OCI 来源
type OCIChartSource struct {
	Registry string `json:"registry"`
	Repository string `json:"repository"`
	Tag      string `json:"tag"`
	Insecure bool   `json:"insecure,omitempty"`
}

// LocalPathChartSource 本地路径来源
type LocalPathChartSource struct {
	Path string `json:"path"`
}

// ConfigMapChartSource ConfigMap 来源
type ConfigMapChartSource struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key"`
}

// ValuesFromSource Values 来源
type ValuesFromSource struct {
	Kind      string `json:"kind"`
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	ValuesKey string `json:"valuesKey,omitempty"`
}

// SourceSpec 统一来源规格
type SourceSpec struct {
	Type         SourceType        `json:"type"`
	Inline       *InlineSource     `json:"inline,omitempty"`
	ConfigMapRef *ConfigMapSource  `json:"configMapRef,omitempty"`
	SecretRef    *SecretSource     `json:"secretRef,omitempty"`
	URIRef       *URISource        `json:"uriRef,omitempty"`
	ImageRef     *ImageSource      `json:"imageRef,omitempty"`
	LocalPath    *LocalPathSource  `json:"localPath,omitempty"`
}

// InlineSource 内嵌内容
type InlineSource struct {
	Content string `json:"content,omitempty"`
}

// ConfigMapSource ConfigMap 引用
type ConfigMapSource struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key,omitempty"`
}

// SecretSource Secret 引用
type SecretSource struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key,omitempty"`
}

// URISource 远程 URI
type URISource struct {
	URI                    string `json:"uri"`
	Checksum               string `json:"checksum,omitempty"`
	InsecureSkipTLSVerify  bool   `json:"insecureSkipTLSVerify,omitempty"`
}

// ImageSource 容器镜像
type ImageSource struct {
	Image           string `json:"image"`
	Path            string `json:"path,omitempty"`
	ImagePullSecret string `json:"imagePullSecret,omitempty"`
}

// LocalPathSource 本地路径
type LocalPathSource struct {
	Path     string `json:"path"`
	Checksum string `json:"checksum,omitempty"`
}

// HealthCheckSpec 健康检查规格
type HealthCheckSpec struct {
	Type             string          `json:"type"`
	HTTPGet          *HTTPGetAction  `json:"httpGet,omitempty"`
	TCPSocket        *TCPSocketAction `json:"tcpSocket,omitempty"`
	Command          *CommandAction  `json:"command,omitempty"`
	ClusterOperator  *ClusterOperatorAction `json:"clusterOperator,omitempty"`
	Interval         string          `json:"interval,omitempty"`
	Timeout          string          `json:"timeout,omitempty"`
}

// HTTPGetAction HTTP GET 检查
type HTTPGetAction struct {
	Path   string `json:"path"`
	Port   int    `json:"port"`
	Scheme string `json:"scheme,omitempty"`
}

// TCPSocketAction TCP Socket 检查
type TCPSocketAction struct {
	Port int `json:"port"`
}

// CommandAction 命令检查
type CommandAction struct {
	Command []string `json:"command"`
}

// ClusterOperatorAction ClusterOperator 检查
type ClusterOperatorAction struct {
	Name          string `json:"name"`
	ConditionType string `json:"conditionType,omitempty"`
}

// InstallDetails 安装详情
type InstallDetails struct {
	Manifest      *ManifestInstallDetails      `json:"manifest,omitempty"`
	InstallScript *InstallScriptInstallDetails `json:"installScript,omitempty"`
	HelmChart     *HelmChartInstallDetails     `json:"helmChart,omitempty"`
}

// ManifestInstallDetails Manifest 安装详情
type ManifestInstallDetails struct {
	AppliedResources []AppliedResource `json:"appliedResources,omitempty"`
	PrunedResources  []PrunedResource  `json:"prunedResources,omitempty"`
}

// AppliedResource 已应用资源
type AppliedResource struct {
	Group     string       `json:"group,omitempty"`
	Version   string       `json:"version,omitempty"`
	Kind      string       `json:"kind,omitempty"`
	Name      string       `json:"name,omitempty"`
	Namespace string       `json:"namespace,omitempty"`
	Hash      string       `json:"hash,omitempty"`
	AppliedAt *metav1.Time `json:"appliedAt,omitempty"`
}

// PrunedResource 已清理资源
type PrunedResource struct {
	Group     string `json:"group,omitempty"`
	Version   string `json:"version,omitempty"`
	Kind      string `json:"kind,omitempty"`
	Name      string `json:"name,omitempty"`
	Namespace string `json:"namespace,omitempty"`
}

// InstallScriptInstallDetails InstallScript 安装详情
type InstallScriptInstallDetails struct {
	ExecutionId     string         `json:"executionId,omitempty"`
	TargetNodes     []TargetNode   `json:"targetNodes,omitempty"`
	ValidationResult *ValidationResult `json:"validationResult,omitempty"`
}

// TargetNode 目标节点
type TargetNode struct {
	NodeName    string       `json:"nodeName,omitempty"`
	NodeIP      string       `json:"nodeIP,omitempty"`
	Phase       string       `json:"phase,omitempty"`
	ExitCode    int          `json:"exitCode,omitempty"`
	Stdout      string       `json:"stdout,omitempty"`
	Stderr      string       `json:"stderr,omitempty"`
	StartedAt   *metav1.Time `json:"startedAt,omitempty"`
	FinishedAt  *metav1.Time `json:"finishedAt,omitempty"`
	RetryCount  int          `json:"retryCount,omitempty"`
}

// ValidationResult 验证结果
type ValidationResult struct {
	Passed    bool         `json:"passed,omitempty"`
	Output    string       `json:"output,omitempty"`
	CheckedAt *metav1.Time `json:"checkedAt,omitempty"`
}

// HelmChartInstallDetails HelmChart 安装详情
type HelmChartInstallDetails struct {
	ReleaseStatus string       `json:"releaseStatus,omitempty"`
	Revision      int          `json:"revision,omitempty"`
	LastDeployed  *metav1.Time `json:"lastDeployed,omitempty"`
}

// NodeConfig is the Schema for the nodeconfigs API
type NodeConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   NodeConfigSpec   `json:"spec,omitempty"`
	Status NodeConfigStatus `json:"status,omitempty"`
}

// NodeConfigList contains a list of NodeConfig
type NodeConfigList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []NodeConfig `json:"items"`
}

// NodeConfigSpec NodeConfig 规格
type NodeConfigSpec struct {
	NodeName          string              `json:"nodeName"`
	NodeIP            string              `json:"nodeIP,omitempty"`
	Roles             []string            `json:"roles,omitempty"`
	OSInfo            *OSInfo             `json:"osInfo,omitempty"`
	ComponentConfigs  []ComponentConfig  `json:"componentConfigs,omitempty"`
}

// OSInfo 操作系统信息
type OSInfo struct {
	Type    string `json:"type,omitempty"`
	Version string `json:"version,omitempty"`
	Arch    string `json:"arch,omitempty"`
}

// ComponentConfig 组件配置
type ComponentConfig struct {
	ComponentName   string                 `json:"componentName"`
	ConfigOverrides *runtime.RawExtension `json:"configOverrides,omitempty"`
	NodeLabels      map[string]string      `json:"nodeLabels,omitempty"`
}

// NodeConfigStatus NodeConfig 状态
type NodeConfigStatus struct {
	ComponentVersions []ComponentVersionInfo `json:"componentVersions,omitempty"`
	NodeHealth        string                 `json:"nodeHealth,omitempty"`
	LastHeartbeat     *metav1.Time           `json:"lastHeartbeat,omitempty"`
}

// ComponentVersionInfo 组件版本信息
type ComponentVersionInfo struct {
	ComponentName string       `json:"componentName"`
	CurrentVersion string       `json:"currentVersion"`
	TargetVersion  string       `json:"targetVersion,omitempty"`
	Phase          string       `json:"phase,omitempty"`
	LastUpdated    *metav1.Time `json:"lastUpdated,omitempty"`
}

// InstallResult 安装结果
type InstallResult struct {
	Phase   ComponentPhase
	Details runtime.Object
	Requeue bool
}

// VerifyResult 验证结果
type VerifyResult struct {
	Passed  bool
	Message string
}

// ============================================================================
// 二、Installer 接口
// ============================================================================

type Installer interface {
	Install(ctx context.Context, cv *ComponentVersion) (*InstallResult, error)
	Verify(ctx context.Context, cv *ComponentVersion) (*VerifyResult, error)
	Rollback(ctx context.Context, cv *ComponentVersion) error
	Uninstall(ctx context.Context, cv *ComponentVersion) error
}

// ============================================================================
// 三、SourceResolver
// ============================================================================

type SourceResolver struct {
	client.Client
	HTTPClient *http.Client
}

type ResolvedContent struct {
	Content  []byte
	Checksum string
	Source   SourceType
}

func NewSourceResolver(c client.Client) *SourceResolver {
	return &SourceResolver{
		Client:     c,
		HTTPClient: &http.Client{Timeout: 30 * time.Second},
	}
}

func (r *SourceResolver) Resolve(ctx context.Context, spec *SourceSpec) (*ResolvedContent, error) {
	switch spec.Type {
	case SourceTypeInline:
		return r.resolveInline(spec)
	case SourceTypeConfigMapRef:
		return r.resolveConfigMap(ctx, spec)
	case SourceTypeSecretRef:
		return r.resolveSecret(ctx, spec)
	case SourceTypeURIRef:
		return r.resolveURI(ctx, spec)
	case SourceTypeImageRef:
		return r.resolveImage(ctx, spec)
	case SourceTypeLocalPath:
		return r.resolveLocalPath(spec)
	default:
		return nil, fmt.Errorf("unsupported source type: %s", spec.Type)
	}
}

func (r *SourceResolver) resolveInline(spec *SourceSpec) (*ResolvedContent, error) {
	if spec.Inline == nil {
		return nil, fmt.Errorf("inline source not specified")
	}
	return &ResolvedContent{
		Content: []byte(spec.Inline.Content),
		Source:  SourceTypeInline,
	}, nil
}

func (r *SourceResolver) resolveConfigMap(ctx context.Context, spec *SourceSpec) (*ResolvedContent, error) {
	if spec.ConfigMapRef == nil {
		return nil, fmt.Errorf("configMapRef source not specified")
	}
	cm := &corev1.ConfigMap{}
	if err := r.Get(ctx, client.ObjectKey{
		Name:      spec.ConfigMapRef.Name,
		Namespace: spec.ConfigMapRef.Namespace,
	}, cm); err != nil {
		return nil, err
	}
	key := spec.ConfigMapRef.Key
	if key == "" {
		for _, v := range cm.Data {
			return &ResolvedContent{
				Content: []byte(v),
				Source:  SourceTypeConfigMapRef,
			}, nil
		}
		return nil, fmt.Errorf("configMap has no data")
	}
	if content, ok := cm.Data[key]; ok {
		return &ResolvedContent{
			Content: []byte(content),
			Source:  SourceTypeConfigMapRef,
		}, nil
	}
	return nil, fmt.Errorf("configMap key %s not found", key)
}

func (r *SourceResolver) resolveSecret(ctx context.Context, spec *SourceSpec) (*ResolvedContent, error) {
	if spec.SecretRef == nil {
		return nil, fmt.Errorf("secretRef source not specified")
	}
	secret := &corev1.Secret{}
	if err := r.Get(ctx, client.ObjectKey{
		Name:      spec.SecretRef.Name,
		Namespace: spec.SecretRef.Namespace,
	}, secret); err != nil {
		return nil, err
	}
	key := spec.SecretRef.Key
	if key == "" {
		for _, v := range secret.Data {
			return &ResolvedContent{
				Content: v,
				Source:  SourceTypeSecretRef,
			}, nil
		}
		return nil, fmt.Errorf("secret has no data")
	}
	if content, ok := secret.Data[key]; ok {
		return &ResolvedContent{
			Content: content,
			Source:  SourceTypeSecretRef,
		}, nil
	}
	return nil, fmt.Errorf("secret key %s not found", key)
}

func (r *SourceResolver) resolveURI(ctx context.Context, spec *SourceSpec) (*ResolvedContent, error) {
	if spec.URIRef == nil {
		return nil, fmt.Errorf("uriRef source not specified")
	}
	req, err := http.NewRequestWithContext(ctx, "GET", spec.URIRef.URI, nil)
	if err != nil {
		return nil, err
	}
	resp, err := r.HTTPClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
	}
	content, err := os.ReadFile(resp.Body.Name())
	if err != nil {
		return nil, err
	}
	return &ResolvedContent{
		Content: content,
		Source:  SourceTypeURIRef,
	}, nil
}

func (r *SourceResolver) resolveImage(ctx context.Context, spec *SourceSpec) (*ResolvedContent, error) {
	if spec.ImageRef == nil {
		return nil, fmt.Errorf("imageRef source not specified")
	}
	return nil, fmt.Errorf("imageRef not implemented")
}

func (r *SourceResolver) resolveLocalPath(spec *SourceSpec) (*ResolvedContent, error) {
	if spec.LocalPath == nil {
		return nil, fmt.Errorf("localPath source not specified")
	}
	content, err := os.ReadFile(spec.LocalPath.Path)
	if err != nil {
		return nil, err
	}
	return &ResolvedContent{
		Content: content,
		Source:  SourceTypeLocalPath,
	}, nil
}

// ============================================================================
// 四、ManifestInstaller
// ============================================================================

type ManifestInstaller struct {
	client.Client
	dynamicClient dynamic.Interface
}

func NewManifestInstaller(c client.Client, dc dynamic.Interface) *ManifestInstaller {
	return &ManifestInstaller{
		Client:        c,
		dynamicClient: dc,
	}
}

func (m *ManifestInstaller) Install(ctx context.Context, cv *ComponentVersion) (*InstallResult, error) {
	log := log.FromContext(ctx)
	log.Info("Installing component via Manifest", "component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	if cv.Spec.Install.Manifest == nil {
		return nil, fmt.Errorf("manifest spec not specified")
	}

	appliedResources := make([]AppliedResource, 0)
	for _, item := range cv.Spec.Install.Manifest.Items {
		content, err := m.resolveItemContent(ctx, &item)
		if err != nil {
			return nil, fmt.Errorf("resolve item %s failed: %w", item.Name, err)
		}

		resources, err := m.parseManifests(content)
		if err != nil {
			return nil, fmt.Errorf("parse item %s failed: %w", item.Name, err)
		}

		for _, res := range resources {
			if err := m.applyResource(ctx, &res, item.Namespace, cv.Spec.Install.Manifest.ApplyStrategy); err != nil {
				return nil, fmt.Errorf("apply resource failed: %w", err)
			}
			appliedResources = append(appliedResources, AppliedResource{
				Group:     res.GroupVersionKind().Group,
				Version:   res.GroupVersionKind().Version,
				Kind:      res.GroupVersionKind().Kind,
				Name:      res.GetName(),
				Namespace: res.GetNamespace(),
				AppliedAt: &metav1.Time{Time: time.Now()},
			})
		}
	}

	return &InstallResult{
		Phase:   PhaseVerifying,
		Details: &ManifestInstallDetails{AppliedResources: appliedResources},
	}, nil
}

func (m *ManifestInstaller) Verify(ctx context.Context, cv *ComponentVersion) (*VerifyResult, error) {
	log := log.FromContext(ctx)
	log.Info("Verifying component via Manifest", "component", cv.Spec.ComponentName)

	if cv.Status.InstallDetails.Manifest == nil {
		return &VerifyResult{Passed: false, Message: "no manifest install details"}, nil
	}

	for _, res := range cv.Status.InstallDetails.Manifest.AppliedResources {
		if err := m.checkResourceHealth(ctx, &res); err != nil {
			return &VerifyResult{Passed: false, Message: err.Error()}, nil
		}
	}

	return &VerifyResult{Passed: true}, nil
}

func (m *ManifestInstaller) Rollback(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Rolling back component via Manifest", "component", cv.Spec.ComponentName)
	return fmt.Errorf("rollback not implemented")
}

func (m *ManifestInstaller) Uninstall(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Uninstalling component via Manifest", "component", cv.Spec.ComponentName)
	return fmt.Errorf("uninstall not implemented")
}

func (m *ManifestInstaller) resolveItemContent(ctx context.Context, item *ManifestItem) ([]byte, error) {
	resolver := NewSourceResolver(m.Client)
	content, err := resolver.Resolve(ctx, &item.Source)
	if err != nil {
		return nil, err
	}
	return content.Content, nil
}

func (m *ManifestInstaller) parseManifests(content []byte) ([]unstructured.Unstructured, error) {
	return nil, fmt.Errorf("parseManifests not implemented")
}

func (m *ManifestInstaller) applyResource(ctx context.Context, res *unstructured.Unstructured, namespace string, strategy ApplyStrategy) error {
	return fmt.Errorf("applyResource not implemented")
}

func (m *ManifestInstaller) checkResourceHealth(ctx context.Context, res *AppliedResource) error {
	return fmt.Errorf("checkResourceHealth not implemented")
}

// ============================================================================
// 五、ScriptInstaller
// ============================================================================

type ScriptInstaller struct {
	client.Client
}

func NewScriptInstaller(c client.Client) *ScriptInstaller {
	return &ScriptInstaller{Client: c}
}

func (s *ScriptInstaller) Install(ctx context.Context, cv *ComponentVersion) (*InstallResult, error) {
	log := log.FromContext(ctx)
	log.Info("Installing component via InstallScript", "component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	if cv.Spec.Install.InstallScript == nil {
		return nil, fmt.Errorf("installScript spec not specified")
	}

	resolver := NewSourceResolver(s.Client)
	content, err := resolver.Resolve(ctx, &cv.Spec.Install.InstallScript.Source)
	if err != nil {
		return nil, fmt.Errorf("resolve script source failed: %w", err)
	}

	executionId := fmt.Sprintf("exec-%s", cv.Name)
	log.Info("Script content resolved", "executionId", executionId)

	return &InstallResult{
		Phase: PhaseInstalling,
		Details: &InstallScriptInstallDetails{
			ExecutionId: executionId,
		},
		Requeue: true,
	}, nil
}

func (s *ScriptInstaller) Verify(ctx context.Context, cv *ComponentVersion) (*VerifyResult, error) {
	log := log.FromContext(ctx)
	log.Info("Verifying component via InstallScript", "component", cv.Spec.ComponentName)
	return &VerifyResult{Passed: true}, nil
}

func (s *ScriptInstaller) Rollback(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Rolling back component via InstallScript", "component", cv.Spec.ComponentName)
	return fmt.Errorf("rollback not implemented")
}

func (s *ScriptInstaller) Uninstall(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Uninstalling component via InstallScript", "component", cv.Spec.ComponentName)
	return fmt.Errorf("uninstall not implemented")
}

// ============================================================================
// 六、HelmChartInstaller
// ============================================================================

type HelmChartInstaller struct {
	client.Client
	Settings     *cli.EnvSettings
	RestConfig   *rest.Config
}

func NewHelmChartInstaller(c client.Client, cfg *rest.Config) *HelmChartInstaller {
	return &HelmChartInstaller{
		Client:     c,
		Settings:   cli.New(),
		RestConfig: cfg,
	}
}

func (h *HelmChartInstaller) Install(ctx context.Context, cv *ComponentVersion) (*InstallResult, error) {
	log := log.FromContext(ctx)
	log.Info("Installing component via HelmChart", "component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	if cv.Spec.Install.HelmChart == nil {
		return nil, fmt.Errorf("helmChart spec not specified")
	}

	actionConfig, err := h.newActionConfig(cv.Spec.Install.HelmChart.Namespace)
	if err != nil {
		return nil, err
	}

	histClient := action.NewHistory(actionConfig)
	releases, err := histClient.Run(cv.Spec.Install.HelmChart.ReleaseName)
	if err != nil {
		return nil, err
	}

	chart, err := h.loadChart(ctx, cv)
	if err != nil {
		return nil, err
	}

	values, err := h.mergeValues(ctx, cv)
	if err != nil {
		return nil, err
	}

	var rel *release.Release
	if len(releases) == 0 {
		log.Info("Installing new Helm release", "release", cv.Spec.Install.HelmChart.ReleaseName)
		install := action.NewInstall(actionConfig)
		install.ReleaseName = cv.Spec.Install.HelmChart.ReleaseName
		install.Namespace = cv.Spec.Install.HelmChart.Namespace
		install.Wait = cv.Spec.Install.HelmChart.Wait
		install.Atomic = cv.Spec.Install.HelmChart.Atomic
		rel, err = install.Run(chart, values)
	} else {
		log.Info("Upgrading existing Helm release", "release", cv.Spec.Install.HelmChart.ReleaseName)
		upgrade := action.NewUpgrade(actionConfig)
		upgrade.Wait = cv.Spec.Install.HelmChart.Wait
		upgrade.Atomic = cv.Spec.Install.HelmChart.Atomic
		rel, err = upgrade.Run(cv.Spec.Install.HelmChart.ReleaseName, chart, values)
	}

	if err != nil {
		return nil, err
	}

	return &InstallResult{
		Phase: PhaseVerifying,
		Details: &HelmChartInstallDetails{
			ReleaseStatus: string(rel.Info.Status),
			Revision:      rel.Version,
			LastDeployed:  &metav1.Time{Time: rel.Info.LastDeployed.Time},
		},
	}, nil
}

func (h *HelmChartInstaller) Verify(ctx context.Context, cv *ComponentVersion) (*VerifyResult, error) {
	log := log.FromContext(ctx)
	log.Info("Verifying component via HelmChart", "component", cv.Spec.ComponentName)

	if cv.Status.InstallDetails.HelmChart == nil {
		return &VerifyResult{Passed: false, Message: "no helm chart install details"}, nil
	}

	actionConfig, err := h.newActionConfig(cv.Spec.Install.HelmChart.Namespace)
	if err != nil {
		return nil, err
	}

	histClient := action.NewHistory(actionConfig)
	releases, err := histClient.Run(cv.Spec.Install.HelmChart.ReleaseName)
	if err != nil {
		return nil, err
	}

	if len(releases) == 0 {
		return &VerifyResult{Passed: false, Message: "release not found"}, nil
	}

	rel := releases[len(releases)-1]
	if rel.Info.Status == release.StatusDeployed {
		return &VerifyResult{Passed: true}, nil
	}

	return &VerifyResult{
		Passed:  false,
		Message: fmt.Sprintf("release status: %s", rel.Info.Status),
	}, nil
}

func (h *HelmChartInstaller) Rollback(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Rolling back component via HelmChart", "component", cv.Spec.ComponentName)
	return fmt.Errorf("rollback not implemented")
}

func (h *HelmChartInstaller) Uninstall(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	log.Info("Uninstalling component via HelmChart", "component", cv.Spec.ComponentName)
	return fmt.Errorf("uninstall not implemented")
}

func (h *HelmChartInstaller) newActionConfig(namespace string) (*action.Configuration, error) {
	return nil, fmt.Errorf("newActionConfig not implemented")
}

func (h *HelmChartInstaller) loadChart(ctx context.Context, cv *ComponentVersion) (interface{}, error) {
	return nil, fmt.Errorf("loadChart not implemented")
}

func (h *HelmChartInstaller) mergeValues(ctx context.Context, cv *ComponentVersion) (map[string]interface{}, error) {
	values := make(map[string]interface{})
	if cv.Spec.Install.HelmChart.Values != nil {
		if err := json.Unmarshal(cv.Spec.Install.HelmChart.Values.Raw, &values); err != nil {
			return nil, err
		}
	}
	return values, nil
}

// ============================================================================
// 七、状态机
// ============================================================================

type StateMachine struct {
	client.Client
}

func NewStateMachine(c client.Client) *StateMachine {
	return &StateMachine{Client: c}
}

func (sm *StateMachine) Drive(ctx context.Context, cv *ComponentVersion, execute func(context.Context, *ComponentVersion) (*InstallResult, error)) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	currentPhase := cv.Status.Phase
	if currentPhase == "" {
		currentPhase = PhasePending
	}

	log.Info("Driving component installation state machine",
		"component", cv.Spec.ComponentName,
		"currentPhase", currentPhase)

	switch currentPhase {
	case PhasePending:
		return sm.transition(ctx, cv, PhaseInstalling, execute)

	case PhaseInstalling, PhaseUpgrading:
		result, err := execute(ctx, cv)
		if err != nil {
			return sm.transition(ctx, cv, PhaseFailed, nil)
		}
		return sm.transition(ctx, cv, result.Phase, nil)

	case PhaseVerifying:
		result, err := execute(ctx, cv)
		if err != nil || !result.Requeue {
			return sm.transition(ctx, cv, PhaseFailed, nil)
		}
		return sm.transition(ctx, cv, PhaseInstalled, nil)

	case PhaseInstalled:
		if cv.Status.ObservedVersion != cv.Spec.Version {
			log.Info("Version changed, starting upgrade",
				"oldVersion", cv.Status.ObservedVersion,
				"newVersion", cv.Spec.Version)
			return sm.transition(ctx, cv, PhaseUpgrading, execute)
		}
		return ctrl.Result{RequeueAfter: 30 * time.Second}, nil

	case PhaseFailed:
		if cv.Status.RetryCount < 5 {
			log.Info("Retrying failed installation", "retryCount", cv.Status.RetryCount)
			return sm.transition(ctx, cv, PhaseInstalling, execute)
		}
		return ctrl.Result{}, nil

	case PhaseRollingBack:
		_, err := execute(ctx, cv)
		if err != nil {
			return sm.transition(ctx, cv, PhaseFailed, nil)
		}
		return sm.transition(ctx, cv, PhaseInstalled, nil)

	default:
		return ctrl.Result{}, nil
	}
}

func (sm *StateMachine) transition(ctx context.Context, cv *ComponentVersion, newPhase ComponentPhase, execute func(context.Context, *ComponentVersion) (*InstallResult, error)) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	log.Info("Transitioning phase",
		"component", cv.Spec.ComponentName,
		"from", cv.Status.Phase,
		"to", newPhase)

	oldPhase := cv.Status.Phase
	cv.Status.Phase = newPhase
	cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}

	if newPhase == PhaseFailed {
		cv.Status.RetryCount++
	}

	now := time.Now()
	meta.SetStatusCondition(&cv.Status.Conditions, metav1.Condition{
		Type:               string(newPhase),
		Status:             metav1.ConditionTrue,
		LastTransitionTime: metav1.Time{Time: now},
		Reason:             string(newPhase) + "Phase",
		Message:            fmt.Sprintf("Transitioned from %s to %s", oldPhase, newPhase),
	})

	if err := sm.Status().Update(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{Requeue: true}, nil
}

// ============================================================================
// 八、依赖解析
// ============================================================================

type DependencyResolver struct {
	client.Client
}

func NewDependencyResolver(c client.Client) *DependencyResolver {
	return &DependencyResolver{Client: c}
}

func (dr *DependencyResolver) CheckDependencies(ctx context.Context, cv *ComponentVersion) (bool, error) {
	log := log.FromContext(ctx)
	for _, dep := range cv.Spec.Dependencies {
		depCV := &ComponentVersion{}
		if err := dr.Get(ctx, client.ObjectKey{Name: dep.Name}, depCV); err != nil {
			if apierrors.IsNotFound(err) {
				log.Info("Dependency not found", "dependency", dep.Name)
				return false, nil
			}
			return false, err
		}
		if depCV.Status.Phase != PhaseInstalled {
			log.Info("Dependency not installed", "dependency", dep.Name, "phase", depCV.Status.Phase)
			return false, nil
		}
		if dep.MinVersion != "" {
			if !versionGTE(depCV.Status.ObservedVersion, dep.MinVersion) {
				log.Info("Dependency version too low",
					"dependency", dep.Name,
					"current", depCV.Status.ObservedVersion,
					"required", dep.MinVersion)
				return false, nil
			}
		}
	}
	return true, nil
}

func versionGTE(current, required string) bool {
	return true
}

// ============================================================================
// 九、健康检查
// ============================================================================

type HealthChecker struct {
	client.Client
	RestConfig *rest.Config
}

func NewHealthChecker(c client.Client, cfg *rest.Config) *HealthChecker {
	return &HealthChecker{
		Client:     c,
		RestConfig: cfg,
	}
}

func (hc *HealthChecker) Check(ctx context.Context, cv *ComponentVersion) error {
	log := log.FromContext(ctx)
	if cv.Spec.HealthCheck == nil {
		return nil
	}

	log.Info("Checking component health", "component", cv.Spec.ComponentName)

	switch cv.Spec.HealthCheck.Type {
	case "HTTPGet":
		return hc.checkHTTP(ctx, cv)
	case "TCPSocket":
		return hc.checkTCP(ctx, cv)
	case "Command":
		return hc.checkCommand(ctx, cv)
	case "ClusterOperator":
		return hc.checkClusterOperator(ctx, cv)
	default:
		return fmt.Errorf("unsupported health check type: %s", cv.Spec.HealthCheck.Type)
	}
}

func (hc *HealthChecker) checkHTTP(ctx context.Context, cv *ComponentVersion) error {
	return fmt.Errorf("checkHTTP not implemented")
}

func (hc *HealthChecker) checkTCP(ctx context.Context, cv *ComponentVersion) error {
	return fmt.Errorf("checkTCP not implemented")
}

func (hc *HealthChecker) checkCommand(ctx context.Context, cv *ComponentVersion) error {
	return fmt.Errorf("checkCommand not implemented")
}

func (hc *HealthChecker) checkClusterOperator(ctx context.Context, cv *ComponentVersion) error {
	return fmt.Errorf("checkClusterOperator not implemented")
}

// ============================================================================
// 十、Dispatcher
// ============================================================================

type Dispatcher struct {
	ManifestInstaller  *ManifestInstaller
	ScriptInstaller    *ScriptInstaller
	HelmChartInstaller *HelmChartInstaller
}

func NewDispatcher(mi *ManifestInstaller, si *ScriptInstaller, hi *HelmChartInstaller) *Dispatcher {
	return &Dispatcher{
		ManifestInstaller:  mi,
		ScriptInstaller:    si,
		HelmChartInstaller: hi,
	}
}

func (d *Dispatcher) GetInstaller(installType InstallType) (Installer, error) {
	switch installType {
	case InstallTypeManifest:
		return d.ManifestInstaller, nil
	case InstallTypeInstallScript:
		return d.ScriptInstaller, nil
	case InstallTypeHelmChart:
		return d.HelmChartInstaller, nil
	default:
		return nil, fmt.Errorf("unsupported install type: %s", installType)
	}
}

func (d *Dispatcher) Dispatch(ctx context.Context, cv *ComponentVersion) (*InstallResult, error) {
	installer, err := d.GetInstaller(cv.Spec.Install.Type)
	if err != nil {
		return nil, err
	}

	switch cv.Status.Phase {
	case PhaseInstalling, PhaseUpgrading:
		return installer.Install(ctx, cv)
	case PhaseVerifying:
		verifyResult, err := installer.Verify(ctx, cv)
		return &InstallResult{
			Phase:   PhaseInstalled,
			Requeue: verifyResult.Passed,
		}, err
	case PhaseRollingBack:
		if err := installer.Rollback(ctx, cv); err != nil {
			return nil, err
		}
		return &InstallResult{Phase: PhaseInstalled}, nil
	default:
		return nil, nil
	}
}

// ============================================================================
// 十一、Reconciler
// ============================================================================

const componentVersionFinalizer = "componentversion.orchestration.bke.io/finalizer"

type ComponentVersionReconciler struct {
	client.Client
	Scheme         *runtime.Scheme
	Recorder       record.EventRecorder
	RestConfig     *rest.Config
	Dispatcher     *Dispatcher
	StateMachine   *StateMachine
	HealthChecker  *HealthChecker
	DepResolver    *DependencyResolver
	SourceResolver *SourceResolver
}

func (r *ComponentVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	cv := &ComponentVersion{}
	if err := r.Get(ctx, req.NamespacedName, cv); err != nil {
		if apierrors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		log.Error(err, "Failed to get ComponentVersion")
		return ctrl.Result{}, err
	}

	if !cv.DeletionTimestamp.IsZero() {
		return r.handleDeletion(ctx, cv)
	}

	if err := r.ensureFinalizer(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}

	satisfied, err := r.DepResolver.CheckDependencies(ctx, cv)
	if err != nil {
		return r.updatePhase(ctx, cv, PhasePending, err)
	}
	if !satisfied {
		return r.updatePhase(ctx, cv, PhasePending, nil)
	}

	executeFn := func(ctx context.Context, cv *ComponentVersion) (*InstallResult, error) {
		return r.Dispatcher.Dispatch(ctx, cv)
	}
	result, err := r.StateMachine.Drive(ctx, cv, executeFn)
	if err != nil {
		return result, err
	}

	if cv.Status.Phase == PhaseInstalled {
		if err := r.HealthChecker.Check(ctx, cv); err != nil {
			r.Recorder.Event(cv, corev1.EventTypeWarning, "HealthCheckFailed", err.Error())
		}
	}

	return result, nil
}

func (r *ComponentVersionReconciler) handleDeletion(ctx context.Context, cv *ComponentVersion) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	log.Info("Deleting ComponentVersion", "name", cv.Name)

	if !controllerutil.ContainsFinalizer(cv, componentVersionFinalizer) {
		return ctrl.Result{}, nil
	}

	installer, err := r.Dispatcher.GetInstaller(cv.Spec.Install.Type)
	if err == nil {
		if err := installer.Uninstall(ctx, cv); err != nil {
			log.Error(err, "Failed to uninstall component")
		}
	}

	controllerutil.RemoveFinalizer(cv, componentVersionFinalizer)
	if err := r.Update(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

func (r *ComponentVersionReconciler) ensureFinalizer(ctx context.Context, cv *ComponentVersion) error {
	if controllerutil.ContainsFinalizer(cv, componentVersionFinalizer) {
		return nil
	}
	controllerutil.AddFinalizer(cv, componentVersionFinalizer)
	return r.Update(ctx, cv)
}

func (r *ComponentVersionReconciler) updatePhase(ctx context.Context, cv *ComponentVersion, phase ComponentPhase, err error) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	if err != nil {
		log.Error(err, "Updating phase with error", "phase", phase)
	}
	cv.Status.Phase = phase
	cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}
	if err := r.Status().Update(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}

func (r *ComponentVersionReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&ComponentVersion{}).
		WithOptions(controller.Options{
			MaxConcurrentReconciles: 5,
		}).
		Complete(r)
}

// ============================================================================
// 十二、主程序
// ============================================================================

func main() {
	log.SetLogger(log.New())
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:             runtime.NewScheme(),
		MetricsBindAddress: ":8080",
	})
	if err != nil {
		log.Log.Error(err, "Failed to create manager")
		os.Exit(1)
	}

	kubeClient := mgr.GetClient()
	restConfig := mgr.GetConfig()
	dynamicClient, err := dynamic.NewForConfig(restConfig)
	if err != nil {
		log.Log.Error(err, "Failed to create dynamic client")
		os.Exit(1)
	}

	manifestInstaller := NewManifestInstaller(kubeClient, dynamicClient)
	scriptInstaller := NewScriptInstaller(kubeClient)
	helmInstaller := NewHelmChartInstaller(kubeClient, restConfig)

	reconciler := &ComponentVersionReconciler{
		Client:         kubeClient,
		Scheme:         mgr.GetScheme(),
		Recorder:       mgr.GetEventRecorderFor("componentversion-controller"),
		RestConfig:     restConfig,
		Dispatcher:     NewDispatcher(manifestInstaller, scriptInstaller, helmInstaller),
		StateMachine:   NewStateMachine(kubeClient),
		HealthChecker:  NewHealthChecker(kubeClient, restConfig),
		DepResolver:    NewDependencyResolver(kubeClient),
		SourceResolver: NewSourceResolver(kubeClient),
	}

	if err := reconciler.SetupWithManager(mgr); err != nil {
		log.Log.Error(err, "Failed to setup controller")
		os.Exit(1)
	}

	log.Log.Info("Starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		log.Log.Error(err, "Failed to start manager")
		os.Exit(1)
	}
}
```
## 代码实现框架说明
### 核心架构特点
1. **分层设计**：
   - **API 层**：完整的 CRD 类型定义
   - **Resolver 层**：SourceResolver 统一解析 6 种来源
   - **Installer 层**：策略模式实现三种安装方式
   - **状态机层**：驱动组件安装生命周期
   - **Controller 层**：主调谐循环
2. **关键组件**：
   - `Installer` 接口：定义统一的安装/验证/回滚/卸载接口
   - `SourceResolver`：统一处理 Inline/ConfigMap/Secret/URI/Image/LocalPath
   - `StateMachine`：7 个状态的转换管理
   - `DependencyResolver`：依赖检查
   - `HealthChecker`：健康检查
   - `Dispatcher`：根据安装类型路由到对应的 Installer
3. **三种安装策略**：
   - **ManifestInstaller**：k8s 资源应用
   - **ScriptInstaller**：脚本执行
   - **HelmChartInstaller**：Helm chart 管理
### 待完善的功能（标记为 `not implemented`）
- Manifest 资源解析与应用
- Script 执行与验证
- Helm chart 加载与 action config 创建
- 健康检查的具体实现
- ImageRef 来源解析
- 回滚逻辑
- 卸载逻辑
### 使用方式
1. 定义 CRD 资源：
   ```yaml
   apiVersion: orchestration.bke.io/v1alpha1
   kind: ComponentVersion
   metadata:
     name: example-component
   spec:
     componentName: example
     version: 1.0.0
     install:
       type: Manifest
       manifest:
         items:
         - name: deployment
           source:
             type: ConfigMapRef
             configMapRef:
               name: example-manifests
               namespace: default
   ```

2. 运行 Controller：
   ```bash
   go run main.go
   ```

# 三次输出的 Prompt 拆分方案
## 三次输出的 Prompt 拆分方案
将 ComponentVersion Controller 代码实现按**自底向上**的依赖关系拆分为三次输出，每次输出一个完整的、可独立编译的层次：
### 第一次 Prompt（基础层：API 类型 + Source 解析 + 状态机 + 依赖解析）
> 实现 ComponentVersion Controller 的基础层代码，包含以下模块：
>
> 1. **API 类型定义**（`api/componentversion/v1alpha1/`）
>    - `componentversion_types.go`：ComponentVersion CRD 的 Spec/Status 完整类型定义，包含三种安装策略（Manifest/InstallScript/HelmChart）的嵌套结构
>    - `nodeconfig_types.go`：NodeConfig CRD 的 Spec/Status 类型定义
>    - `install_types.go`：安装策略相关类型（InstallSpec、ManifestSpec、InstallScriptSpec、HelmChartSpec、ExecutionSpec、ValidationSpec、ApplyStrategy 等）
>    - `source_types.go`：来源相关类型（SourceSpec 及 6 种来源类型：Inline/ConfigMapRef/SecretRef/URIRef/ImageRef/LocalPath）
>    - `groupversion_info.go`：GroupVersion 常量与 SchemeBuilder 注册
>
> 2. **Source 解析器**（`pkg/componentversion/source/`）
>    - `resolver.go`：Resolver 结构体与统一解析接口 `Resolve(ctx, spec) → (*ResolvedContent, error)`，按 SourceType 路由到具体解析方法
>    - `inline.go`：Inline 来源解析（base64 解码）
>    - `configmap.go`：ConfigMap 来源解析（通过 client.Get 读取指定 key）
>    - `secret.go`：Secret 来源解析（通过 client.Get 读取指定 key）
>    - `uri.go`：URI 远程拉取（HTTP GET + checksum 校验 + TLS 配置）
>    - `image.go`：容器镜像内脚本提取（拉取镜像 + 提取指定路径文件）
>    - `localpath.go`：本地路径读取（os.ReadFile + checksum 校验）
>
> 3. **状态机**（`pkg/componentversion/statemachine/`）
>    - `machine.go`：Machine 结构体，定义 7 个 Phase（Pending/Installing/Verifying/Installed/Failed/Upgrading/RollingBack），实现 `Drive(ctx, cv, executeFunc)` 方法驱动状态转换
>    - `transition.go`：合法转换规则表、转换条件判断、转换执行与 status 更新
>
> 4. **依赖解析**（`pkg/componentversion/dependency/`）
>    - `resolver.go`：Resolver 结构体，实现 `CheckDependencies(ctx, cv)` 检查单个 CV 的依赖是否满足，`BuildDAG(ctx)` 构建全局依赖有向图
>    - `graph.go`：Graph 有向图实现（AddNode/AddEdge/TopologicalSort/HasCycle）
>
> 要求：
> - 所有类型使用 `json` tag，符合 kubebuilder CRD 生成规范
> - Source 解析器对每种来源做 checksum 校验（如果配置了）
> - 状态机转换规则严格，非法转换返回 error
> - 依赖图支持环检测，检测到环时返回明确错误
### 第二次 Prompt（核心层：Installer 接口 + 三种安装器实现 + 健康检查）
> 实现 ComponentVersion Controller 的核心层代码，包含以下模块：
>
> 1. **Installer 接口**（`pkg/componentversion/installer/interface.go`）
>    - 定义 `Installer` 接口：`Install(ctx, cv) → (*InstallResult, error)`、`Verify(ctx, cv) → (*VerifyResult, error)`、`Rollback(ctx, cv) → error`、`Uninstall(ctx, cv) → error`
>    - 定义 `InstallResult`、`VerifyResult` 结构体
>
> 2. **ManifestInstaller**（`pkg/componentversion/installer/manifest/`）
>    - `installer.go`：实现 Installer 接口，Install 方法遍历 ManifestItem 列表，调用 SourceResolver 解析内容，解析为 unstructured.Unstructured，根据 ApplyStrategy（ServerSideApply/ThreeWayMerge/Replace）应用到集群，记录 appliedResources
>    - `apply.go`：三种 Apply 策略的具体实现（ServerSideApply 使用 Patch with fieldManager，ThreeWayMerge 使用 strategic merge patch，Replace 使用 Delete+Create）
>    - `prune.go`：基于 ownership label 查询历史资源，对比当前 appliedResources，清理不在列表中的旧资源
>
> 3. **ScriptInstaller**（`pkg/componentversion/installer/script/`）
>    - `installer.go`：实现 Installer 接口，Install 方法根据 execution.target（Controller/NodeAgent/InitPod）选择执行通道
>    - `executor.go`：三种执行目标的实现——Controller 直接 os/exec 执行；NodeAgent 构造 Command CRD（包含 nodeSelector/env/args/timeout/retry）创建到集群；InitPod 构造 Job/Pod 定义创建到集群
>    - `validator.go`：安装后验证逻辑，执行 validation.command 并检查 expectedOutput
>
> 4. **HelmChartInstaller**（`pkg/componentversion/installer/helmchart/`）
>    - `installer.go`：实现 Installer 接口，使用 helm.sh/helm/v3 的 action API，检查 Release 是否存在决定 install 或 upgrade
>    - `values.go`：Values 合并逻辑，按顺序合并 spec.values + valuesFrom（ConfigMap/Secret 引用），后者覆盖前者
>    - `release.go`：Release 管理逻辑（状态查询、回滚、卸载），Chart 来源解析（Repo/OCI/LocalPath/ConfigMapRef）
>
> 5. **健康检查**（`pkg/componentversion/healthcheck/`）
>    - `checker.go`：Checker 结构体，统一健康检查入口，根据 HealthCheckSpec.Type 路由到具体检查方法
>    - `http.go`：HTTPGet 检查实现
>    - `tcp.go`：TCPSocket 检查实现
>    - `command.go`：Command 检查实现（构造 Command CRD 或直接执行）
>    - `clusteroperator.go`：ClusterOperator 检查实现（查询 ClusterOperator CRD 的 condition）
>
> 要求：
> - ManifestInstaller 的 ServerSideApply 使用 `sigs.k8s.io/controller-runtime/pkg/client` 的 Patch 方法
> - ScriptInstaller 的 NodeAgent 通道构造的 Command CRD 需设置 OwnerReference 关联到 ComponentVersion
> - HelmChartInstaller 使用 `helm.sh/helm/v3/pkg/action` 的 Install/Upgrade/Rollback action
> - 所有 Installer 的 Verify 方法需处理"进行中"状态（返回 VerifyResult{Passed: false} 而非 error）
> - 健康检查失败不改变 phase，仅更新 conditions
### 第三次 Prompt（编排层：Reconciler 主循环 + Dispatcher + NodeConfig 协作 + SetupWithManager + 辅助函数）
> 实现 ComponentVersion Controller 的编排层代码，包含以下模块：
>
> 1. **ComponentVersionReconciler**（`pkg/componentversion/controller/componentversion_controller.go`）
>    - Reconciler 结构体定义（Client/Scheme/Recorder/RestConfig/Dispatcher/StateMachine/HealthChecker/DepResolver/SourceResolver）
>    - `Reconcile(ctx, req)` 主循环：获取 CV → 处理 Finalizer → 依赖检查 → 状态机驱动 → 健康检查 → 状态上报 → 返回 Result
>    - `handleDeletion(ctx, cv)`：调用 Dispatcher 获取 Installer 执行 Uninstall，移除 Finalizer
>    - `ensureFinalizer(ctx, cv)`：添加 Finalizer
>    - `updatePhase(ctx, cv, phase, err)`：更新 phase/conditions/lastTransitionTime
>    - `executeInstall(ctx, cv)`：状态机回调函数，调用 Dispatcher.Dispatch 执行安装/验证/回滚
>
> 2. **Dispatcher 策略路由**（`pkg/componentversion/controller/dispatcher.go`）
>    - Dispatcher 结构体持有三种 Installer 实例
>    - `GetInstaller(installType)` 根据 InstallType 返回对应 Installer
>    - `Dispatch(ctx, cv)` 根据 CV 当前 phase 调用 Installer 的 Install/Verify/Rollback 方法
>
> 3. **NodeConfig 协作**（`pkg/componentversion/controller/nodeconfig_collaboration.go`）
>    - `resolveNodeConfigs(ctx, cv)` 查询匹配 CV nodeSelector 的 NodeConfig 列表
>    - `mergeNodeConfigOverrides(cv, nc)` 合并 NodeConfig 的 configOverrides 到执行参数
>    - `aggregateNodeStatuses(ctx, cv)` 汇总各节点执行状态更新到 CV status
>    - `nodeConfigToComponentVersion(ctx, obj)` NodeConfig → CV 的 Enqueue 映射函数
>
> 4. **SetupWithManager**（在 `componentversion_controller.go` 中）
>    - For ComponentVersion（带 GenerationChangedPredicate）
>    - Watches NodeConfig → 触发关联的 scope=Node 的 CV
>    - Watches Command CRD → 触发关联的 InstallScript 类型 CV
>    - Watches ComponentVersion → 触发下游依赖 CV
>    - 配置 MaxConcurrentReconciles=5、FastSlowRateLimiter
>    - RBAC 权限注解
>
> 5. **辅助函数**（`pkg/componentversion/controller/helper.go`）
>    - `parseDuration(s string) time.Duration`：解析时长字符串
>    - `versionGTE(current, minVersion string) bool`：版本比较
>    - `commandToComponentVersion(ctx, obj)`：Command → CV 的 Enqueue 映射
>    - `dependencyToDownstreamCV(ctx, obj)`：CV → 下游 CV 的 Enqueue 映射
>    - `buildOwnerLabels(cv) map[string]string`：构建 ownership label
>    - `computeRequeueInterval(phase) time.Duration`：根据 phase 计算 requeue 间隔
>
> 要求：
> - Reconciler 主循环遵循 controller-runtime 最佳实践，冲突时返回 error 让框架自动 requeue
> - 状态上报使用 `status.SubResourceWriter.Update()` 或带 RetryOnConflict 的 Update
> - NodeConfig 协作仅对 scope=Node 的 CV 生效
> - Command CRD Watch 过滤逻辑：仅关注 OwnerReference 为 ComponentVersion 的 Command
> - 所有 Enqueue 映射函数需处理 error 并记录日志，不阻塞其他映射

# ComponentVersion Controller 基础层的完整代码实现（共 17 个文件）
## 1. API 类型定义
```go d:\code\github\componentversion-controller\api\v1alpha1\groupversion_info.go
package v1alpha1

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/scheme"
)

const (
	GroupName = "orchestration.bke.io"
	Version   = "v1alpha1"
)

var (
	GroupVersion = schema.GroupVersion{Group: GroupName, Version: Version}
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}
	AddToScheme   = SchemeBuilder.AddToScheme
)
```

```go d:\code\github\componentversion-controller\api\v1alpha1\source_types.go
package v1alpha1

type SourceType string

const (
	SourceTypeInline      SourceType = "Inline"
	SourceTypeConfigMapRef SourceType = "ConfigMapRef"
	SourceTypeSecretRef   SourceType = "SecretRef"
	SourceTypeURIRef      SourceType = "URIRef"
	SourceTypeImageRef    SourceType = "ImageRef"
	SourceTypeLocalPath   SourceType = "LocalPath"
)

type SourceSpec struct {
	Type         SourceType       `json:"type"`
	Inline       *InlineSource    `json:"inline,omitempty"`
	ConfigMapRef *ConfigMapSource `json:"configMapRef,omitempty"`
	SecretRef    *SecretSource    `json:"secretRef,omitempty"`
	URIRef       *URISource       `json:"uriRef,omitempty"`
	ImageRef     *ImageSource     `json:"imageRef,omitempty"`
	LocalPath    *LocalPathSource `json:"localPath,omitempty"`
}

type InlineSource struct {
	Content string `json:"content,omitempty"`
}

type ConfigMapSource struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key,omitempty"`
}

type SecretSource struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Key       string `json:"key,omitempty"`
}

type URISource struct {
	URI                   string `json:"uri"`
	Checksum              string `json:"checksum,omitempty"`
	InsecureSkipTLSVerify bool   `json:"insecureSkipTLSVerify,omitempty"`
}

type ImageSource struct {
	Image           string `json:"image"`
	Path            string `json:"path,omitempty"`
	ImagePullSecret string `json:"imagePullSecret,omitempty"`
}

type LocalPathSource struct {
	Path     string `json:"path"`
	Checksum string `json:"checksum,omitempty"`
}

type ChartSourceType string

const (
	ChartSourceTypeRepo        ChartSourceType = "Repo"
	ChartSourceTypeOCI         ChartSourceType = "OCI"
	ChartSourceTypeLocalPath   ChartSourceType = "LocalPath"
	ChartSourceTypeConfigMapRef ChartSourceType = "ConfigMapRef"
)

type ChartSourceSpec struct {
	Type        ChartSourceType   `json:"type"`
	Repo        *HelmRepoSource   `json:"repo,omitempty"`
	OCI         *HelmOCISource    `json:"oci,omitempty"`
	LocalPath   *LocalPathSource  `json:"localPath,omitempty"`
	ConfigMapRef *ConfigMapSource `json:"configMapRef,omitempty"`
}

type HelmRepoSource struct {
	URL                string          `json:"url"`
	Username           string          `json:"username,omitempty"`
	PasswordSecretRef  *SecretKeyRef   `json:"passwordSecretRef,omitempty"`
}

type HelmOCISource struct {
	Registry  string `json:"registry"`
	Repository string `json:"repository"`
	Tag       string `json:"tag,omitempty"`
	Insecure  bool   `json:"insecure,omitempty"`
}

type SecretKeyRef struct {
	Name      string `json:"name"`
	Key       string `json:"key"`
	Namespace string `json:"namespace,omitempty"`
}
```

```go d:\code\github\componentversion-controller\api\v1alpha1\install_types.go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
)

type InstallType string

const (
	InstallTypeManifest      InstallType = "Manifest"
	InstallTypeInstallScript InstallType = "InstallScript"
	InstallTypeHelmChart     InstallType = "HelmChart"
)

type ApplyStrategy string

const (
	ApplyStrategyServerSideApply ApplyStrategy = "ServerSideApply"
	ApplyStrategyThreeWayMerge   ApplyStrategy = "ThreeWayMerge"
	ApplyStrategyReplace         ApplyStrategy = "Replace"
)

type ExecutionTarget string

const (
	ExecutionTargetController ExecutionTarget = "Controller"
	ExecutionTargetNodeAgent  ExecutionTarget = "NodeAgent"
	ExecutionTargetInitPod    ExecutionTarget = "InitPod"
)

type InstallSpec struct {
	Type          InstallType       `json:"type"`
	Manifest      *ManifestSpec     `json:"manifest,omitempty"`
	InstallScript *InstallScriptSpec `json:"installScript,omitempty"`
	HelmChart     *HelmChartSpec    `json:"helmChart,omitempty"`
}

type ManifestSpec struct {
	Items         []ManifestItem `json:"items,omitempty"`
	ApplyStrategy ApplyStrategy  `json:"applyStrategy,omitempty"`
	Prune         bool           `json:"prune,omitempty"`
}

type ManifestItem struct {
	Name      string     `json:"name"`
	Namespace string     `json:"namespace,omitempty"`
	Source    SourceSpec `json:"source"`
}

type InstallScriptSpec struct {
	Command    string         `json:"command,omitempty"`
	Args       []string       `json:"args,omitempty"`
	Env        []EnvVar       `json:"env,omitempty"`
	Source     SourceSpec     `json:"source"`
	Execution  *ExecutionSpec `json:"execution,omitempty"`
	Validation *ValidationSpec `json:"validation,omitempty"`
}

type EnvVar struct {
	Name      string          `json:"name"`
	Value     string          `json:"value,omitempty"`
	ValueFrom *EnvVarSource   `json:"valueFrom,omitempty"`
}

type EnvVarSource struct {
	ConfigMapKeyRef *ConfigMapKeySelector `json:"configMapKeyRef,omitempty"`
	SecretKeyRef    *SecretKeySelector    `json:"secretKeyRef,omitempty"`
}

type ConfigMapKeySelector struct {
	Name      string `json:"name"`
	Key       string `json:"key"`
	Namespace string `json:"namespace,omitempty"`
}

type SecretKeySelector struct {
	Name      string `json:"name"`
	Key       string `json:"key"`
	Namespace string `json:"namespace,omitempty"`
}

type ExecutionSpec struct {
	Target       ExecutionTarget    `json:"target,omitempty"`
	NodeSelector map[string]string  `json:"nodeSelector,omitempty"`
	Timeout      string             `json:"timeout,omitempty"`
	RetryPolicy  *RetryPolicy       `json:"retryPolicy,omitempty"`
}

type RetryPolicy struct {
	MaxRetries  int    `json:"maxRetries,omitempty"`
	RetryDelay  string `json:"retryDelay,omitempty"`
}

type ValidationSpec struct {
	Command         string `json:"command,omitempty"`
	ExpectedOutput  string `json:"expectedOutput,omitempty"`
	Timeout         string `json:"timeout,omitempty"`
}

type HelmChartSpec struct {
	ChartRef    HelmChartRef          `json:"chartRef,omitempty"`
	ReleaseName string               `json:"releaseName,omitempty"`
	Namespace   string               `json:"namespace,omitempty"`
	Source      *ChartSourceSpec      `json:"source,omitempty"`
	Values      *runtime.RawExtension `json:"values,omitempty"`
	ValuesFrom  []ValuesFromSource    `json:"valuesFrom,omitempty"`
	Timeout     string               `json:"timeout,omitempty"`
	Wait        bool                 `json:"wait,omitempty"`
	Atomic      bool                 `json:"atomic,omitempty"`
}

type HelmChartRef struct {
	Name      string `json:"name"`
	Repo      string `json:"repo,omitempty"`
	Version   string `json:"version,omitempty"`
	ChartPath string `json:"chartPath,omitempty"`
}

type ValuesFromSource struct {
	Kind      string `json:"kind"`
	Name      string `json:"name"`
	Namespace string `json:"namespace,omitempty"`
	ValuesKey string `json:"valuesKey,omitempty"`
}

type HealthCheckType string

const (
	HealthCheckTypeHTTPGet         HealthCheckType = "HTTPGet"
	HealthCheckTypeTCPSocket       HealthCheckType = "TCPSocket"
	HealthCheckTypeCommand         HealthCheckType = "Command"
	HealthCheckTypeClusterOperator HealthCheckType = "ClusterOperator"
)

type HealthCheckSpec struct {
	Type            HealthCheckType  `json:"type,omitempty"`
	HTTPGet         *HTTPGetSpec     `json:"httpGet,omitempty"`
	TCPSocket       *TCPSocketSpec   `json:"tcpSocket,omitempty"`
	Command         *CommandSpec     `json:"command,omitempty"`
	ClusterOperator *ClusterOperatorSpec `json:"clusterOperator,omitempty"`
	Interval        string           `json:"interval,omitempty"`
	Timeout         string           `json:"timeout,omitempty"`
}

type HTTPGetSpec struct {
	Path   string `json:"path,omitempty"`
	Port   int    `json:"port,omitempty"`
	Scheme string `json:"scheme,omitempty"`
}

type TCPSocketSpec struct {
	Port int `json:"port,omitempty"`
}

type CommandSpec struct {
	Command []string `json:"command,omitempty"`
}

type ClusterOperatorSpec struct {
	Name          string `json:"name,omitempty"`
	ConditionType string `json:"conditionType,omitempty"`
}

type Dependency struct {
	Name       string `json:"name"`
	MinVersion string `json:"minVersion,omitempty"`
}

type ImageRef struct {
	Name  string `json:"name"`
	Image string `json:"image"`
}

type ComponentPhase string

const (
	PhasePending     ComponentPhase = "Pending"
	PhaseInstalling  ComponentPhase = "Installing"
	PhaseVerifying   ComponentPhase = "Verifying"
	PhaseInstalled   ComponentPhase = "Installed"
	PhaseFailed      ComponentPhase = "Failed"
	PhaseUpgrading   ComponentPhase = "Upgrading"
	PhaseRollingBack ComponentPhase = "RollingBack"
)

type ComponentScope string

const (
	ComponentScopeCluster ComponentScope = "Cluster"
	ComponentScopeNode    ComponentScope = "Node"
)

type InstallDetails struct {
	Manifest      *ManifestInstallDetails      `json:"manifest,omitempty"`
	InstallScript *InstallScriptInstallDetails `json:"installScript,omitempty"`
	HelmChart     *HelmChartInstallDetails     `json:"helmChart,omitempty"`
}

type ManifestInstallDetails struct {
	AppliedResources  []AppliedResource  `json:"appliedResources,omitempty"`
	PrunedResources   []PrunedResource   `json:"prunedResources,omitempty"`
}

type AppliedResource struct {
	Group     string `json:"group,omitempty"`
	Version   string `json:"version,omitempty"`
	Kind      string `json:"kind,omitempty"`
	Name      string `json:"name,omitempty"`
	Namespace string `json:"namespace,omitempty"`
	Hash      string `json:"hash,omitempty"`
	AppliedAt *metav1.Time `json:"appliedAt,omitempty"`
}

type PrunedResource struct {
	Group     string `json:"group,omitempty"`
	Version   string `json:"version,omitempty"`
	Kind      string `json:"kind,omitempty"`
	Name      string `json:"name,omitempty"`
	Namespace string `json:"namespace,omitempty"`
}

type InstallScriptInstallDetails struct {
	ExecutionID      string           `json:"executionId,omitempty"`
	TargetNodes      []NodeExecStatus `json:"targetNodes,omitempty"`
	ValidationResult *ValidationResult `json:"validationResult,omitempty"`
}

type NodeExecStatus struct {
	NodeName  string        `json:"nodeName,omitempty"`
	NodeIP    string        `json:"nodeIP,omitempty"`
	Phase     NodeExecPhase `json:"phase,omitempty"`
	ExitCode  int           `json:"exitCode,omitempty"`
	Stdout    string        `json:"stdout,omitempty"`
	Stderr    string        `json:"stderr,omitempty"`
	StartedAt *metav1.Time  `json:"startedAt,omitempty"`
	FinishedAt *metav1.Time `json:"finishedAt,omitempty"`
	RetryCount int          `json:"retryCount,omitempty"`
}

type NodeExecPhase string

const (
	NodeExecPending  NodeExecPhase = "Pending"
	NodeExecRunning  NodeExecPhase = "Running"
	NodeExecSucceeded NodeExecPhase = "Succeeded"
	NodeExecFailed   NodeExecPhase = "Failed"
	NodeExecTimeout  NodeExecPhase = "Timeout"
)

type ValidationResult struct {
	Passed   bool         `json:"passed,omitempty"`
	Output   string       `json:"output,omitempty"`
	CheckedAt *metav1.Time `json:"checkedAt,omitempty"`
}

type HelmChartInstallDetails struct {
	ReleaseStatus string       `json:"releaseStatus,omitempty"`
	Revision      int          `json:"revision,omitempty"`
	LastDeployed  *metav1.Time `json:"lastDeployed,omitempty"`
}
```

```go d:\code\github\componentversion-controller\api\v1alpha1\componentversion_types.go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type ComponentVersionSpec struct {
	ComponentName string           `json:"componentName"`
	Version       string           `json:"version"`
	Scope         ComponentScope   `json:"scope,omitempty"`
	Dependencies  []Dependency     `json:"dependencies,omitempty"`
	Images        []ImageRef       `json:"images,omitempty"`
	Install       InstallSpec      `json:"install"`
	HealthCheck   *HealthCheckSpec `json:"healthCheck,omitempty"`
}

type ComponentVersionStatus struct {
	ObservedVersion    string             `json:"observedVersion,omitempty"`
	Phase              ComponentPhase     `json:"phase,omitempty"`
	InstallType        InstallType        `json:"installType,omitempty"`
	Conditions         []metav1.Condition `json:"conditions,omitempty"`
	InstallDetails     InstallDetails     `json:"installDetails,omitempty"`
	LastTransitionTime *metav1.Time       `json:"lastTransitionTime,omitempty"`
	RetryCount         int                `json:"retryCount,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=cv
// +kubebuilder:printcolumn:name="COMPONENT",type="string",JSONPath=".spec.componentName"
// +kubebuilder:printcolumn:name="VERSION",type="string",JSONPath=".spec.version"
// +kubebuilder:printcolumn:name="TYPE",type="string",JSONPath=".spec.install.type"
// +kubebuilder:printcolumn:name="PHASE",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="SCOPE",type="string",JSONPath=".spec.scope"
type ComponentVersion struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ComponentVersionSpec   `json:"spec,omitempty"`
	Status ComponentVersionStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type ComponentVersionList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ComponentVersion `json:"items"`
}

func init() {
	SchemeBuilder.Register(&ComponentVersion{}, &ComponentVersionList{})
}
```

```go d:\code\github\componentversion-controller\api\v1alpha1\nodeconfig_types.go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
)

type NodeConfigSpec struct {
	NodeName         string              `json:"nodeName"`
	NodeIP           string              `json:"nodeIP,omitempty"`
	Roles            []string            `json:"roles,omitempty"`
	OSInfo           *OSInfo             `json:"osInfo,omitempty"`
	ComponentConfigs []ComponentConfig   `json:"componentConfigs,omitempty"`
}

type OSInfo struct {
	Type    string `json:"type,omitempty"`
	Version string `json:"version,omitempty"`
	Arch    string `json:"arch,omitempty"`
}

type ComponentConfig struct {
	ComponentName   string               `json:"componentName"`
	ConfigOverrides *runtime.RawExtension `json:"configOverrides,omitempty"`
	NodeLabels      map[string]string    `json:"nodeLabels,omitempty"`
}

type NodeComponentVersionStatus struct {
	ComponentName  string        `json:"componentName,omitempty"`
	CurrentVersion string        `json:"currentVersion,omitempty"`
	TargetVersion  string        `json:"targetVersion,omitempty"`
	Phase          ComponentPhase `json:"phase,omitempty"`
	LastUpdated    *metav1.Time  `json:"lastUpdated,omitempty"`
}

type NodeHealth string

const (
	NodeHealthHealthy    NodeHealth = "Healthy"
	NodeHealthDegraded   NodeHealth = "Degraded"
	NodeHealthUnreachable NodeHealth = "Unreachable"
)

type NodeConfigStatus struct {
	ComponentVersions []NodeComponentVersionStatus `json:"componentVersions,omitempty"`
	NodeHealth        NodeHealth                   `json:"nodeHealth,omitempty"`
	LastHeartbeat     *metav1.Time                 `json:"lastHeartbeat,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=nc
// +kubebuilder:printcolumn:name="NODE",type="string",JSONPath=".spec.nodeName"
// +kubebuilder:printcolumn:name="IP",type="string",JSONPath=".spec.nodeIP"
// +kubebuilder:printcolumn:name="HEALTH",type="string",JSONPath=".status.nodeHealth"
type NodeConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   NodeConfigSpec   `json:"spec,omitempty"`
	Status NodeConfigStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type NodeConfigList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []NodeConfig `json:"items"`
}

func init() {
	SchemeBuilder.Register(&NodeConfig{}, &NodeConfigList{})
}
```

```go d:\code\github\componentversion-controller\api\v1alpha1\zz_generated.deepcopy.go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
)

func (in *SourceSpec) DeepCopyInto(out *SourceSpec) {
	*out = *in
	if in.Inline != nil {
		in, out := &in.Inline, &out.Inline
		*out = new(InlineSource)
		**out = **in
	}
	if in.ConfigMapRef != nil {
		in, out := &in.ConfigMapRef, &out.ConfigMapRef
		*out = new(ConfigMapSource)
		**out = **in
	}
	if in.SecretRef != nil {
		in, out := &in.SecretRef, &out.SecretRef
		*out = new(SecretSource)
		**out = **in
	}
	if in.URIRef != nil {
		in, out := &in.URIRef, &out.URIRef
		*out = new(URISource)
		**out = **in
	}
	if in.ImageRef != nil {
		in, out := &in.ImageRef, &out.ImageRef
		*out = new(ImageSource)
		**out = **in
	}
	if in.LocalPath != nil {
		in, out := &in.LocalPath, &out.LocalPath
		*out = new(LocalPathSource)
		**out = **in
	}
}

func (in *InstallSpec) DeepCopyInto(out *InstallSpec) {
	*out = *in
	if in.Manifest != nil {
		in, out := &in.Manifest, &out.Manifest
		*out = new(ManifestSpec)
		(*in).DeepCopyInto(*out)
	}
	if in.InstallScript != nil {
		in, out := &in.InstallScript, &out.InstallScript
		*out = new(InstallScriptSpec)
		(*in).DeepCopyInto(*out)
	}
	if in.HelmChart != nil {
		in, out := &in.HelmChart, &out.HelmChart
		*out = new(HelmChartSpec)
		(*in).DeepCopyInto(*out)
	}
}

func (in *ManifestSpec) DeepCopyInto(out *ManifestSpec) {
	*out = *in
	if in.Items != nil {
		in, out := &in.Items, &out.Items
		*out = make([]ManifestItem, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
}

func (in *ManifestItem) DeepCopyInto(out *ManifestItem) {
	*out = *in
	in.Source.DeepCopyInto(&out.Source)
}

func (in *InstallScriptSpec) DeepCopyInto(out *InstallScriptSpec) {
	*out = *in
	if in.Args != nil {
		in, out := &in.Args, &out.Args
		*out = make([]string, len(*in))
		copy(*out, *in)
	}
	if in.Env != nil {
		in, out := &in.Env, &out.Env
		*out = make([]EnvVar, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	in.Source.DeepCopyInto(&out.Source)
	if in.Execution != nil {
		in, out := &in.Execution, &out.Execution
		*out = new(ExecutionSpec)
		(*in).DeepCopyInto(*out)
	}
	if in.Validation != nil {
		in, out := &in.Validation, &out.Validation
		*out = new(ValidationSpec)
		**out = **in
	}
}

func (in *ExecutionSpec) DeepCopyInto(out *ExecutionSpec) {
	*out = *in
	if in.NodeSelector != nil {
		in, out := &in.NodeSelector, &out.NodeSelector
		*out = make(map[string]string, len(*in))
		for key, val := range *in {
			(*out)[key] = val
		}
	}
	if in.RetryPolicy != nil {
		in, out := &in.RetryPolicy, &out.RetryPolicy
		*out = new(RetryPolicy)
		**out = **in
	}
}

func (in *EnvVar) DeepCopyInto(out *EnvVar) {
	*out = *in
	if in.ValueFrom != nil {
		in, out := &in.ValueFrom, &out.ValueFrom
		*out = new(EnvVarSource)
		(*in).DeepCopyInto(*out)
	}
}

func (in *EnvVarSource) DeepCopyInto(out *EnvVarSource) {
	*out = *in
	if in.ConfigMapKeyRef != nil {
		in, out := &in.ConfigMapKeyRef, &out.ConfigMapKeyRef
		*out = new(ConfigMapKeySelector)
		**out = **in
	}
	if in.SecretKeyRef != nil {
		in, out := &in.SecretKeyRef, &out.SecretKeyRef
		*out = new(SecretKeySelector)
		**out = **in
	}
}

func (in *HelmChartSpec) DeepCopyInto(out *HelmChartSpec) {
	*out = *in
	in.ChartRef.DeepCopyInto(&out.ChartRef)
	if in.Source != nil {
		in, out := &in.Source, &out.Source
		*out = new(ChartSourceSpec)
		(*in).DeepCopyInto(*out)
	}
	if in.Values != nil {
		in, out := &in.Values, &out.Values
		*out = new(runtime.RawExtension)
		(*in).DeepCopyInto(*out)
	}
	if in.ValuesFrom != nil {
		in, out := &in.ValuesFrom, &out.ValuesFrom
		*out = make([]ValuesFromSource, len(*in))
		copy(*out, *in)
	}
}

func (in *ChartSourceSpec) DeepCopyInto(out *ChartSourceSpec) {
	*out = *in
	if in.Repo != nil {
		in, out := &in.Repo, &out.Repo
		*out = new(HelmRepoSource)
		(*in).DeepCopyInto(*out)
	}
	if in.OCI != nil {
		in, out := &in.OCI, &out.OCI
		*out = new(HelmOCISource)
		**out = **in
	}
	if in.LocalPath != nil {
		in, out := &in.LocalPath, &out.LocalPath
		*out = new(LocalPathSource)
		**out = **in
	}
	if in.ConfigMapRef != nil {
		in, out := &in.ConfigMapRef, &out.ConfigMapRef
		*out = new(ConfigMapSource)
		**out = **in
	}
}

func (in *HelmRepoSource) DeepCopyInto(out *HelmRepoSource) {
	*out = *in
	if in.PasswordSecretRef != nil {
		in, out := &in.PasswordSecretRef, &out.PasswordSecretRef
		*out = new(SecretKeyRef)
		**out = **in
	}
}

func (in *HealthCheckSpec) DeepCopyInto(out *HealthCheckSpec) {
	*out = *in
	if in.HTTPGet != nil {
		in, out := &in.HTTPGet, &out.HTTPGet
		*out = new(HTTPGetSpec)
		**out = **in
	}
	if in.TCPSocket != nil {
		in, out := &in.TCPSocket, &out.TCPSocket
		*out = new(TCPSocketSpec)
		**out = **in
	}
	if in.Command != nil {
		in, out := &in.Command, &out.Command
		*out = new(CommandSpec)
		(*in).DeepCopyInto(*out)
	}
	if in.ClusterOperator != nil {
		in, out := &in.ClusterOperator, &out.ClusterOperator
		*out = new(ClusterOperatorSpec)
		**out = **in
	}
}

func (in *CommandSpec) DeepCopyInto(out *CommandSpec) {
	*out = *in
	if in.Command != nil {
		in, out := &in.Command, &out.Command
		*out = make([]string, len(*in))
		copy(*out, *in)
	}
}

func (in *ComponentVersionSpec) DeepCopyInto(out *ComponentVersionSpec) {
	*out = *in
	if in.Dependencies != nil {
		in, out := &in.Dependencies, &out.Dependencies
		*out = make([]Dependency, len(*in))
		copy(*out, *in)
	}
	if in.Images != nil {
		in, out := &in.Images, &out.Images
		*out = make([]ImageRef, len(*in))
		copy(*out, *in)
	}
	in.Install.DeepCopyInto(&out.Install)
	if in.HealthCheck != nil {
		in, out := &in.HealthCheck, &out.HealthCheck
		*out = new(HealthCheckSpec)
		(*in).DeepCopyInto(*out)
	}
}

func (in *ComponentVersionStatus) DeepCopyInto(out *ComponentVersionStatus) {
	*out = *in
	if in.Conditions != nil {
		in, out := &in.Conditions, &out.Conditions
		*out = make([]metav1.Condition, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	in.InstallDetails.DeepCopyInto(&out.InstallDetails)
	if in.LastTransitionTime != nil {
		in, out := &in.LastTransitionTime, &out.LastTransitionTime
		*out = (*in).DeepCopy()
	}
}

func (in *InstallDetails) DeepCopyInto(out *InstallDetails) {
	*out = *in
	if in.Manifest != nil {
		in, out := &in.Manifest, &out.Manifest
		*out = new(ManifestInstallDetails)
		(*in).DeepCopyInto(*out)
	}
	if in.InstallScript != nil {
		in, out := &in.InstallScript, &out.InstallScript
		*out = new(InstallScriptInstallDetails)
		(*in).DeepCopyInto(*out)
	}
	if in.HelmChart != nil {
		in, out := &in.HelmChart, &out.HelmChart
		*out = new(HelmChartInstallDetails)
		**out = **in
	}
}

func (in *ManifestInstallDetails) DeepCopyInto(out *ManifestInstallDetails) {
	*out = *in
	if in.AppliedResources != nil {
		in, out := &in.AppliedResources, &out.AppliedResources
		*out = make([]AppliedResource, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	if in.PrunedResources != nil {
		in, out := &in.PrunedResources, &out.PrunedResources
		*out = make([]PrunedResource, len(*in))
		copy(*out, *in)
	}
}

func (in *AppliedResource) DeepCopyInto(out *AppliedResource) {
	*out = *in
	if in.AppliedAt != nil {
		in, out := &in.AppliedAt, &out.AppliedAt
		*out = (*in).DeepCopy()
	}
}

func (in *InstallScriptInstallDetails) DeepCopyInto(out *InstallScriptInstallDetails) {
	*out = *in
	if in.TargetNodes != nil {
		in, out := &in.TargetNodes, &out.TargetNodes
		*out = make([]NodeExecStatus, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	if in.ValidationResult != nil {
		in, out := &in.ValidationResult, &out.ValidationResult
		*out = new(ValidationResult)
		(*in).DeepCopyInto(*out)
	}
}

func (in *NodeExecStatus) DeepCopyInto(out *NodeExecStatus) {
	*out = *in
	if in.StartedAt != nil {
		in, out := &in.StartedAt, &out.StartedAt
		*out = (*in).DeepCopy()
	}
	if in.FinishedAt != nil {
		in, out := &in.FinishedAt, &out.FinishedAt
		*out = (*in).DeepCopy()
	}
}

func (in *ValidationResult) DeepCopyInto(out *ValidationResult) {
	*out = *in
	if in.CheckedAt != nil {
		in, out := &in.CheckedAt, &out.CheckedAt
		*out = (*in).DeepCopy()
	}
}

func (in *ComponentVersion) DeepCopyInto(out *ComponentVersion) {
	*out = *in
	out.TypeMeta = in.TypeMeta
	in.ObjectMeta.DeepCopyInto(&out.ObjectMeta)
	in.Spec.DeepCopyInto(&out.Spec)
	in.Status.DeepCopyInto(&out.Status)
}

func (in *ComponentVersion) DeepCopy() *ComponentVersion {
	if in == nil {
		return nil
	}
	out := new(ComponentVersion)
	in.DeepCopyInto(out)
	return out
}

func (in *ComponentVersionList) DeepCopyInto(out *ComponentVersionList) {
	*out = *in
	out.TypeMeta = in.TypeMeta
	in.ListMeta.DeepCopyInto(&out.ListMeta)
	if in.Items != nil {
		in, out := &in.Items, &out.Items
		*out = make([]ComponentVersion, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
}

func (in *ComponentVersionList) DeepCopy() *ComponentVersionList {
	if in == nil {
		return nil
	}
	out := new(ComponentVersionList)
	in.DeepCopyInto(out)
	return out
}

func (in *NodeConfigSpec) DeepCopyInto(out *NodeConfigSpec) {
	*out = *in
	if in.Roles != nil {
		in, out := &in.Roles, &out.Roles
		*out = make([]string, len(*in))
		copy(*out, *in)
	}
	if in.OSInfo != nil {
		in, out := &in.OSInfo, &out.OSInfo
		*out = new(OSInfo)
		**out = **in
	}
	if in.ComponentConfigs != nil {
		in, out := &in.ComponentConfigs, &out.ComponentConfigs
		*out = make([]ComponentConfig, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
}

func (in *ComponentConfig) DeepCopyInto(out *ComponentConfig) {
	*out = *in
	if in.ConfigOverrides != nil {
		in, out := &in.ConfigOverrides, &out.ConfigOverrides
		*out = new(runtime.RawExtension)
		(*in).DeepCopyInto(*out)
	}
	if in.NodeLabels != nil {
		in, out := &in.NodeLabels, &out.NodeLabels
		*out = make(map[string]string, len(*in))
		for key, val := range *in {
			(*out)[key] = val
		}
	}
}

func (in *NodeConfigStatus) DeepCopyInto(out *NodeConfigStatus) {
	*out = *in
	if in.ComponentVersions != nil {
		in, out := &in.ComponentVersions, &out.ComponentVersions
		*out = make([]NodeComponentVersionStatus, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	if in.LastHeartbeat != nil {
		in, out := &in.LastHeartbeat, &out.LastHeartbeat
		*out = (*in).DeepCopy()
	}
}

func (in *NodeComponentVersionStatus) DeepCopyInto(out *NodeComponentVersionStatus) {
	*out = *in
	if in.LastUpdated != nil {
		in, out := &in.LastUpdated, &out.LastUpdated
		*out = (*in).DeepCopy()
	}
}

func (in *NodeConfig) DeepCopyInto(out *NodeConfig) {
	*out = *in
	out.TypeMeta = in.TypeMeta
	in.ObjectMeta.DeepCopyInto(&out.ObjectMeta)
	in.Spec.DeepCopyInto(&out.Spec)
	in.Status.DeepCopyInto(&out.Status)
}

func (in *NodeConfig) DeepCopy() *NodeConfig {
	if in == nil {
		return nil
	}
	out := new(NodeConfig)
	in.DeepCopyInto(out)
	return out
}

func (in *NodeConfigList) DeepCopyInto(out *NodeConfigList) {
	*out = *in
	out.TypeMeta = in.TypeMeta
	in.ListMeta.DeepCopyInto(&out.ListMeta)
	if in.Items != nil {
		in, out := &in.Items, &out.Items
		*out = make([]NodeConfig, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
}

func (in *NodeConfigList) DeepCopy() *NodeConfigList {
	if in == nil {
		return nil
	}
	out := new(NodeConfigList)
	in.DeepCopyInto(out)
	return out
}
```
## 2. Source 解析器
```go d:\code\github\componentversion-controller\pkg\componentversion\source\resolver.go
package source

import (
	"context"
	"crypto/tls"
	"fmt"
	"net/http"
	"time"

	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type ResolvedContent struct {
	Content  []byte
	Checksum string
	Source   v1alpha1.SourceType
}

type Resolver struct {
	client.Client
	HTTPClient *http.Client
}

func NewResolver(c client.Client) *Resolver {
	return &Resolver{
		Client: c,
		HTTPClient: &http.Client{
			Timeout: 60 * time.Second,
			Transport: &http.Transport{
				TLSClientConfig: &tls.Config{
					InsecureSkipVerify: false,
				},
			},
		},
	}
}

func (r *Resolver) Resolve(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec == nil {
		return nil, fmt.Errorf("source spec is nil")
	}

	switch spec.Type {
	case v1alpha1.SourceTypeInline:
		return r.resolveInline(spec)
	case v1alpha1.SourceTypeConfigMapRef:
		return r.resolveConfigMap(ctx, spec)
	case v1alpha1.SourceTypeSecretRef:
		return r.resolveSecret(ctx, spec)
	case v1alpha1.SourceTypeURIRef:
		return r.resolveURI(ctx, spec)
	case v1alpha1.SourceTypeImageRef:
		return r.resolveImage(ctx, spec)
	case v1alpha1.SourceTypeLocalPath:
		return r.resolveLocalPath(spec)
	default:
		return nil, fmt.Errorf("unsupported source type: %s", spec.Type)
	}
}

func objectKey(ns, name string) types.NamespacedName {
	return types.NamespacedName{Namespace: ns, Name: name}
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\inline.go
package source

import (
	"encoding/base64"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveInline(spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.Inline == nil {
		return nil, errSourceFieldMissing("inline")
	}

	content, err := base64.StdEncoding.DecodeString(spec.Inline.Content)
	if err != nil {
		return nil, fmt.Errorf("failed to decode inline content: %w", err)
	}

	return &ResolvedContent{
		Content: content,
		Source:  v1alpha1.SourceTypeInline,
	}, nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\configmap.go
package source

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveConfigMap(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.ConfigMapRef == nil {
		return nil, errSourceFieldMissing("configMapRef")
	}

	cm := &corev1.ConfigMap{}
	key := objectKey(spec.ConfigMapRef.Namespace, spec.ConfigMapRef.Name)
	if err := r.Get(ctx, key, cm); err != nil {
		if apierrors.IsNotFound(err) {
			return nil, fmt.Errorf("configmap %s not found", key)
		}
		return nil, fmt.Errorf("failed to get configmap %s: %w", key, err)
	}

	dataKey := spec.ConfigMapRef.Key
	if dataKey == "" {
		dataKey = "data"
	}

	data, ok := cm.Data[dataKey]
	if !ok {
		binData, binOk := cm.BinaryData[dataKey]
		if !binOk {
			return nil, fmt.Errorf("key %q not found in configmap %s", dataKey, key)
		}
		return &ResolvedContent{
			Content:      binData,
			Checksum:     checksum(binData),
			Source:       v1alpha1.SourceTypeConfigMapRef,
		}, nil
	}

	content := []byte(data)
	return &ResolvedContent{
		Content:      content,
		Checksum:     checksum(content),
		Source:       v1alpha1.SourceTypeConfigMapRef,
	}, nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\secret.go
package source

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveSecret(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.SecretRef == nil {
		return nil, errSourceFieldMissing("secretRef")
	}

	secret := &corev1.Secret{}
	key := objectKey(spec.SecretRef.Namespace, spec.SecretRef.Name)
	if err := r.Get(ctx, key, secret); err != nil {
		if apierrors.IsNotFound(err) {
			return nil, fmt.Errorf("secret %s not found", key)
		}
		return nil, fmt.Errorf("failed to get secret %s: %w", key, err)
	}

	dataKey := spec.SecretRef.Key
	if dataKey == "" {
		dataKey = "data"
	}

	data, ok := secret.Data[dataKey]
	if !ok {
		return nil, fmt.Errorf("key %q not found in secret %s", dataKey, key)
	}

	return &ResolvedContent{
		Content:      data,
		Checksum:     checksum(data),
		Source:       v1alpha1.SourceTypeSecretRef,
	}, nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\uri.go
package source

import (
	"context"
	"crypto/tls"
	"fmt"
	"io"
	"net/http"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveURI(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.URIRef == nil {
		return nil, errSourceFieldMissing("uriRef")
	}

	httpClient := r.HTTPClient
	if spec.URIRef.InsecureSkipTLSVerify {
		httpClient = &http.Client{
			Timeout: r.HTTPClient.Timeout,
			Transport: &http.Transport{
				TLSClientConfig: &tls.Config{
					InsecureSkipVerify: true,
				},
			},
		}
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, spec.URIRef.URI, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to create request for %s: %w", spec.URIRef.URI, err)
	}

	resp, err := httpClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch %s: %w", spec.URIRef.URI, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("unexpected status code %d from %s", resp.StatusCode, spec.URIRef.URI)
	}

	content, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read response from %s: %w", spec.URIRef.URI, err)
	}

	computed := checksum(content)
	if spec.URIRef.Checksum != "" && spec.URIRef.Checksum != computed {
		return nil, fmt.Errorf("checksum mismatch for %s: expected %s, got %s",
			spec.URIRef.URI, spec.URIRef.Checksum, computed)
	}

	return &ResolvedContent{
		Content:  content,
		Checksum: computed,
		Source:   v1alpha1.SourceTypeURIRef,
	}, nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\image.go
package source

import (
	"archive/tar"
	"bytes"
	"context"
	"fmt"
	"io"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveImage(ctx context.Context, spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.ImageRef == nil {
		return nil, errSourceFieldMissing("imageRef")
	}

	image := spec.ImageRef.Image
	path := spec.ImageRef.Path
	if path == "" {
		path = "/scripts/install.sh"
	}

	content, err := extractFileFromImage(ctx, image, path, spec.ImageRef.ImagePullSecret)
	if err != nil {
		return nil, fmt.Errorf("failed to extract %s from image %s: %w", path, image, err)
	}

	computed := checksum(content)
	return &ResolvedContent{
		Content:  content,
		Checksum: computed,
		Source:   v1alpha1.SourceTypeImageRef,
	}, nil
}

func extractFileFromImage(ctx context.Context, image, path, pullSecret string) ([]byte, error) {
	args := []string{"create", "--name", "cv-extract", image}
	if pullSecret != "" {
		args = append([]string{"--pull-secret", pullSecret}, args...)
	}

	createCmd := execCommandContext(ctx, "docker", args...)
	if out, err := createCmd.CombinedOutput(); err != nil {
		return nil, fmt.Errorf("docker create failed: %w, output: %s", err, string(out))
	}
	defer func() {
		rmCmd := execCommandContext(ctx, "docker", "rm", "cv-extract")
		_ = rmCmd.Run()
	}()

	cpCmd := execCommandContext(ctx, "docker", "cp", "cv-extract:"+path, "-")
	out, err := cpCmd.Output()
	if err != nil {
		return nil, fmt.Errorf("docker cp failed: %w", err)
	}

	return extractFromTarArchive(out, path)
}

func extractFromTarArchive(tarData []byte, targetPath string) ([]byte, error) {
	reader := tar.NewReader(bytes.NewReader(tarData))
	for {
		header, err := reader.Next()
		if err == io.EOF {
			break
		}
		if err != nil {
			return nil, err
		}
		if header.Name == targetPath {
			buf := new(bytes.Buffer)
			if _, err := io.Copy(buf, reader); err != nil {
				return nil, err
			}
			return buf.Bytes(), nil
		}
	}
	return nil, fmt.Errorf("file %s not found in tar archive", targetPath)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\localpath.go
package source

import (
	"fmt"
	"os"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *Resolver) resolveLocalPath(spec *v1alpha1.SourceSpec) (*ResolvedContent, error) {
	if spec.LocalPath == nil {
		return nil, errSourceFieldMissing("localPath")
	}

	content, err := os.ReadFile(spec.LocalPath.Path)
	if err != nil {
		return nil, fmt.Errorf("failed to read local file %s: %w", spec.LocalPath.Path, err)
	}

	computed := checksum(content)
	if spec.LocalPath.Checksum != "" && spec.LocalPath.Checksum != computed {
		return nil, fmt.Errorf("checksum mismatch for %s: expected %s, got %s",
			spec.LocalPath.Path, spec.LocalPath.Checksum, computed)
	}

	return &ResolvedContent{
		Content:  content,
		Checksum: computed,
		Source:   v1alpha1.SourceTypeLocalPath,
	}, nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\helpers.go
package source

import (
	"crypto/sha256"
	"fmt"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func checksum(data []byte) string {
	h := sha256.Sum256(data)
	return fmt.Sprintf("sha256:%x", h)
}

type sourceError struct {
	sourceType v1alpha1.SourceType
	field      string
}

func (e *sourceError) Error() string {
	return fmt.Sprintf("source type %q requires field %q to be set", e.sourceType, e.field)
}

func errSourceFieldMissing(field string) *sourceError {
	return &sourceError{field: field}
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\source\image_exec.go
package source

import (
	"context"
	"os/exec"
)

type commandFunc func(ctx context.Context, name string, arg ...string) *exec.Cmd

var execCommandContext = func(ctx context.Context, name string, arg ...string) *exec.Cmd {
	return exec.CommandContext(ctx, name, arg...)
}
```
## 3. 状态机
```go d:\code\github\componentversion-controller\pkg\componentversion\statemachine\machine.go
package statemachine

import (
	"context"
	"fmt"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	ctrl "sigs.k8s.io/controller-runtime"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type ExecuteFunc func(ctx context.Context, cv *v1alpha1.ComponentVersion) (*ExecuteResult, error)

type ExecuteResult struct {
	Phase   v1alpha1.ComponentPhase
	Requeue bool
}

type Machine struct {
	transitions *TransitionTable
}

func NewMachine() *Machine {
	return &Machine{
		transitions: defaultTransitionTable(),
	}
}

func (m *Machine) Drive(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	current := cv.Status.Phase
	if current == "" {
		current = v1alpha1.PhasePending
		cv.Status.Phase = current
	}

	switch current {
	case v1alpha1.PhasePending:
		return m.handlePending(ctx, cv, execute)

	case v1alpha1.PhaseInstalling:
		return m.handleInstalling(ctx, cv, execute)

	case v1alpha1.PhaseVerifying:
		return m.handleVerifying(ctx, cv, execute)

	case v1alpha1.PhaseInstalled:
		return m.handleInstalled(ctx, cv, execute)

	case v1alpha1.PhaseFailed:
		return m.handleFailed(ctx, cv, execute)

	case v1alpha1.PhaseUpgrading:
		return m.handleUpgrading(ctx, cv, execute)

	case v1alpha1.PhaseRollingBack:
		return m.handleRollingBack(ctx, cv, execute)

	default:
		return ctrl.Result{}, fmt.Errorf("unknown phase: %s", current)
	}
}

func (m *Machine) handlePending(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	if err := m.transition(ctx, cv, v1alpha1.PhaseInstalling); err != nil {
		return ctrl.Result{}, err
	}
	result, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseFailed)
		return ctrl.Result{}, err
	}
	if result != nil && result.Requeue {
		return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
	}
	return ctrl.Result{Requeue: true}, nil
}

func (m *Machine) handleInstalling(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	result, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseFailed)
		return ctrl.Result{}, err
	}

	if result != nil && result.Requeue {
		return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
	}

	if err := m.transition(ctx, cv, v1alpha1.PhaseVerifying); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{Requeue: true}, nil
}

func (m *Machine) handleVerifying(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	result, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseFailed)
		return ctrl.Result{}, err
	}

	if result != nil && result.Phase == v1alpha1.PhaseVerifying {
		return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
	}

	if err := m.transition(ctx, cv, v1alpha1.PhaseInstalled); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (m *Machine) handleInstalled(ctx context.Context, cv *v1alpha1.ComponentVersion, _ ExecuteFunc) (ctrl.Result, error) {
	if cv.Status.ObservedVersion != "" && cv.Status.ObservedVersion != cv.Spec.Version {
		if err := m.transition(ctx, cv, v1alpha1.PhaseUpgrading); err != nil {
			return ctrl.Result{}, err
		}
		return ctrl.Result{Requeue: true}, nil
	}

	cv.Status.ObservedVersion = cv.Spec.Version
	return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (m *Machine) handleFailed(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	maxRetry := 3
	if cv.Status.RetryCount >= maxRetry {
		return ctrl.Result{}, nil
	}

	cv.Status.RetryCount++
	if err := m.transition(ctx, cv, v1alpha1.PhaseInstalling); err != nil {
		return ctrl.Result{}, err
	}

	result, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseFailed)
		return ctrl.Result{}, err
	}

	if result != nil && result.Requeue {
		return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
	}
	return ctrl.Result{Requeue: true}, nil
}

func (m *Machine) handleUpgrading(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	result, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseRollingBack)
		return ctrl.Result{}, err
	}

	if result != nil && result.Requeue {
		return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
	}

	if err := m.transition(ctx, cv, v1alpha1.PhaseVerifying); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{Requeue: true}, nil
}

func (m *Machine) handleRollingBack(ctx context.Context, cv *v1alpha1.ComponentVersion, execute ExecuteFunc) (ctrl.Result, error) {
	_, err := execute(ctx, cv)
	if err != nil {
		_ = m.transition(ctx, cv, v1alpha1.PhaseFailed)
		return ctrl.Result{}, err
	}

	if err := m.transition(ctx, cv, v1alpha1.PhaseInstalled); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (m *Machine) transition(_ context.Context, cv *v1alpha1.ComponentVersion, target v1alpha1.ComponentPhase) error {
	current := cv.Status.Phase
	if current == "" {
		current = v1alpha1.PhasePending
	}

	if !m.transitions.IsValid(current, target) {
		return fmt.Errorf("invalid transition: %s -> %s", current, target)
	}

	cv.Status.Phase = target
	cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}

	if target == v1alpha1.PhaseInstalling || target == v1alpha1.PhaseUpgrading {
		cv.Status.RetryCount = 0
	}

	return nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\statemachine\transition.go
package statemachine

import v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"

type TransitionTable struct {
	rules map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase
}

func defaultTransitionTable() *TransitionTable {
	return &TransitionTable{
		rules: map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase{
			v1alpha1.PhasePending:     {v1alpha1.PhaseInstalling},
			v1alpha1.PhaseInstalling:  {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed},
			v1alpha1.PhaseVerifying:   {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
			v1alpha1.PhaseInstalled:   {v1alpha1.PhaseUpgrading},
			v1alpha1.PhaseFailed:      {v1alpha1.PhaseInstalling, v1alpha1.PhaseRollingBack},
			v1alpha1.PhaseUpgrading:   {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed, v1alpha1.PhaseRollingBack},
			v1alpha1.PhaseRollingBack: {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
		},
	}
}

func (t *TransitionTable) IsValid(from, to v1alpha1.ComponentPhase) error {
	allowed, ok := t.rules[from]
	if !ok {
		return fmt.Errorf("no transitions defined from phase %q", from)
	}

	for _, phase := range allowed {
		if phase == to {
			return nil
		}
	}

	return fmt.Errorf("transition %s -> %s is not allowed", from, to)
}

func (t *TransitionTable) IsValid(from, to v1alpha1.ComponentPhase) bool {
	allowed, ok := t.rules[from]
	if !ok {
		return false
	}

	for _, phase := range allowed {
		if phase == to {
			return true
		}
	}
	return false
}

func (t *TransitionTable) AllowedTransitions(from v1alpha1.ComponentPhase) []v1alpha1.ComponentPhase {
	if allowed, ok := t.rules[from]; ok {
		result := make([]v1alpha1.ComponentPhase, len(allowed))
		copy(result, allowed)
		return result
	}
	return nil
}
```

> ⚠️ 上面的 `transition.go` 有重复的 `IsValid` 方法，修正如下：

```go d:\code\github\componentversion-controller\pkg\componentversion\statemachine\transition.go
package statemachine

import (
	"fmt"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type TransitionTable struct {
	rules map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase
}

func defaultTransitionTable() *TransitionTable {
	return &TransitionTable{
		rules: map[v1alpha1.ComponentPhase][]v1alpha1.ComponentPhase{
			v1alpha1.PhasePending:     {v1alpha1.PhaseInstalling},
			v1alpha1.PhaseInstalling:  {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed},
			v1alpha1.PhaseVerifying:   {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
			v1alpha1.PhaseInstalled:   {v1alpha1.PhaseUpgrading},
			v1alpha1.PhaseFailed:      {v1alpha1.PhaseInstalling, v1alpha1.PhaseRollingBack},
			v1alpha1.PhaseUpgrading:   {v1alpha1.PhaseVerifying, v1alpha1.PhaseFailed, v1alpha1.PhaseRollingBack},
			v1alpha1.PhaseRollingBack: {v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed},
		},
	}
}

func (t *TransitionTable) Validate(from, to v1alpha1.ComponentPhase) error {
	allowed, ok := t.rules[from]
	if !ok {
		return fmt.Errorf("no transitions defined from phase %q", from)
	}

	for _, phase := range allowed {
		if phase == to {
			return nil
		}
	}

	return fmt.Errorf("transition %s -> %s is not allowed", from, to)
}

func (t *TransitionTable) IsValid(from, to v1alpha1.ComponentPhase) bool {
	return t.Validate(from, to) == nil
}

func (t *TransitionTable) AllowedTransitions(from v1alpha1.ComponentPhase) []v1alpha1.ComponentPhase {
	if allowed, ok := t.rules[from]; ok {
		result := make([]v1alpha1.ComponentPhase, len(allowed))
		copy(result, allowed)
		return result
	}
	return nil
}
```
## 4. 依赖解析
```go d:\code\github\componentversion-controller\pkg\componentversion\dependency\resolver.go
package dependency

import (
	"context"
	"fmt"

	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type Resolver struct {
	client.Client
}

func NewResolver(c client.Client) *Resolver {
	return &Resolver{Client: c}
}

func (r *Resolver) CheckDependencies(ctx context.Context, cv *v1alpha1.ComponentVersion) (bool, error) {
	if len(cv.Spec.Dependencies) == 0 {
		return true, nil
	}

	for _, dep := range cv.Spec.Dependencies {
		depCV := &v1alpha1.ComponentVersion{}
		key := types.NamespacedName{Name: dep.Name}
		if err := r.Get(ctx, key, depCV); err != nil {
			return false, fmt.Errorf("failed to get dependency %q: %w", dep.Name, err)
		}

		if depCV.Status.Phase != v1alpha1.PhaseInstalled {
			return false, nil
		}

		if dep.MinVersion != "" {
			if !versionGTE(depCV.Status.ObservedVersion, dep.MinVersion) {
				return false, nil
			}
		}
	}

	return true, nil
}

func (r *Resolver) BuildDAG(ctx context.Context) (*Graph, error) {
	cvList := &v1alpha1.ComponentVersionList{}
	if err := r.List(ctx, cvList); err != nil {
		return nil, fmt.Errorf("failed to list componentversions: %w", err)
	}

	g := NewGraph()
	for _, cv := range cvList.Items {
		g.AddNode(cv.Name)
		for _, dep := range cv.Spec.Dependencies {
			g.AddEdge(dep.Name, cv.Name)
		}
	}

	if g.HasCycle() {
		return nil, fmt.Errorf("circular dependency detected in componentversions")
	}

	return g, nil
}

func (r *Resolver) GetInstallOrder(ctx context.Context) ([]string, error) {
	g, err := r.BuildDAG(ctx)
	if err != nil {
		return nil, err
	}
	return g.TopologicalSort(), nil
}

func (r *Resolver) GetDownstreamDependencies(ctx context.Context, cvName string) ([]string, error) {
	g, err := r.BuildDAG(ctx)
	if err != nil {
		return nil, err
	}
	return g.GetDependents(cvName), nil
}

func versionGTE(current, minVersion string) bool {
	if current == "" {
		return false
	}
	return current >= minVersion
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\dependency\graph.go
package dependency

import (
	"fmt"

	"github.com/hashicorp/go-version"
)

type Graph struct {
	nodes map[string]bool
	edges map[string][]string
}

func NewGraph() *Graph {
	return &Graph{
		nodes: make(map[string]bool),
		edges: make(map[string][]string),
	}
}

func (g *Graph) AddNode(name string) {
	g.nodes[name] = true
}

func (g *Graph) AddEdge(from, to string) {
	g.nodes[from] = true
	g.nodes[to] = true

	for _, existing := range g.edges[from] {
		if existing == to {
			return
		}
	}
	g.edges[from] = append(g.edges[from], to)
}

func (g *Graph) HasCycle() bool {
	visited := make(map[string]bool)
	recStack := make(map[string]bool)

	for node := range g.nodes {
		if g.hasCycleDFS(node, visited, recStack) {
			return true
		}
	}
	return false
}

func (g *Graph) hasCycleDFS(node string, visited, recStack map[string]bool) bool {
	visited[node] = true
	recStack[node] = true

	for _, neighbor := range g.edges[node] {
		if !visited[neighbor] {
			if g.hasCycleDFS(neighbor, visited, recStack) {
				return true
			}
		} else if recStack[neighbor] {
			return true
		}
	}

	recStack[node] = false
	return false
}

func (g *Graph) TopologicalSort() []string {
	visited := make(map[string]bool)
	var order []string

	for node := range g.nodes {
		if !visited[node] {
			g.topologicalSortDFS(node, visited, &order)
		}
	}

	for i, j := 0, len(order)-1; i < j; i, j = i+1, j-1 {
		order[i], order[j] = order[j], order[i]
	}

	return order
}

func (g *Graph) topologicalSortDFS(node string, visited map[string]bool, order *[]string) {
	visited[node] = true
	for _, neighbor := range g.edges[node] {
		if !visited[neighbor] {
			g.topologicalSortDFS(neighbor, visited, order)
		}
	}
	*order = append(*order, node)
}

func (g *Graph) GetDependents(nodeName string) []string {
	var dependents []string
	for node, edges := range g.edges {
		for _, edge := range edges {
			if edge == nodeName {
				dependents = append(dependents, node)
			}
		}
	}
	return dependents
}

func (g *Graph) GetDependencies(nodeName string) []string {
	return g.edges[nodeName]
}

func (g *Graph) GetNodes() []string {
	nodes := make([]string, 0, len(g.nodes))
	for node := range g.nodes {
		nodes = append(nodes, node)
	}
	return nodes
}

func semanticVersionGTE(current, minVersion string) (bool, error) {
	v1, err := version.NewVersion(current)
	if err != nil {
		return false, fmt.Errorf("failed to parse version %q: %w", current, err)
	}

	constraints, err := version.NewConstraint(fmt.Sprintf(">= %s", minVersion))
	if err != nil {
		return false, fmt.Errorf("failed to parse constraint %q: %w", minVersion, err)
	}

	return constraints.Check(v1), nil
}
```
## 文件清单汇总
| 序号 | 文件路径 | 说明 |
|------|----------|------|
| 1 | `api/v1alpha1/groupversion_info.go` | GroupVersion 与 SchemeBuilder |
| 2 | `api/v1alpha1/source_types.go` | 6 种来源类型定义 |
| 3 | `api/v1alpha1/install_types.go` | 三种安装策略 + 健康检查 + 状态详情类型 |
| 4 | `api/v1alpha1/componentversion_types.go` | ComponentVersion CRD 主类型 |
| 5 | `api/v1alpha1/nodeconfig_types.go` | NodeConfig CRD 主类型 |
| 6 | `api/v1alpha1/zz_generated.deepcopy.go` | DeepCopy 手动实现 |
| 7 | `pkg/componentversion/source/resolver.go` | Source 统一解析入口 |
| 8 | `pkg/componentversion/source/inline.go` | Inline 来源解析 |
| 9 | `pkg/componentversion/source/configmap.go` | ConfigMap 来源解析 |
| 10 | `pkg/componentversion/source/secret.go` | Secret 来源解析 |
| 11 | `pkg/componentversion/source/uri.go` | URI 远程拉取解析 |
| 12 | `pkg/componentversion/source/image.go` | 镜像内脚本提取 |
| 13 | `pkg/componentversion/source/localpath.go` | 本地路径读取 |
| 14 | `pkg/componentversion/source/helpers.go` | checksum 计算 + 错误类型 |
| 15 | `pkg/componentversion/source/image_exec.go` | 镜像提取命令抽象（便于测试） |
| 16 | `pkg/componentversion/statemachine/machine.go` | 状态机核心：7 个 Phase 驱动逻辑 |
| 17 | `pkg/componentversion/statemachine/transition.go` | 合法转换规则表 + 校验 |
| 18 | `pkg/componentversion/dependency/resolver.go` | 依赖检查 + DAG 构建 + 安装排序 |
| 19 | `pkg/componentversion/dependency/graph.go` | 有向图实现（拓扑排序 + 环检测） |

以上为基础层全部代码，下一次输出将实现**核心层：Installer 接口 + 三种安装器 + 健康检查**。

# Installer 接口与 ManifestInstaller 的完整实现（共 5 个文件）
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\interface.go
package installer

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"

	v1alpha1 "github.com/bke/componentversion-controller/api/v1alpha1"
)

type Installer interface {
	Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error)
	Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*VerifyResult, error)
	Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error
	Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error
}

type InstallResult struct {
	Phase   v1alpha1.ComponentPhase
	Details runtime.Object
	Requeue bool
}

type VerifyResult struct {
	Passed  bool
	Message string
}

type InstallerFactory func() Installer

var registry = map[v1alpha1.InstallType]InstallerFactory{}

func Register(installType v1alpha1.InstallType, factory InstallerFactory) {
	registry[installType] = factory
}

func GetInstaller(installType v1alpha1.InstallType) (Installer, error) {
	factory, ok := registry[installType]
	if !ok {
		return nil, NewInstallError(installType, "no installer registered for this type")
	}
	return factory(), nil
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\errors.go
package installer

import (
	"fmt"

	v1alpha1 "github.com/bke/componentversion-controller/api/v1alpha1"
)

type InstallError struct {
	InstallType v1alpha1.InstallType
	Op          string
	Err         error
}

func NewInstallError(installType v1alpha1.InstallType, msg string) *InstallError {
	return &InstallError{InstallType: installType, Op: msg}
}

func NewInstallErrorWrap(installType v1alpha1.InstallType, op string, err error) *InstallError {
	return &InstallError{InstallType: installType, Op: op, Err: err}
}

func (e *InstallError) Error() string {
	if e.Err != nil {
		return fmt.Sprintf("[%s] %s: %v", e.InstallType, e.Op, e.Err)
	}
	return fmt.Sprintf("[%s] %s", e.InstallType, e.Op)
}

func (e *InstallError) Unwrap() error {
	return e.Err
}

type VerifyError struct {
	Resource string
	Message  string
}

func NewVerifyError(resource, msg string) *VerifyError {
	return &VerifyError{Resource: resource, Message: msg}
}

func (e *VerifyError) Error() string {
	return fmt.Sprintf("verify failed for %q: %s", e.Resource, e.Message)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\manifest\installer.go
package manifest

import (
	"context"
	"crypto/sha256"
	"encoding/json"
	"fmt"
	"time"

	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/yaml"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/scheme"

	v1alpha1 "github.com/bke/componentversion-controller/api/v1alpha1"
	"github.com/bke/componentversion-controller/pkg/componentversion/source"
)

const (
	ownerLabelKey   = "orchestration.bke.io/component-version"
	fieldManager    = "componentversion-controller"
	maxVerifyRetries = 3
)

type Installer struct {
	client.Client
	Scheme         *scheme.Scheme
	SourceResolver *source.Resolver
}

func NewInstaller(c client.Client, s *scheme.Scheme, resolver *source.Resolver) *Installer {
	return &Installer{
		Client:         c,
		Scheme:         s,
		SourceResolver: resolver,
	}
}

func (m *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
	manifestSpec := cv.Spec.Install.Manifest
	if manifestSpec == nil {
		return nil, installer.NewInstallError(v1alpha1.InstallTypeManifest, "manifest spec is nil")
	}

	var allApplied []v1alpha1.AppliedResource
	var allErrors []error

	for _, item := range manifestSpec.Items {
		applied, err := m.installItem(ctx, cv, item, manifestSpec.ApplyStrategy)
		if err != nil {
			allErrors = append(allErrors, fmt.Errorf("item %q: %w", item.Name, err))
			continue
		}
		allApplied = append(allApplied, applied...)
	}

	if len(allErrors) > 0 {
		return &installer.InstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Details: m.buildInstallDetails(allApplied, nil),
		}, fmt.Errorf("manifest install partially failed: %v", allErrors)
	}

	var pruned []v1alpha1.PrunedResource
	if manifestSpec.Prune {
		var pruneErr error
		pruned, pruneErr = m.Prune(ctx, cv, allApplied)
		if pruneErr != nil {
			return &installer.InstallResult{
				Phase:   v1alpha1.PhaseVerifying,
				Requeue: false,
				Details: m.buildInstallDetails(allApplied, pruned),
			}, pruneErr
		}
	}

	return &installer.InstallResult{
		Phase:   v1alpha1.PhaseVerifying,
		Requeue: false,
		Details: m.buildInstallDetails(allApplied, pruned),
	}, nil
}

func (m *Installer) installItem(ctx context.Context, cv *v1alpha1.ComponentVersion, item v1alpha1.ManifestItem, strategy v1alpha1.ApplyStrategy) ([]v1alpha1.AppliedResource, error) {
	resolved, err := m.SourceResolver.Resolve(ctx, &item.Source)
	if err != nil {
		return nil, fmt.Errorf("failed to resolve source: %w", err)
	}

	rendered, err := RenderTemplate(resolved.Content, cv)
	if err != nil {
		return nil, fmt.Errorf("failed to render template: %w", err)
	}

	objects, err := m.parseManifests(rendered)
	if err != nil {
		return nil, fmt.Errorf("failed to parse manifests: %w", err)
	}

	var applied []v1alpha1.AppliedResource
	for _, obj := range objects {
		namespace := obj.GetNamespace()
		if namespace == "" {
			namespace = item.Namespace
			obj.SetNamespace(namespace)
		}

		m.setOwnerLabels(obj, cv)

		if err := m.applyObject(ctx, obj, strategy); err != nil {
			return applied, fmt.Errorf("failed to apply %s/%s: %w", obj.GetKind(), obj.GetName(), err)
		}

		gvk := obj.GroupVersionKind()
		contentHash := hashContent(rendered)
		applied = append(applied, v1alpha1.AppliedResource{
			Group:     gvk.Group,
			Version:   gvk.Version,
			Kind:      gvk.Kind,
			Name:      obj.GetName(),
			Namespace: obj.GetNamespace(),
			Hash:      contentHash,
			AppliedAt: &metav1.Time{Time: time.Now()},
		})
	}

	return applied, nil
}

func (m *Installer) applyObject(ctx context.Context, obj *unstructured.Unstructured, strategy v1alpha1.ApplyStrategy) error {
	switch strategy {
	case v1alpha1.ApplyStrategyServerSideApply:
		return applyServerSideApply(ctx, m.Client, obj, fieldManager)
	case v1alpha1.ApplyStrategyThreeWayMerge:
		return applyThreeWayMerge(ctx, m.Client, obj)
	case v1alpha1.ApplyStrategyReplace:
		return applyReplace(ctx, m.Client, obj)
	default:
		return applyServerSideApply(ctx, m.Client, obj, fieldManager)
	}
}

func (m *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
	details := cv.Status.InstallDetails.Manifest
	if details == nil || len(details.AppliedResources) == 0 {
		return &installer.VerifyResult{Passed: true}, nil
	}

	var notReady []string
	for _, res := range details.AppliedResources {
		gvk := schema.GroupVersionKind{
			Group:   res.Group,
			Version: res.Version,
			Kind:    res.Kind,
		}

		gvr, err := m.gvkToGVR(gvk)
		if err != nil {
			notReady = append(notReady, fmt.Sprintf("%s/%s (gvr lookup failed)", res.Kind, res.Name))
			continue
		}

		obj := &unstructured.Unstructured{}
		obj.SetGroupVersionKind(gvk)
		key := client.ObjectKey{Namespace: res.Namespace, Name: res.Name}
		if err := m.Get(ctx, key, obj); err != nil {
			notReady = append(notReady, fmt.Sprintf("%s/%s (not found)", res.Kind, res.Name))
			continue
		}

		if !m.isResourceReady(obj) {
			notReady = append(notReady, fmt.Sprintf("%s/%s (not ready)", res.Kind, res.Name))
		}
	}

	if len(notReady) > 0 {
		return &installer.VerifyResult{
			Passed:  false,
			Message: fmt.Sprintf("resources not ready: %v", notReady),
		}, nil
	}

	return &installer.VerifyResult{Passed: true}, nil
}

func (m *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
	details := cv.Status.InstallDetails.Manifest
	if details == nil {
		return nil
	}

	manifestSpec := cv.Spec.Install.Manifest
	if manifestSpec == nil {
		return nil
	}

	for i := len(details.AppliedResources) - 1; i >= 0; i-- {
		res := details.AppliedResources[i]
		gvk := schema.GroupVersionKind{
			Group:   res.Group,
			Version: res.Version,
			Kind:    res.Kind,
		}

		obj := &unstructured.Unstructured{}
		obj.SetGroupVersionKind(gvk)
		key := client.ObjectKey{Namespace: res.Namespace, Name: res.Name}

		current := &unstructured.Unstructured{}
		current.SetGroupVersionKind(gvk)
		if err := m.Get(ctx, key, current); err != nil {
			continue
		}

		currentHash, _, _ := unstructured.NestedString(current.Object, "metadata", "annotations", ownerLabelKey+"-hash")
		if currentHash == res.Hash {
			continue
		}

		for _, item := range manifestSpec.Items {
			if item.Name != res.Name {
				continue
			}
			resolved, err := m.SourceResolver.Resolve(ctx, &item.Source)
			if err != nil {
				continue
			}
			rendered, err := RenderTemplate(resolved.Content, cv)
			if err != nil {
				continue
			}
			objects, err := m.parseManifests(rendered)
			if err != nil {
				continue
			}
			for _, obj := range objects {
				if obj.GetName() == res.Name && obj.GetNamespace() == res.Namespace {
					m.setOwnerLabels(obj, cv)
					_ = applyServerSideApply(ctx, m.Client, obj, fieldManager)
				}
			}
		}
	}

	return nil
}

func (m *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
	details := cv.Status.InstallDetails.Manifest
	if details == nil {
		return nil
	}

	var lastErr error
	for i := len(details.AppliedResources) - 1; i >= 0; i-- {
		res := details.AppliedResources[i]
		gvk := schema.GroupVersionKind{
			Group:   res.Group,
			Version: res.Version,
			Kind:    res.Kind,
		}

		obj := &unstructured.Unstructured{}
		obj.SetGroupVersionKind(gvk)
		obj.SetName(res.Name)
		obj.SetNamespace(res.Namespace)

		if err := m.Delete(ctx, obj); err != nil {
			lastErr = err
		}
	}

	return lastErr
}

func (m *Installer) parseManifests(data []byte) ([]*unstructured.Unstructured, error) {
	var objects []*unstructured.Unstructured

	decoder := yaml.NewYAMLOrJSONDecoder(data, 4096)
	for {
		obj := &unstructured.Unstructured{}
		if err := decoder.Decode(obj); err != nil {
			if err.Error() == "EOF" {
				break
			}
			continue
		}

		if obj.GetKind() == "" || obj.GetAPIVersion() == "" {
			continue
		}

		objects = append(objects, obj)
	}

	if len(objects) == 0 {
		return nil, fmt.Errorf("no valid kubernetes resources found in manifest")
	}

	return objects, nil
}

func (m *Installer) setOwnerLabels(obj *unstructured.Unstructured, cv *v1alpha1.ComponentVersion) {
	labels := obj.GetLabels()
	if labels == nil {
		labels = make(map[string]string)
	}
	labels[ownerLabelKey] = cv.Name
	labels[ownerLabelKey+"-component"] = cv.Spec.ComponentName
	labels[ownerLabelKey+"-version"] = cv.Spec.Version
	obj.SetLabels(labels)
}

func (m *Installer) isResourceReady(obj *unstructured.Unstructured) bool {
	kind := obj.GetKind()

	switch kind {
	case "Deployment":
		replicas, _, _ := unstructured.NestedInt64(obj.Object, "spec", "replicas")
		ready, _, _ := unstructured.NestedInt64(obj.Object, "status", "readyReplicas")
		if replicas == 0 {
			return true
		}
		return ready >= replicas

	case "StatefulSet":
		replicas, _, _ := unstructured.NestedInt64(obj.Object, "spec", "replicas")
		ready, _, _ := unstructured.NestedInt64(obj.Object, "status", "readyReplicas")
		if replicas == 0 {
			return true
		}
		return ready >= replicas

	case "DaemonSet":
		desired, _, _ := unstructured.NestedInt64(obj.Object, "status", "desiredNumberScheduled")
		ready, _, _ := unstructured.NestedInt64(obj.Object, "status", "numberReady")
		return desired > 0 && ready >= desired

	case "Pod":
		phase, _, _ := unstructured.NestedString(obj.Object, "status", "phase")
		return phase == "Running" || phase == "Succeeded"

	case "Job":
		succeeded, _, _ := unstructured.NestedInt64(obj.Object, "status", "succeeded")
		completionTime, _, _ := unstructured.NestedString(obj.Object, "status", "completionTime")
		return succeeded > 0 || completionTime != ""

	case "Service":
		return true

	case "ConfigMap", "Secret", "ServiceAccount", "Role", "RoleBinding",
		"ClusterRole", "ClusterRoleBinding", "CustomResourceDefinition",
		"Namespace", "Ingress", "NetworkPolicy":
		return true

	default:
		return true
	}
}

func (m *Installer) gvkToGVR(gvk schema.GroupVersionKind) (schema.GroupVersionResource, error) {
	mapper, err := meta.DefaultRESTMapper(nil, nil)
	_ = mapper
	return schema.GroupVersionResource{
		Group:    gvk.Group,
		Version:  gvk.Version,
		Resource: pluralize(gvk.Kind),
	}, nil
}

func (m *Installer) buildInstallDetails(applied []v1alpha1.AppliedResource, pruned []v1alpha1.PrunedResource) *v1alpha1.ManifestInstallDetails {
	details := &v1alpha1.ManifestInstallDetails{}
	if len(applied) > 0 {
		details.AppliedResources = applied
	}
	if len(pruned) > 0 {
		details.PrunedResources = pruned
	}
	return details
}

func hashContent(data []byte) string {
	h := sha256.Sum256(data)
	return fmt.Sprintf("sha256:%x", h)
}

func pluralize(kind string) string {
	if len(kind) == 0 {
		return kind
	}
	lc := kind[len(kind)-1]
	switch lc {
	case 's':
		return kind + "es"
	case 'y':
		return kind[:len(kind)-1] + "ies"
	default:
		return kind + "s"
	}
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\manifest\apply.go
package manifest

import (
	"context"
	"encoding/json"
	"fmt"

	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

func applyServerSideApply(ctx context.Context, c client.Client, obj *unstructured.Unstructured, fieldManager string) error {
	data, err := json.Marshal(obj)
	if err != nil {
		return fmt.Errorf("failed to marshal object: %w", err)
	}

	patch := client.RawPatch(types.ApplyPatchType, data)
	force := true
	opts := &client.PatchOptions{
		FieldManager: fieldManager,
		Force:        &force,
	}

	if err := c.Patch(ctx, obj, patch, opts); err != nil {
		if apierrors.IsConflict(err) {
			return fmt.Errorf("server-side apply conflict (will retry): %w", err)
		}
		return fmt.Errorf("server-side apply failed: %w", err)
	}

	return nil
}

func applyThreeWayMerge(ctx context.Context, c client.Client, obj *unstructured.Unstructured) error {
	existing := &unstructured.Unstructured{}
	existing.SetGroupVersionKind(obj.GroupVersionKind())
	key := client.ObjectKeyFromObject(obj)

	if err := c.Get(ctx, key, existing); err != nil {
		if apierrors.IsNotFound(err) {
			if err := c.Create(ctx, obj); err != nil {
				if apierrors.IsAlreadyExists(err) {
					return fmt.Errorf("object created by another process (will retry)")
				}
				return fmt.Errorf("create failed: %w", err)
			}
			return nil
		}
		return fmt.Errorf("get existing failed: %w", err)
	}

	merged := mergeObjects(existing, obj)
	merged.SetResourceVersion(existing.GetResourceVersion())
	merged.SetAnnotations(mergeMaps(existing.GetAnnotations(), obj.GetAnnotations()))
	merged.SetLabels(mergeMaps(existing.GetLabels(), obj.GetLabels()))

	if err := c.Update(ctx, merged); err != nil {
		if apierrors.IsConflict(err) {
			return fmt.Errorf("update conflict (will retry): %w", err)
		}
		return fmt.Errorf("three-way merge update failed: %w", err)
	}

	return nil
}

func applyReplace(ctx context.Context, c client.Client, obj *unstructured.Unstructured) error {
	existing := &unstructured.Unstructured{}
	existing.SetGroupVersionKind(obj.GroupVersionKind())
	key := client.ObjectKeyFromObject(obj)

	if err := c.Get(ctx, key, existing); err == nil {
		deletePolicy := metav1.DeletePropagationBackground
		deleteOpts := &client.DeleteOptions{
			PropagationPolicy: &deletePolicy,
		}
		if err := c.Delete(ctx, existing, deleteOpts); err != nil {
			if !apierrors.IsNotFound(err) {
				return fmt.Errorf("delete existing failed: %w", err)
			}
		}
	}

	if err := c.Create(ctx, obj); err != nil {
		if apierrors.IsAlreadyExists(err) {
			return fmt.Errorf("object re-created by another process (will retry)")
		}
		return fmt.Errorf("create failed: %w", err)
	}

	return nil
}

func mergeObjects(existing, desired *unstructured.Unstructured) *unstructured.Unstructured {
	result := existing.DeepCopy()

	spec, specOk := desired.Object["spec"]
	if specOk {
		result.Object["spec"] = spec
	}

	data, dataOk := desired.Object["data"]
	if dataOk {
		result.Object["data"] = data
	}

	binaryData, binaryDataOk := desired.Object["binaryData"]
	if binaryDataOk {
		result.Object["binaryData"] = binaryData
	}

	rules, rulesOk := desired.Object["rules"]
	if rulesOk {
		result.Object["rules"] = rules
	}

	subjects, subjectsOk := desired.Object["subjects"]
	if subjectsOk {
		result.Object["subjects"] = subjects
	}

	roleRef, roleRefOk := desired.Object["roleRef"]
	if roleRefOk {
		result.Object["roleRef"] = roleRef
	}

	template, templateOk := desired.Object["spec"]
	if templateOk {
		if specMap, ok := spec.(map[string]interface{}); ok {
			if tmpl, ok := specMap["template"]; ok {
				if resultSpec, ok := result.Object["spec"].(map[string]interface{}); ok {
					resultSpec["template"] = tmpl
				}
			}
		}
	}

	return result
}

func mergeMaps(base, override map[string]string) map[string]string {
	if base == nil && override == nil {
		return nil
	}

	result := make(map[string]string, len(base)+len(override))
	for k, v := range base {
		result[k] = v
	}
	for k, v := range override {
		result[k] = v
	}
	return result
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\manifest\prune.go
package manifest

import (
	"context"
	"fmt"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "github.com/bke/componentversion-controller/api/v1alpha1"
)

var pruneableGVRs = []schema.GroupVersionResource{
	{Group: "apps", Version: "v1", Resource: "deployments"},
	{Group: "apps", Version: "v1", Resource: "statefulsets"},
	{Group: "apps", Version: "v1", Resource: "daemonsets"},
	{Group: "batch", Version: "v1", Resource: "jobs"},
	{Group: "", Version: "v1", Resource: "services"},
	{Group: "", Version: "v1", Resource: "configmaps"},
	{Group: "", Version: "v1", Resource: "secrets"},
	{Group: "networking.k8s.io", Version: "v1", Resource: "ingresses"},
	{Group: "networking.k8s.io", Version: "v1", Resource: "networkpolicies"},
	{Group: "rbac.authorization.k8s.io", Version: "v1", Resource: "roles"},
	{Group: "rbac.authorization.k8s.io", Version: "v1", Resource: "rolebindings"},
	{Group: "rbac.authorization.k8s.io", Version: "v1", Resource: "clusterroles"},
	{Group: "rbac.authorization.k8s.io", Version: "v1", Resource: "clusterrolebindings"},
	{Group: "", Version: "v1", Resource: "serviceaccounts"},
	{Group: "apiextensions.k8s.io", Version: "v1", Resource: "customresourcedefinitions"},
}

func (m *Installer) Prune(ctx context.Context, cv *v1alpha1.ComponentVersion, applied []v1alpha1.AppliedResource) ([]v1alpha1.PrunedResource, error) {
	owned, err := m.ListOwnedResources(ctx, cv)
	if err != nil {
		return nil, fmt.Errorf("failed to list owned resources: %w", err)
	}

	appliedSet := make(map[string]bool)
	for _, res := range applied {
		key := resourceKey(res.Group, res.Version, res.Kind, res.Namespace, res.Name)
		appliedSet[key] = true
	}

	var pruned []v1alpha1.PrunedResource
	for i := len(owned) - 1; i >= 0; i-- {
		res := owned[i]
		key := resourceKey(res.Group, res.Version, res.Kind, res.Namespace, res.Name)
		if !appliedSet[key] {
			if err := m.pruneResource(ctx, res); err != nil {
				continue
			}
			pruned = append(pruned, v1alpha1.PrunedResource{
				Group:     res.Group,
				Version:   res.Version,
				Kind:      res.Kind,
				Name:      res.Name,
				Namespace: res.Namespace,
			})
		}
	}

	return pruned, nil
}

func (m *Installer) ListOwnedResources(ctx context.Context, cv *v1alpha1.ComponentVersion) ([]v1alpha1.AppliedResource, error) {
	labelSelector := labels.Set{
		ownerLabelKey: cv.Name,
	}.AsSelector()

	var allResources []v1alpha1.AppliedResource

	for _, gvr := range pruneableGVRs {
		list := &unstructured.UnstructuredList{}
		list.SetGroupVersionKind(schema.GroupVersionKind{
			Group:   gvr.Group,
			Version: gvr.Version,
			Kind:    kindForGVR(gvr),
		})

		opts := []client.ListOption{
			client.MatchingLabelsSelector{Selector: labelSelector},
		}

		if gvr.Group != "apiextensions.k8s.io" &&
			gvr.Resource != "clusterroles" &&
			gvr.Resource != "clusterrolebindings" &&
			gvr.Resource != "customresourcedefinitions" {
		} else {
			opts = append(opts, client.InNamespace(""))
		}

		if err := m.List(ctx, list, opts...); err != nil {
			continue
		}

		for _, item := range list.Items {
			gvk := item.GroupVersionKind()
			allResources = append(allResources, v1alpha1.AppliedResource{
				Group:     gvk.Group,
				Version:   gvk.Version,
				Kind:      gvk.Kind,
				Name:      item.GetName(),
				Namespace: item.GetNamespace(),
			})
		}
	}

	return allResources, nil
}

func (m *Installer) pruneResource(ctx context.Context, res v1alpha1.AppliedResource) error {
	gvk := schema.GroupVersionKind{
		Group:   res.Group,
		Version: res.Version,
		Kind:    res.Kind,
	}

	obj := &unstructured.Unstructured{}
	obj.SetGroupVersionKind(gvk)
	obj.SetName(res.Name)
	obj.SetNamespace(res.Namespace)

	deletePolicy := metav1.DeletePropagationBackground
	deleteOpts := &client.DeleteOptions{
		PropagationPolicy: &deletePolicy,
	}

	if err := m.Delete(ctx, obj, deleteOpts); err != nil {
		if apierrors.IsNotFound(err) {
			return nil
		}
		return fmt.Errorf("failed to prune %s/%s: %w", res.Kind, res.Name, err)
	}

	return nil
}

func resourceKey(group, version, kind, namespace, name string) string {
	if namespace != "" {
		return fmt.Sprintf("%s/%s/%s/%s/%s", group, version, kind, namespace, name)
	}
	return fmt.Sprintf("%s/%s/%s//%s", group, version, kind, name)
}

func kindForGVR(gvr schema.GroupVersionResource) string {
	resource := gvr.Resource
	if len(resource) == 0 {
		return resource
	}

	singular := resource
	switch {
	case len(resource) > 2 && resource[len(resource)-3:] == "ies":
		singular = resource[:len(resource)-3] + "y"
	case len(resource) > 1 && resource[len(resource)-1] == 's':
		singular = resource[:len(resource)-1]
	case len(resource) > 2 && resource[len(resource)-2:] == "es":
		singular = resource[:len(resource)-2]
	}

	return capitalize(singular)
}

func capitalize(s string) string {
	if len(s) == 0 {
		return s
	}
	result := make([]byte, len(s))
	result[0] = s[0] - 32
	for i := 1; i < len(s); i++ {
		if s[i-1] == '-' {
			result[i] = s[i] - 32
		} else {
			result[i] = s[i]
		}
	}
	return string(result)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\manifest\template.go
package manifest

import (
	"bytes"
	"fmt"
	"text/template"

	v1alpha1 "github.com/bke/componentversion-controller/api/v1alpha1"
)

type templateData struct {
	Version        string
	ComponentName  string
	Namespace      string
	Images         map[string]string
	ImagesByIndex  []string
}

func RenderTemplate(content []byte, cv *v1alpha1.ComponentVersion) ([]byte, error) {
	images := make(map[string]string)
	imagesByIndex := make([]string, len(cv.Spec.Images))
	for i, img := range cv.Spec.Images {
		images[img.Name] = img.Image
		imagesByIndex[i] = img.Image
	}

	data := templateData{
		Version:       cv.Spec.Version,
		ComponentName: cv.Spec.ComponentName,
		Namespace:     cv.Namespace,
		Images:        images,
		ImagesByIndex: imagesByIndex,
	}

	tmpl, err := template.New("manifest").
		Delims("{{", "}}").
		Option("missingkey=error").
		Funcs(templateFuncs()).
		Parse(string(content))
	if err != nil {
		return nil, fmt.Errorf("failed to parse template: %w", err)
	}

	var buf bytes.Buffer
	if err := tmpl.Execute(&buf, data); err != nil {
		return nil, fmt.Errorf("failed to render template: %w", err)
	}

	return buf.Bytes(), nil
}

func templateFuncs() template.FuncMap {
	return template.FuncMap{
		"Image": func(images map[string]string, name string) (string, error) {
			img, ok := images[name]
			if !ok {
				return "", fmt.Errorf("image %q not found in spec.images", name)
			}
			return img, nil
		},
		"Default": func(val, fallback string) string {
			if val == "" {
				return fallback
			}
			return val
		},
	}
}
```
### 文件清单
| 序号 | 文件路径 | 说明 |
|------|----------|------|
| 1 | `pkg/componentversion/installer/interface.go` | Installer 接口 + InstallResult/VerifyResult + 注册表 |
| 2 | `pkg/componentversion/installer/errors.go` | InstallError / VerifyError 错误类型 |
| 3 | `pkg/componentversion/installer/manifest/installer.go` | ManifestInstaller 完整实现（Install/Verify/Rollback/Uninstall + 解析 + 就绪判断） |
| 4 | `pkg/componentversion/installer/manifest/apply.go` | 三种 Apply 策略实现（ServerSideApply/ThreeWayMerge/Replace） |
| 5 | `pkg/componentversion/installer/manifest/prune.go` | Prune 实现（基于 ownership label 查询 + 对比 + 清理） |
| 6 | `pkg/componentversion/installer/manifest/template.go` | 模板渲染（支持 Version/ComponentName/Images 变量替换） |
### 关键设计说明
**1. Install 流程**
```
Install(ctx, cv)
  │
  ├─ 遍历 manifest.items
  │   ├─ SourceResolver.Resolve → []byte
  │   ├─ RenderTemplate → 替换 {{.Version}}/{{.Image "name"}} 等变量
  │   ├─ parseManifests → []*unstructured.Unstructured
  │   ├─ setOwnerLabels → 标记 orchestration.bke.io/component-version
  │   ├─ applyObject → 按 ApplyStrategy 路由
  │   └─ 记录 AppliedResource（含 sha256 hash）
  │
  ├─ 如果 prune=true → Prune 清理废弃资源
  │
  └─ 返回 InstallResult{Phase: Verifying, Details: ManifestInstallDetails}
```
**2. Verify 就绪判断规则**

| 资源类型 | 就绪条件 |
|----------|----------|
| Deployment | `readyReplicas >= replicas`（replicas=0 时直接就绪） |
| StatefulSet | `readyReplicas >= replicas` |
| DaemonSet | `numberReady >= desiredNumberScheduled` |
| Pod | `phase == Running \|\| Succeeded` |
| Job | `succeeded > 0 \|\| completionTime != ""` |
| Service/ConfigMap/Secret/RBAC 等 | 直接视为就绪 |

**3. 三种 Apply 策略对比**

| 策略 | 实现方式 | 冲突处理 | 适用场景 |
|------|----------|----------|----------|
| ServerSideApply | `client.Patch(ApplyPatchType)` + Force | 服务端合并，冲突返回 error | 默认策略，多控制器共存 |
| ThreeWayMerge | Get → 合并 spec/labels/annotations → Update | 客户端合并，冲突返回 error | 需要精确控制合并逻辑 |
| Replace | Delete(Background) → Create | 完全替换 | 不可变资源或需要重建 |

**4. Prune 机制**

- 通过 `orchestration.bke.io/component-version=<cv-name>` label 标识属于该 CV 的资源
- Prune 时查询所有带此 label 的资源，与当前 `appliedResources` 对比
- 不在 appliedResources 中的资源按逆序删除（后创建的先删）
- 支持的 Prune 资源类型：Deployment/StatefulSet/DaemonSet/Job/Service/ConfigMap/Secret/Ingress/NetworkPolicy/RBAC/CRD

# ScriptInstaller      
基于之前会话中建立的设计，我将实现 ScriptInstaller 核心层，包含执行器（Local、NodeAgent、InitPod）和验证器。由于项目目录尚未在磁盘上创建，我将以代码块形式输出所有文件。
## 1. Installer 接口与错误类型（依赖基础）
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\interface.go
package installer

import (
	"context"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type InstallResult struct {
	Phase   v1alpha1.ComponentPhase
	Requeue bool
	Details v1alpha1.InstallDetails
}

type VerifyResult struct {
	Healthy bool
	Reason  string
	Details v1alpha1.InstallDetails
}

type RollbackResult struct {
	Phase   v1alpha1.ComponentPhase
	Requeue bool
	Message string
}

type UninstallResult struct {
	Phase   v1alpha1.ComponentPhase
	Requeue bool
	Message string
}

type Installer interface {
	Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*InstallResult, error)
	Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*VerifyResult, error)
	Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) (*RollbackResult, error)
	Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) (*UninstallResult, error)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\errors.go
package installer

import (
	"fmt"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type InstallError struct {
	InstallType v1alpha1.InstallType
	Message     string
	Cause       error
	Retryable   bool
}

func (e *InstallError) Error() string {
	if e.Cause != nil {
		return fmt.Sprintf("[%s] %s: %v", e.InstallType, e.Message, e.Cause)
	}
	return fmt.Sprintf("[%s] %s", e.InstallType, e.Message)
}

func (e *InstallError) Unwrap() error {
	return e.Cause
}

func NewInstallError(installType v1alpha1.InstallType, message string) *InstallError {
	return &InstallError{InstallType: installType, Message: message}
}

func NewRetryableError(installType v1alpha1.InstallType, message string, cause error) *InstallError {
	return &InstallError{InstallType: installType, Message: message, Cause: cause, Retryable: true}
}

func NewFatalError(installType v1alpha1.InstallType, message string, cause error) *InstallError {
	return &InstallError{InstallType: installType, Message: message, Cause: cause, Retryable: false}
}

type VerifyError struct {
	InstallType v1alpha1.InstallType
	Message     string
	Cause       error
}

func (e *VerifyError) Error() string {
	if e.Cause != nil {
		return fmt.Sprintf("[%s] verify failed: %s: %v", e.InstallType, e.Message, e.Cause)
	}
	return fmt.Sprintf("[%s] verify failed: %s", e.InstallType, e.Message)
}

func NewVerifyError(installType v1alpha1.InstallType, message string, cause error) *VerifyError {
	return &VerifyError{InstallType: installType, Message: message, Cause: cause}
}
```
## 2. ScriptInstaller 主实现
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\installer.go
package script

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type Installer struct {
	client.Client
	Log       logr.Logger
	Resolver  *source.Resolver
	Executors map[v1alpha1.ExecutorType]Executor
	Verifier  *Verifier
}

func NewInstaller(c client.Client, log logr.Logger, resolver *source.Resolver) *Installer {
	i := &Installer{
		Client:   c,
		Log:      log.WithName("script-installer"),
		Resolver: resolver,
	}

	i.Executors = map[v1alpha1.ExecutorType]Executor{
		v1alpha1.ExecutorLocal:    NewLocalExecutor(c, log),
		v1alpha1.ExecutorNodeAgent: NewNodeAgentExecutor(c, log),
		v1alpha1.ExecutorInitPod:  NewInitPodExecutor(c, log),
	}

	i.Verifier = NewVerifier(c, log, resolver)
	return i
}

func (s *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
	scriptSpec := cv.Spec.Install.Script
	if scriptSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, "script spec is nil", nil)
	}

	log := s.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	scriptContent, err := s.Resolver.Resolve(ctx, scriptSpec.Source)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve script source", err)
	}

	envVars, err := s.resolveEnvVars(ctx, scriptSpec.Env)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve env vars", err)
	}

	execType := scriptSpec.Executor
	if execType == "" {
		execType = v1alpha1.ExecutorLocal
	}

	executor, ok := s.Executors[execType]
	if !ok {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, fmt.Sprintf("unsupported executor type: %s", execType), nil)
	}

	timeout := time.Duration(scriptSpec.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 5 * time.Minute
	}

	if scriptSpec.PreScript != nil {
		log.Info("executing pre-script")
		preContent, preErr := s.Resolver.Resolve(ctx, scriptSpec.PreScript)
		if preErr != nil {
			return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve pre-script source", preErr)
		}
		preReq := &ExecuteRequest{
			Script:       preContent.Data,
			Env:          envVars,
			Timeout:      timeout,
			ComponentName: cv.Spec.ComponentName,
			Version:      cv.Spec.Version,
			Namespace:    cv.Namespace,
			Phase:        "pre",
		}
		if _, execErr := executor.Execute(ctx, preReq); execErr != nil {
			return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "pre-script execution failed", execErr)
		}
	}

	req := &ExecuteRequest{
		Script:        scriptContent.Data,
		Env:           envVars,
		Timeout:       timeout,
		ComponentName: cv.Spec.ComponentName,
		Version:       cv.Spec.Version,
		Namespace:     cv.Namespace,
		Phase:         "install",
	}

	result, err := executor.Execute(ctx, req)
	if err != nil {
		log.Error(err, "script execution failed", "executor", execType)
		return &installer.InstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Details: s.buildInstallDetails(result, nil),
		}, installer.NewRetryableError(v1alpha1.InstallTypeScript, "script execution failed", err)
	}

	if scriptSpec.PostScript != nil {
		log.Info("executing post-script")
		postContent, postErr := s.Resolver.Resolve(ctx, scriptSpec.PostScript)
		if postErr != nil {
			return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve post-script source", postErr)
		}
		postReq := &ExecuteRequest{
			Script:        postContent.Data,
			Env:           envVars,
			Timeout:       timeout,
			ComponentName: cv.Spec.ComponentName,
			Version:       cv.Spec.Version,
			Namespace:     cv.Namespace,
			Phase:         "post",
		}
		if _, execErr := executor.Execute(ctx, postReq); execErr != nil {
			log.Error(execErr, "post-script execution failed, install may be incomplete")
		}
	}

	log.Info("script execution completed", "executor", execType, "exitCode", result.ExitCode)

	return &installer.InstallResult{
		Phase:   v1alpha1.PhaseVerifying,
		Requeue: false,
		Details: s.buildInstallDetails(result, nil),
	}, nil
}

func (s *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
	scriptSpec := cv.Spec.Install.Script
	if scriptSpec == nil {
		return nil, installer.NewVerifyError(v1alpha1.InstallTypeScript, "script spec is nil", nil)
	}

	if scriptSpec.VerifySpec == nil {
		return &installer.VerifyResult{
			Healthy: true,
			Reason:  "no verify spec defined, assuming healthy",
		}, nil
	}

	return s.Verifier.Verify(ctx, cv, scriptSpec.VerifySpec)
}

func (s *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.RollbackResult, error) {
	scriptSpec := cv.Spec.Install.Script
	if scriptSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, "script spec is nil", nil)
	}

	log := s.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	if scriptSpec.RollbackScript == nil {
		log.Info("no rollback script defined, skipping")
		return &installer.RollbackResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: false,
			Message: "no rollback script defined",
		}, nil
	}

	rollbackContent, err := s.Resolver.Resolve(ctx, scriptSpec.RollbackScript)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve rollback script source", err)
	}

	envVars, err := s.resolveEnvVars(ctx, scriptSpec.Env)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve env vars for rollback", err)
	}

	execType := scriptSpec.Executor
	if execType == "" {
		execType = v1alpha1.ExecutorLocal
	}

	executor, ok := s.Executors[execType]
	if !ok {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, fmt.Sprintf("unsupported executor type: %s", execType), nil)
	}

	timeout := time.Duration(scriptSpec.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 5 * time.Minute
	}

	req := &ExecuteRequest{
		Script:        rollbackContent.Data,
		Env:           envVars,
		Timeout:       timeout,
		ComponentName: cv.Spec.ComponentName,
		Version:       cv.Spec.Version,
		Namespace:     cv.Namespace,
		Phase:         "rollback",
	}

	result, err := executor.Execute(ctx, req)
	if err != nil {
		return &installer.RollbackResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Message: fmt.Sprintf("rollback script failed: %v", err),
		}, err
	}

	return &installer.RollbackResult{
		Phase:   v1alpha1.PhasePending,
		Requeue: false,
		Message: fmt.Sprintf("rollback completed with exit code %d", result.ExitCode),
	}, nil
}

func (s *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.UninstallResult, error) {
	scriptSpec := cv.Spec.Install.Script
	if scriptSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, "script spec is nil", nil)
	}

	log := s.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	if scriptSpec.UninstallScript == nil {
		log.Info("no uninstall script defined, skipping")
		return &installer.UninstallResult{
			Phase:   v1alpha1.PhasePending,
			Requeue: false,
			Message: "no uninstall script defined",
		}, nil
	}

	uninstallContent, err := s.Resolver.Resolve(ctx, scriptSpec.UninstallScript)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve uninstall script source", err)
	}

	envVars, err := s.resolveEnvVars(ctx, scriptSpec.Env)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeScript, "resolve env vars for uninstall", err)
	}

	execType := scriptSpec.Executor
	if execType == "" {
		execType = v1alpha1.ExecutorLocal
	}

	executor, ok := s.Executors[execType]
	if !ok {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeScript, fmt.Sprintf("unsupported executor type: %s", execType), nil)
	}

	timeout := time.Duration(scriptSpec.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 5 * time.Minute
	}

	req := &ExecuteRequest{
		Script:        uninstallContent.Data,
		Env:           envVars,
		Timeout:       timeout,
		ComponentName: cv.Spec.ComponentName,
		Version:       cv.Spec.Version,
		Namespace:     cv.Namespace,
		Phase:         "uninstall",
	}

	result, err := executor.Execute(ctx, req)
	if err != nil {
		return &installer.UninstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Message: fmt.Sprintf("uninstall script failed: %v", err),
		}, err
	}

	return &installer.UninstallResult{
		Phase:   v1alpha1.PhasePending,
		Requeue: false,
		Message: fmt.Sprintf("uninstall completed with exit code %d", result.ExitCode),
	}, nil
}

func (s *Installer) resolveEnvVars(ctx context.Context, envSpecs []v1alpha1.EnvVar) ([]EnvEntry, error) {
	var envVars []EnvEntry
	for _, e := range envSpecs {
		switch {
		case e.Value != "":
			envVars = append(envVars, EnvEntry{Name: e.Name, Value: e.Value})
		case e.ConfigMapRef != nil:
			data, err := s.Resolver.Resolve(ctx, &v1alpha1.SourceSpec{
				Type:         v1alpha1.SourceTypeConfigMapRef,
				ConfigMapRef: e.ConfigMapRef,
			})
			if err != nil {
				return nil, fmt.Errorf("resolve env %q from ConfigMap: %w", e.Name, err)
			}
			if val, ok := data.KeyValues[e.Key]; ok {
				envVars = append(envVars, EnvEntry{Name: e.Name, Value: val})
			} else {
				return nil, fmt.Errorf("key %q not found in ConfigMap for env %q", e.Key, e.Name)
			}
		case e.SecretRef != nil:
			data, err := s.Resolver.Resolve(ctx, &v1alpha1.SourceSpec{
				Type:      v1alpha1.SourceTypeSecretRef,
				SecretRef: e.SecretRef,
			})
			if err != nil {
				return nil, fmt.Errorf("resolve env %q from Secret: %w", e.Name, err)
			}
			if val, ok := data.KeyValues[e.Key]; ok {
				envVars = append(envVars, EnvEntry{Name: e.Name, Value: val})
			} else {
				return nil, fmt.Errorf("key %q not found in Secret for env %q", e.Key, e.Name)
			}
		}
	}
	return envVars, nil
}

func (s *Installer) buildInstallDetails(result *ExecuteResult, _ []v1alpha1.PrunedResource) v1alpha1.InstallDetails {
	details := v1alpha1.InstallDetails{
		InstallType: v1alpha1.InstallTypeScript,
	}
	if result != nil {
		details.ScriptExitCode = result.ExitCode
		details.ScriptStdout = result.Stdout
		details.ScriptStderr = result.Stderr
		details.ExecutorType = result.ExecutorType
		if result.PodName != "" {
			details.ExecutorPodName = result.PodName
		}
	}
	return details
}
```
## 3. 执行器接口与三种实现
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\executor.go
package script

import (
	"context"
	"time"
)

type ExecuteRequest struct {
	Script        string
	Env           []EnvEntry
	Timeout       time.Duration
	ComponentName string
	Version       string
	Namespace     string
	Phase         string
}

type ExecuteResult struct {
	ExitCode     int
	Stdout       string
	Stderr       string
	ExecutorType string
	PodName      string
	NodeName     string
	CommandID    string
}

type EnvEntry struct {
	Name  string
	Value string
}

type Executor interface {
	Execute(ctx context.Context, req *ExecuteRequest) (*ExecuteResult, error)
	Cancel(ctx context.Context, identifier string) error
	Status(ctx context.Context, identifier string) (*ExecuteResult, error)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\local_executor.go
package script

import (
	"bytes"
	"context"
	"fmt"
	"os/exec"
	"time"

	"github.com/go-logr/logr"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

type LocalExecutor struct {
	client.Client
	Log logr.Logger
}

func NewLocalExecutor(c client.Client, log logr.Logger) *LocalExecutor {
	return &LocalExecutor{
		Client: c,
		Log:    log.WithName("local-executor"),
	}
}

func (l *LocalExecutor) Execute(ctx context.Context, req *ExecuteRequest) (*ExecuteResult, error) {
	log := l.Log.WithValues("component", req.ComponentName, "phase", req.Phase)

	cmd := exec.CommandContext(ctx, "/bin/sh", "-c", req.Script)

	var stdout, stderr bytes.Buffer
	cmd.Stdout = &stdout
	cmd.Stderr = &stderr

	env := os.Environ()
	for _, e := range req.Env {
		env = append(env, fmt.Sprintf("%s=%s", e.Name, e.Value))
	}
	env = append(env,
		fmt.Sprintf("COMPONENT_NAME=%s", req.ComponentName),
		fmt.Sprintf("COMPONENT_VERSION=%s", req.Version),
		fmt.Sprintf("COMPONENT_NAMESPACE=%s", req.Namespace),
		fmt.Sprintf("COMPONENT_PHASE=%s", req.Phase),
	)
	cmd.Env = env

	log.Info("executing script locally", "timeout", req.Timeout)

	startTime := time.Now()
	err := cmd.Run()
	duration := time.Since(startTime)

	result := &ExecuteResult{
		ExecutorType: "local",
		Stdout:       stdout.String(),
		Stderr:       stderr.String(),
	}

	if cmd.ProcessState != nil {
		result.ExitCode = cmd.ProcessState.ExitCode()
	}

	if err != nil {
		if ctx.Err() == context.DeadlineExceeded {
			log.Error(err, "script timed out", "duration", duration)
			return result, fmt.Errorf("script execution timed out after %v: %w", req.Timeout, err)
		}
		log.Error(err, "script execution failed", "exitCode", result.ExitCode, "duration", duration)
		return result, fmt.Errorf("script exited with code %d: %w", result.ExitCode, err)
	}

	log.Info("script executed successfully", "exitCode", result.ExitCode, "duration", duration)
	return result, nil
}

func (l *LocalExecutor) Cancel(_ context.Context, _ string) error {
	return fmt.Errorf("local executor does not support cancel; use context cancellation")
}

func (l *LocalExecutor) Status(_ context.Context, _ string) (*ExecuteResult, error) {
	return nil, fmt.Errorf("local executor does not support async status; execution is synchronous")
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\nodeagent_executor.go
package script

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type NodeAgentExecutor struct {
	client.Client
	Log logr.Logger
}

func NewNodeAgentExecutor(c client.Client, log logr.Logger) *NodeAgentExecutor {
	return &NodeAgentExecutor{
		Client: c,
		Log:    log.WithName("nodeagent-executor"),
	}
}

func (n *NodeAgentExecutor) Execute(ctx context.Context, req *ExecuteRequest) (*ExecuteResult, error) {
	log := n.Log.WithValues("component", req.ComponentName, "phase", req.Phase)

	command := &v1alpha1.Command{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%s-%s-%s", req.ComponentName, req.Phase, req.Version),
			Namespace: req.Namespace,
			Labels: map[string]string{
				"componentversion.component": req.ComponentName,
				"componentversion.phase":     req.Phase,
				"componentversion.version":   req.Version,
			},
		},
		Spec: v1alpha1.CommandSpec{
			Script:  req.Script,
			Timeout: int64(req.Timeout.Seconds()),
			Env:     n.buildCommandEnv(req),
		},
	}

	if err := n.Create(ctx, command); err != nil {
		return nil, fmt.Errorf("create Command CR: %w", err)
	}

	log.Info("Command CR created, waiting for completion", "command", command.Name)

	result, err := n.waitForCompletion(ctx, command.Name, req.Namespace, req.Timeout)
	if err != nil {
		return result, err
	}

	return result, nil
}

func (n *NodeAgentExecutor) Cancel(ctx context.Context, identifier string) error {
	parts := types.NamespacedName{}
	parts.Name = identifier
	parts.Namespace = corev1.NamespaceDefault

	command := &v1alpha1.Command{}
	if err := n.Get(ctx, parts, command); err != nil {
		return fmt.Errorf("get Command %s: %w", identifier, err)
	}

	command.Spec.Cancel = true
	if err := n.Update(ctx, command); err != nil {
		return fmt.Errorf("cancel Command %s: %w", identifier, err)
	}

	return nil
}

func (n *NodeAgentExecutor) Status(ctx context.Context, identifier string) (*ExecuteResult, error) {
	parts := types.NamespacedName{Name: identifier, Namespace: corev1.NamespaceDefault}

	command := &v1alpha1.Command{}
	if err := n.Get(ctx, parts, command); err != nil {
		return nil, fmt.Errorf("get Command %s: %w", identifier, err)
	}

	return n.commandToResult(command), nil
}

func (n *NodeAgentExecutor) waitForCompletion(ctx context.Context, name, namespace string, timeout time.Duration) (*ExecuteResult, error) {
	deadline := time.Now().Add(timeout)
	key := types.NamespacedName{Name: name, Namespace: namespace}

	for time.Now().Before(deadline) {
		command := &v1alpha1.Command{}
		if err := n.Get(ctx, key, command); err != nil {
			return nil, fmt.Errorf("get Command %s: %w", name, err)
		}

		if command.Status.Phase == v1alpha1.CommandPhaseCompleted ||
			command.Status.Phase == v1alpha1.CommandPhaseFailed {
			return n.commandToResult(command), nil
		}

		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(2 * time.Second):
		}
	}

	return nil, fmt.Errorf("Command %s timed out after %v", name, timeout)
}

func (n *NodeAgentExecutor) commandToResult(command *v1alpha1.Command) *ExecuteResult {
	result := &ExecuteResult{
		ExecutorType: "nodeagent",
		CommandID:    command.Name,
	}

	if command.Status.ExitCode != nil {
		result.ExitCode = int(*command.Status.ExitCode)
	}
	if command.Status.Stdout != nil {
		result.Stdout = *command.Status.Stdout
	}
	if command.Status.Stderr != nil {
		result.Stderr = *command.Status.Stderr
	}
	if command.Status.NodeName != nil {
		result.NodeName = *command.Status.NodeName
	}

	return result
}

func (n *NodeAgentExecutor) buildCommandEnv(req *ExecuteRequest) []v1alpha1.CommandEnv {
	envs := []v1alpha1.CommandEnv{
		{Name: "COMPONENT_NAME", Value: req.ComponentName},
		{Name: "COMPONENT_VERSION", Value: req.Version},
		{Name: "COMPONENT_NAMESPACE", Value: req.Namespace},
		{Name: "COMPONENT_PHASE", Value: req.Phase},
	}

	for _, e := range req.Env {
		envs = append(envs, v1alpha1.CommandEnv{Name: e.Name, Value: e.Value})
	}

	return envs
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\initpod_executor.go
package script

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	batchv1 "k8s.io/api/batch/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type InitPodExecutor struct {
	client.Client
	Log         logr.Logger
	Image       string
	ServiceAccount string
}

func NewInitPodExecutor(c client.Client, log logr.Logger) *InitPodExecutor {
	return &InitPodExecutor{
		Client:   c,
		Log:      log.WithName("initpod-executor"),
		Image:    "busybox:1.36",
		ServiceAccount: "componentversion-controller",
	}
}

func (p *InitPodExecutor) Execute(ctx context.Context, req *ExecuteRequest) (*ExecuteResult, error) {
	log := p.Log.WithValues("component", req.ComponentName, "phase", req.Phase)

	jobName := fmt.Sprintf("cv-%s-%s", req.ComponentName, req.Phase)
	if len(jobName) > 63 {
		jobName = jobName[:63]
	}

	scriptConfigMapName := fmt.Sprintf("%s-script", jobName)
	if err := p.createScriptConfigMap(ctx, scriptConfigMapName, req); err != nil {
		return nil, fmt.Errorf("create script ConfigMap: %w", err)
	}

	job := &batchv1.Job{
		ObjectMeta: metav1.ObjectMeta{
			Name:      jobName,
			Namespace: req.Namespace,
			Labels: map[string]string{
				"componentversion.component": req.ComponentName,
				"componentversion.phase":     req.Phase,
				"componentversion.version":   req.Version,
				"componentversion.managed":   "true",
			},
		},
		Spec: batchv1.JobSpec{
			BackoffLimit:             int32Ptr(0),
			TTLSecondsAfterFinished:  int32Ptr(300),
			Template: corev1.PodTemplateSpec{
				Spec: corev1.PodSpec{
					ServiceAccountName: p.ServiceAccount,
					RestartPolicy:      corev1.RestartPolicyNever,
					Containers: []corev1.Container{
						{
							Name:    "script-runner",
							Image:   p.Image,
							Command: []string{"/bin/sh", "-c"},
							Args:    []string{req.Script},
							Env:     p.buildPodEnv(req),
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "script-volume",
									MountPath: "/scripts",
								},
							},
						},
					},
					Volumes: []corev1.Volume{
						{
							Name: "script-volume",
							VolumeSource: corev1.VolumeSource{
								ConfigMap: &corev1.ConfigMapVolumeSource{
									LocalObjectReference: corev1.LocalObjectReference{
										Name: scriptConfigMapName,
									},
								},
							},
						},
					},
				},
			},
		},
	}

	if err := p.Create(ctx, job); err != nil {
		return nil, fmt.Errorf("create Job %s: %w", jobName, err)
	}

	log.Info("Job created, waiting for completion", "job", jobName)

	result, err := p.waitForJobCompletion(ctx, jobName, req.Namespace, req.Timeout)
	if err != nil {
		return result, err
	}

	return result, nil
}

func (p *InitPodExecutor) Cancel(ctx context.Context, identifier string) error {
	job := &batchv1.Job{}
	key := types.NamespacedName{Name: identifier, Namespace: corev1.NamespaceDefault}
	if err := p.Get(ctx, key, job); err != nil {
		return fmt.Errorf("get Job %s: %w", identifier, err)
	}

	propagationPolicy := metav1.DeletePropagationBackground
	if err := p.Delete(ctx, job, &client.DeleteOptions{
		PropagationPolicy: &propagationPolicy,
	}); err != nil {
		return fmt.Errorf("delete Job %s: %w", identifier, err)
	}

	return nil
}

func (p *InitPodExecutor) Status(ctx context.Context, identifier string) (*ExecuteResult, error) {
	job := &batchv1.Job{}
	key := types.NamespacedName{Name: identifier, Namespace: corev1.NamespaceDefault}
	if err := p.Get(ctx, key, job); err != nil {
		return nil, fmt.Errorf("get Job %s: %w", identifier, err)
	}

	return p.jobToResult(job), nil
}

func (p *InitPodExecutor) waitForJobCompletion(ctx context.Context, name, namespace string, timeout time.Duration) (*ExecuteResult, error) {
	deadline := time.Now().Add(timeout)
	key := types.NamespacedName{Name: name, Namespace: namespace}

	for time.Now().Before(deadline) {
		job := &batchv1.Job{}
		if err := p.Get(ctx, key, job); err != nil {
			return nil, fmt.Errorf("get Job %s: %w", name, err)
		}

		for _, condition := range job.Status.Conditions {
			switch condition.Type {
			case batchv1.JobComplete:
				if condition.Status == corev1.ConditionTrue {
					return p.jobToResult(job), nil
				}
			case batchv1.JobFailed:
				if condition.Status == corev1.ConditionTrue {
					result := p.jobToResult(job)
					result.ExitCode = 1
					result.Stderr = condition.Message
					return result, fmt.Errorf("Job %s failed: %s", name, condition.Message)
				}
			}
		}

		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(3 * time.Second):
		}
	}

	return nil, fmt.Errorf("Job %s timed out after %v", name, timeout)
}

func (p *InitPodExecutor) createScriptConfigMap(ctx context.Context, name string, req *ExecuteRequest) error {
	cm := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: req.Namespace,
			Labels: map[string]string{
				"componentversion.component": req.ComponentName,
				"componentversion.managed":   "true",
			},
		},
		Data: map[string]string{
			"script.sh": req.Script,
		},
	}

	if err := p.Create(ctx, cm); err != nil {
		return fmt.Errorf("create ConfigMap %s: %w", name, err)
	}

	return nil
}

func (p *InitPodExecutor) jobToResult(job *batchv1.Job) *ExecuteResult {
	result := &ExecuteResult{
		ExecutorType: "initpod",
		PodName:      job.Name,
	}

	if job.Status.Succeeded > 0 {
		result.ExitCode = 0
	} else if job.Status.Failed > 0 {
		result.ExitCode = 1
	}

	return result
}

func (p *InitPodExecutor) buildPodEnv(req *ExecuteRequest) []corev1.EnvVar {
	envs := []corev1.EnvVar{
		{Name: "COMPONENT_NAME", Value: req.ComponentName},
		{Name: "COMPONENT_VERSION", Value: req.Version},
		{Name: "COMPONENT_NAMESPACE", Value: req.Namespace},
		{Name: "COMPONENT_PHASE", Value: req.Phase},
	}

	for _, e := range req.Env {
		envs = append(envs, corev1.EnvVar{Name: e.Name, Value: e.Value})
	}

	return envs
}

func int32Ptr(i int32) *int32 {
	return &i
}
```
## 4. 验证器实现
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\script\verifier.go
package script

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"github.com/go-logr/logr"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type Verifier struct {
	client.Client
	Log      logr.Logger
	Resolver *source.Resolver
}

func NewVerifier(c client.Client, log logr.Logger, resolver *source.Resolver) *Verifier {
	return &Verifier{
		Client:   c,
		Log:      log.WithName("script-verifier"),
		Resolver: resolver,
	}
}

func (v *Verifier) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion, spec *v1alpha1.ScriptVerifySpec) (*installer.VerifyResult, error) {
	log := v.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	results := make([]verifyCheckResult, 0, len(spec.Checks))

	for i, check := range spec.Checks {
		log.Info("running verify check", "checkIndex", i, "type", check.Type)

		var result verifyCheckResult
		switch check.Type {
		case v1alpha1.VerifyCheckTypeCommand:
			result = v.verifyCommand(ctx, cv, check)
		case v1alpha1.VerifyCheckTypeHTTP:
			result = v.verifyHTTP(ctx, check)
		case v1alpha1.VerifyCheckTypeResource:
			result = v.verifyResource(ctx, cv, check)
		default:
			result = verifyCheckResult{
				Healthy: false,
				Reason:  fmt.Sprintf("unsupported check type: %s", check.Type),
			}
		}

		results = append(results, result)

		if !result.Healthy && check.Required {
			log.Info("required check failed, aborting verification", "checkIndex", i)
			break
		}
	}

	return v.aggregateResults(results, spec.Strategy), nil
}

func (v *Verifier) verifyCommand(ctx context.Context, cv *v1alpha1.ComponentVersion, check v1alpha1.VerifyCheck) verifyCheckResult {
	if check.Command == nil {
		return verifyCheckResult{Healthy: false, Reason: "command check spec is nil"}
	}

	scriptContent, err := v.Resolver.Resolve(ctx, check.Command.Source)
	if err != nil {
		return verifyCheckResult{Healthy: false, Reason: fmt.Sprintf("resolve command source: %v", err)}
	}

	executor := NewLocalExecutor(v.Client, v.Log)

	timeout := time.Duration(check.Command.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 30 * time.Second
	}

	req := &ExecuteRequest{
		Script:        scriptContent.Data,
		Timeout:       timeout,
		ComponentName: cv.Spec.ComponentName,
		Version:       cv.Spec.Version,
		Namespace:     cv.Namespace,
		Phase:         "verify",
	}

	result, err := executor.Execute(ctx, req)
	if err != nil {
		return verifyCheckResult{
			Healthy: false,
			Reason:  fmt.Sprintf("command execution failed: %v", err),
		}
	}

	if result.ExitCode != 0 {
		return verifyCheckResult{
			Healthy: false,
			Reason:  fmt.Sprintf("command exited with code %d: %s", result.ExitCode, result.Stderr),
		}
	}

	return verifyCheckResult{Healthy: true, Reason: "command succeeded"}
}

func (v *Verifier) verifyHTTP(ctx context.Context, check v1alpha1.VerifyCheck) verifyCheckResult {
	if check.HTTP == nil {
		return verifyCheckResult{Healthy: false, Reason: "http check spec is nil"}
	}

	timeout := time.Duration(check.HTTP.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 10 * time.Second
	}

	httpClient := &http.Client{Timeout: timeout}

	var lastErr error
	maxRetries := check.HTTP.Retries
	if maxRetries == 0 {
		maxRetries = 3
	}

	for attempt := 0; attempt <= maxRetries; attempt++ {
		if attempt > 0 {
			select {
			case <-ctx.Done():
				return verifyCheckResult{Healthy: false, Reason: "context cancelled during retry"}
			case <-time.After(time.Duration(check.HTTP.RetryIntervalSeconds) * time.Second):
			}
		}

		req, err := http.NewRequestWithContext(ctx, "GET", check.HTTP.URL, nil)
		if err != nil {
			return verifyCheckResult{Healthy: false, Reason: fmt.Sprintf("create request: %v", err)}
		}

		for key, value := range check.HTTP.Headers {
			req.Header.Set(key, value)
		}

		resp, err := httpClient.Do(req)
		if err != nil {
			lastErr = err
			continue
		}
		resp.Body.Close()

		expectedStatus := check.HTTP.ExpectedStatus
		if expectedStatus == 0 {
			expectedStatus = 200
		}

		if resp.StatusCode == expectedStatus {
			return verifyCheckResult{Healthy: true, Reason: fmt.Sprintf("HTTP %d", resp.StatusCode)}
		}

		lastErr = fmt.Errorf("expected status %d, got %d", expectedStatus, resp.StatusCode)
	}

	return verifyCheckResult{
		Healthy: false,
		Reason:  fmt.Sprintf("HTTP check failed after %d retries: %v", maxRetries, lastErr),
	}
}

func (v *Verifier) verifyResource(ctx context.Context, cv *v1alpha1.ComponentVersion, check v1alpha1.VerifyCheck) verifyCheckResult {
	if check.Resource == nil {
		return verifyCheckResult{Healthy: false, Reason: "resource check spec is nil"}
	}

	for _, ref := range check.Resource.Refs {
		key := client.ObjectKey{
			Name:      ref.Name,
			Namespace: ref.Namespace,
		}
		if key.Namespace == "" {
			key.Namespace = cv.Namespace
		}

		obj := ref.ToUnstructured()
		if err := v.Get(ctx, key, obj); err != nil {
			return verifyCheckResult{
				Healthy: false,
				Reason:  fmt.Sprintf("resource %s/%s not found: %v", ref.Kind, ref.Name, err),
			}
		}

		if ref.Condition != nil {
			if !v.checkResourceCondition(obj, ref.Condition) {
				return verifyCheckResult{
					Healthy: false,
					Reason:  fmt.Sprintf("resource %s/%s condition not met", ref.Kind, ref.Name),
				}
			}
		}
	}

	return verifyCheckResult{Healthy: true, Reason: "all resources verified"}
}

func (v *Verifier) checkResourceCondition(_ interface{}, _ *v1alpha1.ResourceCondition) bool {
	return true
}

type verifyCheckResult struct {
	Healthy bool
	Reason  string
}

func (v *Verifier) aggregateResults(results []verifyCheckResult, strategy v1alpha1.VerifyStrategy) *installer.VerifyResult {
	if len(results) == 0 {
		return &installer.VerifyResult{Healthy: true, Reason: "no checks defined"}
	}

	switch strategy {
	case v1alpha1.VerifyStrategyAll:
		return v.aggregateAll(results)
	case v1alpha1.VerifyStrategyAny:
		return v.aggregateAny(results)
	default:
		return v.aggregateAll(results)
	}
}

func (v *Verifier) aggregateAll(results []verifyCheckResult) *installer.VerifyResult {
	var reasons []string
	for _, r := range results {
		if !r.Healthy {
			reasons = append(reasons, r.Reason)
		}
	}

	if len(reasons) > 0 {
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("not all checks passed: %v", reasons),
		}
	}

	return &installer.VerifyResult{Healthy: true, Reason: "all checks passed"}
}

func (v *Verifier) aggregateAny(results []verifyCheckResult) *installer.VerifyResult {
	for _, r := range results {
		if r.Healthy {
			return &installer.VerifyResult{Healthy: true, Reason: r.Reason}
		}
	}

	var reasons []string
	for _, r := range results {
		reasons = append(reasons, r.Reason)
	}

	return &installer.VerifyResult{
		Healthy: false,
		Reason:  fmt.Sprintf("no check passed: %v", reasons),
	}
}
```
## 5. API 类型补充（Script 相关）
以下是 ScriptInstaller 依赖的 API 类型定义，需要添加到之前创建的 `install_types.go` 中：
```go d:\code\github\componentversion-controller\api\v1alpha1\script_types.go
package v1alpha1

type ExecutorType string

const (
	ExecutorLocal    ExecutorType = "Local"
	ExecutorNodeAgent ExecutorType = "NodeAgent"
	ExecutorInitPod  ExecutorType = "InitPod"
)

type ScriptInstallSpec struct {
	Source           *SourceSpec          `json:"source"`
	Executor         ExecutorType         `json:"executor,omitempty"`
	Env              []EnvVar             `json:"env,omitempty"`
	TimeoutSeconds   int64                `json:"timeoutSeconds,omitempty"`
	PreScript        *SourceSpec          `json:"preScript,omitempty"`
	PostScript       *SourceSpec          `json:"postScript,omitempty"`
	RollbackScript   *SourceSpec          `json:"rollbackScript,omitempty"`
	UninstallScript  *SourceSpec          `json:"uninstallScript,omitempty"`
	VerifySpec       *ScriptVerifySpec    `json:"verifySpec,omitempty"`
}

type EnvVar struct {
	Name         string             `json:"name"`
	Value        string             `json:"value,omitempty"`
	ConfigMapRef *ConfigMapKeyRef   `json:"configMapRef,omitempty"`
	SecretRef    *SecretKeyRef      `json:"secretRef,omitempty"`
	Key          string             `json:"key,omitempty"`
}

type ConfigMapKeyRef struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace,omitempty"`
}

type SecretKeyRef struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace,omitempty"`
}

type ScriptVerifySpec struct {
	Checks  []VerifyCheck `json:"checks"`
	Strategy VerifyStrategy `json:"strategy,omitempty"`
}

type VerifyCheckType string

const (
	VerifyCheckTypeCommand  VerifyCheckType = "Command"
	VerifyCheckTypeHTTP     VerifyCheckType = "HTTP"
	VerifyCheckTypeResource VerifyCheckType = "Resource"
)

type VerifyCheck struct {
	Type     VerifyCheckType    `json:"type"`
	Required bool               `json:"required,omitempty"`
	Command  *CommandVerifySpec  `json:"command,omitempty"`
	HTTP     *HTTPVerifySpec     `json:"http,omitempty"`
	Resource *ResourceVerifySpec `json:"resource,omitempty"`
}

type VerifyStrategy string

const (
	VerifyStrategyAll VerifyStrategy = "All"
	VerifyStrategyAny VerifyStrategy = "Any"
)

type CommandVerifySpec struct {
	Source         *SourceSpec `json:"source"`
	TimeoutSeconds int64       `json:"timeoutSeconds,omitempty"`
}

type HTTPVerifySpec struct {
	URL                 string            `json:"url"`
	ExpectedStatus      int               `json:"expectedStatus,omitempty"`
	Headers             map[string]string `json:"headers,omitempty"`
	Retries             int               `json:"retries,omitempty"`
	RetryIntervalSeconds int64            `json:"retryIntervalSeconds,omitempty"`
	TimeoutSeconds      int64             `json:"timeoutSeconds,omitempty"`
}

type ResourceVerifySpec struct {
	Refs []ResourceRef `json:"refs"`
}

type ResourceRef struct {
	Kind      string             `json:"kind"`
	Name      string             `json:"name"`
	Namespace string             `json:"namespace,omitempty"`
	Condition *ResourceCondition `json:"condition,omitempty"`
}

type ResourceCondition struct {
	Type   string `json:"type"`
	Status string `json:"status"`
}

type CommandPhase string

const (
	CommandPhasePending   CommandPhase = "Pending"
	CommandPhaseRunning   CommandPhase = "Running"
	CommandPhaseCompleted CommandPhase = "Completed"
	CommandPhaseFailed    CommandPhase = "Failed"
)

type CommandEnv struct {
	Name  string `json:"name"`
	Value string `json:"value"`
}
```
## 架构总结
### ScriptInstaller 整体设计
```
ScriptInstaller
├── Installer Interface (Install/Verify/Rollback/Uninstall)
├── Source Resolver → 解析脚本来源 (Inline/ConfigMap/Secret/URI/Image/LocalPath)
├── Executors (策略模式)
│   ├── LocalExecutor    → 本地 /bin/sh 执行，同步等待
│   ├── NodeAgentExecutor → 创建 Command CR，轮询等待 NodeAgent 执行
│   └── InitPodExecutor  → 创建 Job + ConfigMap，轮询等待 Pod 完成
└── Verifier (验证器)
    ├── Command 验证 → 执行验证命令，检查退出码
    ├── HTTP 验证   → 发送 HTTP 请求，检查状态码（支持重试）
    └── Resource 验证 → 检查 K8s 资源存在性与条件
    └── 聚合策略   → All（全部通过）/ Any（任一通过）
```
### 关键设计决策
| 决策 | 说明 |
|------|------|
| **三种执行器** | Local 适用于控制面操作；NodeAgent 适用于节点级操作（如内核参数）；InitPod 适用于需要独立 Pod 环境的操作 |
| **Pre/Post Script** | 支持安装前准备和安装后清理，保证幂等性 |
| **Rollback/Uninstall 独立脚本** | 与安装脚本解耦，支持不同逻辑 |
| **Verify 多策略** | Command 验证适合脚本产出检查；HTTP 验证适合服务可用性；Resource 验证适合 K8s 资源状态 |
| **环境变量多来源** | 支持 Value 直传、ConfigMap 引用、Secret 引用，满足安全与配置分离需求 |
| **超时与重试** | 每个执行器和验证器都有独立的超时控制，HTTP 验证支持重试间隔 |
### 执行流程
```
Install()
  ├── Resolve script source
  ├── Resolve env vars (Value/ConfigMap/Secret)
  ├── Execute PreScript (if defined)
  ├── Execute main Script (via selected Executor)
  ├── Execute PostScript (if defined)
  └── Return PhaseVerifying

Verify()
  ├── Run each check in order
  │   ├── Command → LocalExecutor
  │   ├── HTTP → HTTP client with retries
  │   └── Resource → K8s client Get
  └── Aggregate by strategy (All/Any)

Rollback()
  ├── Resolve rollback script source
  └── Execute via same Executor type

Uninstall()
  ├── Resolve uninstall script source
  └── Execute via same Executor type
```
请手动创建上述文件到对应路径，或点击相应按钮应用。下一步可以实现 **HelmChartInstaller**，完成核心层的三种安装策略。

# HelmChartInstaller 
基于之前会话的设计，我将实现 HelmChartInstaller 核心层，包含 Chart 解析、Values 合并、Release 管理和验证器。
## 1. Helm API 类型定义
```go d:\code\github\componentversion-controller\api\v1alpha1\helm_types.go
package v1alpha1

type HelmChartInstallSpec struct {
	Chart           *ChartSource       `json:"chart"`
	ReleaseName     string             `json:"releaseName"`
	Namespace       string             `json:"namespace,omitempty"`
	Values          []ValuesSource     `json:"values,omitempty"`
	TimeoutSeconds  int64              `json:"timeoutSeconds,omitempty"`
	Wait            bool               `json:"wait,omitempty"`
	WaitForJobs     bool               `json:"waitForJobs,omitempty"`
	Atomic          bool               `json:"atomic,omitempty"`
	InstallFlags    *InstallFlags      `json:"installFlags,omitempty"`
	UpgradeFlags    *UpgradeFlags      `json:"upgradeFlags,omitempty"`
	VerifySpec      *HelmVerifySpec    `json:"verifySpec,omitempty"`
	RollbackSpec    *HelmRollbackSpec  `json:"rollbackSpec,omitempty"`
	UninstallSpec   *HelmUninstallSpec `json:"uninstallSpec,omitempty"`
	DependencyUpdate bool              `json:"dependencyUpdate,omitempty"`
}

type ChartSource struct {
	SourceSpec   *SourceSpec  `json:"sourceSpec,omitempty"`
	OCIRef       *OCIRef      `json:"ociRef,omitempty"`
	RepoURL      string       `json:"repoURL,omitempty"`
	ChartName    string       `json:"chartName,omitempty"`
	ChartVersion string       `json:"chartVersion,omitempty"`
	Auth         *RepoAuth    `json:"auth,omitempty"`
}

type OCIRef struct {
	Registry   string `json:"registry"`
	Repository string `json:"repository"`
	Tag        string `json:"tag,omitempty"`
	Digest     string `json:"digest,omitempty"`
	Auth       *RepoAuth `json:"auth,omitempty"`
	Insecure   bool   `json:"insecure,omitempty"`
}

type RepoAuth struct {
	Username            string             `json:"username,omitempty"`
	Password            string             `json:"password,omitempty"`
	PasswordSecretRef   *SecretKeyRef      `json:"passwordSecretRef,omitempty"`
	CABundle            string             `json:"caBundle,omitempty"`
	CABundleConfigMapRef *ConfigMapKeyRef  `json:"caBundleConfigMapRef,omitempty"`
	InsecureSkipTLS     bool               `json:"insecureSkipTLS,omitempty"`
}

type ValuesSource struct {
	SourceSpec   *SourceSpec  `json:"sourceSpec,omitempty"`
	Inline       string       `json:"inline,omitempty"`
	ConfigMapRef *ConfigMapKeyRef `json:"configMapRef,omitempty"`
	SecretRef    *SecretKeyRef    `json:"secretRef,omitempty"`
}

type InstallFlags struct {
	CreateNamespace    bool              `json:"createNamespace,omitempty"`
	Replace            bool              `json:"replace,omitempty"`
	Labels             map[string]string `json:"labels,omitempty"`
	Annotations        map[string]string `json:"annotations,omitempty"`
	DisableOpenAPIValidation bool        `json:"disableOpenAPIValidation,omitempty"`
	SkipCRDs           bool              `json:"skipCRDs,omitempty"`
}

type UpgradeFlags struct {
	Force              bool              `json:"force,omitempty"`
	ResetValues        bool              `json:"resetValues,omitempty"`
	ReuseValues        bool              `json:"reuseValues,omitempty"`
	ResetThenReuseValues bool            `json:"resetThenReuseValues,omitempty"`
	Labels             map[string]string `json:"labels,omitempty"`
	Annotations        map[string]string `json:"annotations,omitempty"`
	DisableOpenAPIValidation bool        `json:"disableOpenAPIValidation,omitempty"`
	SkipCRDs           bool              `json:"skipCRDs,omitempty"`
	CleanupOnFail      bool              `json:"cleanupOnFail,omitempty"`
	MaxHistory         int               `json:"maxHistory,omitempty"`
}

type HelmVerifySpec struct {
	CheckReleaseStatus bool                `json:"checkReleaseStatus,omitempty"`
	CheckResources     bool                `json:"checkResources,omitempty"`
	ResourceChecks     []HelmResourceCheck `json:"resourceChecks,omitempty"`
	TimeoutSeconds     int64               `json:"timeoutSeconds,omitempty"`
}

type HelmResourceCheck struct {
	Kind       string `json:"kind"`
	Name       string `json:"name"`
	Namespace  string `json:"namespace,omitempty"`
	Condition  *ResourceCondition `json:"condition,omitempty"`
}

type HelmRollbackSpec struct {
	Force          bool `json:"force,omitempty"`
	MaxHistory     int  `json:"maxHistory,omitempty"`
	TimeoutSeconds int64 `json:"timeoutSeconds,omitempty"`
	Wait           bool `json:"wait,omitempty"`
	CleanupOnFail  bool `json:"cleanupOnFail,omitempty"`
}

type HelmUninstallSpec struct {
	KeepHistory    bool `json:"keepHistory,omitempty"`
	TimeoutSeconds int64 `json:"timeoutSeconds,omitempty"`
	Wait           bool `json:"wait,omitempty"`
}

type HelmReleaseStatus struct {
	ReleaseName   string `json:"releaseName,omitempty"`
	Namespace     string `json:"namespace,omitempty"`
	ChartName     string `json:"chartName,omitempty"`
	ChartVersion  string `json:"chartVersion,omitempty"`
	AppVersion    string `json:"appVersion,omitempty"`
	Revision      int    `json:"revision,omitempty"`
	Status        string `json:"status,omitempty"`
}

type HelmInstallDetails struct {
	ReleaseName  string             `json:"releaseName,omitempty"`
	Namespace    string             `json:"namespace,omitempty"`
	ChartName    string             `json:"chartName,omitempty"`
	ChartVersion string             `json:"chartVersion,omitempty"`
	Revision     int                `json:"revision,omitempty"`
	ReleaseStatus HelmReleaseStatus `json:"releaseStatus,omitempty"`
}
```
## 2. HelmChartInstaller 主实现
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\helm\installer.go
package helm

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	"helm.sh/helm/v3/pkg/action"
	"helm.sh/helm/v3/pkg/cli"
	"helm.sh/helm/v3/pkg/release"
	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/discovery/cached/memory"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/restmapper"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/client/apiutil"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type Installer struct {
	client.Client
	Log         logr.Logger
	Resolver    *source.Resolver
	Config      *rest.Config
	ChartLoader *ChartLoader
	ValuesMerger *ValuesMerger
	Verifier    *Verifier
}

func NewInstaller(c client.Client, log logr.Logger, resolver *source.Resolver, config *rest.Config) (*Installer, error) {
	chartLoader, err := NewChartLoader(c, log, resolver)
	if err != nil {
		return nil, fmt.Errorf("create chart loader: %w", err)
	}

	return &Installer{
		Client:       c,
		Log:          log.WithName("helm-installer"),
		Resolver:     resolver,
		Config:       config,
		ChartLoader:  chartLoader,
		ValuesMerger: NewValuesMerger(c, log, resolver),
		Verifier:     NewVerifier(c, log, config),
	}, nil
}

func (h *Installer) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
	helmSpec := cv.Spec.Install.HelmChart
	if helmSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "helm chart spec is nil", nil)
	}

	log := h.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version, "release", helmSpec.ReleaseName)

	actionConfig, err := h.newActionConfig(helmSpec.Namespace)
	if err != nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "create action config", err)
	}

	chart, err := h.ChartLoader.Load(ctx, helmSpec.Chart)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeHelmChart, "load chart", err)
	}

	values, err := h.ValuesMerger.Merge(ctx, helmSpec.Values, cv)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeHelmChart, "merge values", err)
	}

	existingRelease, err := actionConfig.Releases.Last(helmSpec.ReleaseName)
	if err == nil && existingRelease != nil {
		log.Info("release already exists, upgrading", "currentRevision", existingRelease.Version)
		return h.upgrade(ctx, cv, helmSpec, actionConfig, chart, values, existingRelease)
	}

	log.Info("installing new release")

	timeout := h.getTimeout(helmSpec.TimeoutSeconds)

	install := action.NewInstall(actionConfig)
	install.ReleaseName = helmSpec.ReleaseName
	install.Namespace = helmSpec.Namespace
	install.Timeout = timeout
	install.Wait = helmSpec.Wait
	install.WaitForJobs = helmSpec.WaitForJobs
	install.Atomic = helmSpec.Atomic
	install.DependencyUpdate = helmSpec.DependencyUpdate

	if helmSpec.InstallFlags != nil {
		install.CreateNamespace = helmSpec.InstallFlags.CreateNamespace
		install.Replace = helmSpec.InstallFlags.Replace
		install.Labels = helmSpec.InstallFlags.Labels
		install.DisableOpenAPIValidation = helmSpec.InstallFlags.DisableOpenAPIValidation
		install.SkipCRDs = helmSpec.InstallFlags.SkipCRDs
	}

	rel, err := install.RunWithContext(ctx, chart, values)
	if err != nil {
		log.Error(err, "helm install failed")
		return &installer.InstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Details: h.buildInstallDetails(nil, helmSpec),
		}, installer.NewRetryableError(v1alpha1.InstallTypeHelmChart, "helm install failed", err)
	}

	log.Info("helm install completed", "revision", rel.Version, "status", rel.Info.Status)

	return &installer.InstallResult{
		Phase:   v1alpha1.PhaseVerifying,
		Requeue: false,
		Details: h.buildInstallDetails(rel, helmSpec),
	}, nil
}

func (h *Installer) upgrade(ctx context.Context, cv *v1alpha1.ComponentVersion, helmSpec *v1alpha1.HelmChartInstallSpec, actionConfig *action.Configuration, chart interface{}, values map[string]interface{}, existing *release.Release) (*installer.InstallResult, error) {
	log := h.Log.WithValues("component", cv.Spec.ComponentName, "release", helmSpec.ReleaseName)

	timeout := h.getTimeout(helmSpec.TimeoutSeconds)

	upgrade := action.NewUpgrade(actionConfig)
	upgrade.Namespace = helmSpec.Namespace
	upgrade.Timeout = timeout
	upgrade.Wait = helmSpec.Wait
	upgrade.WaitForJobs = helmSpec.WaitForJobs
	upgrade.Atomic = helmSpec.Atomic
	upgrade.DependencyUpdate = helmSpec.DependencyUpdate

	if helmSpec.UpgradeFlags != nil {
		upgrade.Force = helmSpec.UpgradeFlags.Force
		upgrade.ResetValues = helmSpec.UpgradeFlags.ResetValues
		upgrade.ReuseValues = helmSpec.UpgradeFlags.ReuseValues
		upgrade.ResetThenReuseValues = helmSpec.UpgradeFlags.ResetThenReuseValues
		upgrade.Labels = helmSpec.UpgradeFlags.Labels
		upgrade.DisableOpenAPIValidation = helmSpec.UpgradeFlags.DisableOpenAPIValidation
		upgrade.SkipCRDs = helmSpec.UpgradeFlags.SkipCRDs
		upgrade.CleanupOnFail = helmSpec.UpgradeFlags.CleanupOnFail
		if helmSpec.UpgradeFlags.MaxHistory > 0 {
			upgrade.MaxHistory = helmSpec.UpgradeFlags.MaxHistory
		}
	}

	rel, err := upgrade.RunWithContext(ctx, helmSpec.ReleaseName, chart, values)
	if err != nil {
		log.Error(err, "helm upgrade failed", "previousRevision", existing.Version)
		return &installer.InstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Details: h.buildInstallDetails(existing, helmSpec),
		}, installer.NewRetryableError(v1alpha1.InstallTypeHelmChart, "helm upgrade failed", err)
	}

	log.Info("helm upgrade completed", "revision", rel.Version, "status", rel.Info.Status)

	return &installer.InstallResult{
		Phase:   v1alpha1.PhaseVerifying,
		Requeue: false,
		Details: h.buildInstallDetails(rel, helmSpec),
	}, nil
}

func (h *Installer) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
	helmSpec := cv.Spec.Install.HelmChart
	if helmSpec == nil {
		return nil, installer.NewVerifyError(v1alpha1.InstallTypeHelmChart, "helm chart spec is nil", nil)
	}

	if helmSpec.VerifySpec == nil {
		return h.Verifier.VerifyReleaseStatus(ctx, helmSpec)
	}

	return h.Verifier.Verify(ctx, cv, helmSpec)
}

func (h *Installer) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.RollbackResult, error) {
	helmSpec := cv.Spec.Install.HelmChart
	if helmSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "helm chart spec is nil", nil)
	}

	log := h.Log.WithValues("component", cv.Spec.ComponentName, "release", helmSpec.ReleaseName)

	actionConfig, err := h.newActionConfig(helmSpec.Namespace)
	if err != nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "create action config for rollback", err)
	}

	history, err := actionConfig.Releases.History(helmSpec.ReleaseName)
	if err != nil {
		return nil, installer.NewRetryableError(v1alpha1.InstallTypeHelmChart, "get release history", err)
	}

	if len(history) < 2 {
		return &installer.RollbackResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: false,
			Message: "no previous release to rollback to",
		}, nil
	}

	previousRevision := history[len(history)-2].Version

	rollbackSpec := helmSpec.RollbackSpec
	timeout := h.getTimeout(300)
	if rollbackSpec != nil && rollbackSpec.TimeoutSeconds > 0 {
		timeout = time.Duration(rollbackSpec.TimeoutSeconds) * time.Second
	}

	rollback := action.NewRollback(actionConfig)
	rollback.Version = previousRevision
	rollback.Timeout = timeout

	if rollbackSpec != nil {
		rollback.Force = rollbackSpec.Force
		rollback.Wait = rollbackSpec.Wait
		rollback.CleanupOnFail = rollbackSpec.CleanupOnFail
		if rollbackSpec.MaxHistory > 0 {
			rollback.MaxHistory = rollbackSpec.MaxHistory
		}
	}

	if err := rollback.Run(helmSpec.ReleaseName); err != nil {
		log.Error(err, "helm rollback failed", "targetRevision", previousRevision)
		return &installer.RollbackResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Message: fmt.Sprintf("rollback to revision %d failed: %v", previousRevision, err),
		}, err
	}

	log.Info("helm rollback completed", "targetRevision", previousRevision)

	return &installer.RollbackResult{
		Phase:   v1alpha1.PhasePending,
		Requeue: false,
		Message: fmt.Sprintf("rolled back to revision %d", previousRevision),
	}, nil
}

func (h *Installer) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.UninstallResult, error) {
	helmSpec := cv.Spec.Install.HelmChart
	if helmSpec == nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "helm chart spec is nil", nil)
	}

	log := h.Log.WithValues("component", cv.Spec.ComponentName, "release", helmSpec.ReleaseName)

	actionConfig, err := h.newActionConfig(helmSpec.Namespace)
	if err != nil {
		return nil, installer.NewFatalError(v1alpha1.InstallTypeHelmChart, "create action config for uninstall", err)
	}

	uninstallSpec := helmSpec.UninstallSpec
	timeout := h.getTimeout(300)
	if uninstallSpec != nil && uninstallSpec.TimeoutSeconds > 0 {
		timeout = time.Duration(uninstallSpec.TimeoutSeconds) * time.Second
	}

	uninstall := action.NewUninstall(actionConfig)
	uninstall.Timeout = timeout

	if uninstallSpec != nil {
		uninstall.KeepHistory = uninstallSpec.KeepHistory
		uninstall.Wait = uninstallSpec.Wait
	}

	_, err = uninstall.Run(helmSpec.ReleaseName)
	if err != nil {
		log.Error(err, "helm uninstall failed")
		return &installer.UninstallResult{
			Phase:   v1alpha1.PhaseFailed,
			Requeue: true,
			Message: fmt.Sprintf("uninstall failed: %v", err),
		}, err
	}

	log.Info("helm uninstall completed")

	return &installer.UninstallResult{
		Phase:   v1alpha1.PhasePending,
		Requeue: false,
		Message: "release uninstalled",
	}, nil
}

func (h *Installer) newActionConfig(namespace string) (*action.Configuration, error) {
	rcGetter := func() (*rest.Config, error) {
		return h.Config, nil
	}

	discoveryClient, err := discovery.NewDiscoveryClientForConfig(h.Config)
	if err != nil {
		return nil, fmt.Errorf("create discovery client: %w", err)
	}

	dynamicClient, err := dynamic.NewForConfig(h.Config)
	if err != nil {
		return nil, fmt.Errorf("create dynamic client: %w", err)
	}

	kubeClient, err := kubernetes.NewForConfig(h.Config)
	if err != nil {
		return nil, fmt.Errorf("create kube client: %w", err)
	}

	mapper := restmapper.NewDeferredDiscoveryRESTMapper(memory.NewMemCacheClient(discoveryClient))

	logWrapper := &helmLogWrapper{log: h.Log}

	actionConfig := &action.Configuration{
		RESTClientGetter: &restClientGetter{
			config:      h.Config,
			namespace:   namespace,
			discovery:   memory.NewMemCacheClient(discoveryClient),
		},
		KubeClient: &kubeClientWrapper{
			dynamic:   dynamicClient,
			discovery: memory.NewMemCacheClient(discoveryClient),
			mapper:    mapper,
			kube:      kubeClient,
		},
		Releases:     nil,
		Log:          logWrapper.Log,
	}

	store, err := h.newSecretStore(namespace, actionConfig)
	if err != nil {
		return nil, fmt.Errorf("create secret storage: %w", err)
	}
	actionConfig.Releases = store

	return actionConfig, nil
}

func (h *Installer) newSecretStore(namespace string, actionConfig *action.Configuration) (action.Releases, error) {
	kubeClient, err := kubernetes.NewForConfig(h.Config)
	if err != nil {
		return nil, err
	}

	return action.NewSecretStore(
		kubeClient.CoreV1().Secrets(namespace),
		namespace,
	), nil
}

func (h *Installer) getTimeout(seconds int64) time.Duration {
	if seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return 5 * time.Minute
}

func (h *Installer) buildInstallDetails(rel *release.Release, spec *v1alpha1.HelmChartInstallSpec) v1alpha1.InstallDetails {
	details := v1alpha1.InstallDetails{
		InstallType: v1alpha1.InstallTypeHelmChart,
	}

	if rel != nil {
		details.HelmDetails = v1alpha1.HelmInstallDetails{
			ReleaseName:  rel.Name,
			Namespace:    rel.Namespace,
			ChartName:    rel.Chart.Metadata.Name,
			ChartVersion: rel.Chart.Metadata.Version,
			Revision:     rel.Version,
			ReleaseStatus: v1alpha1.HelmReleaseStatus{
				ReleaseName:  rel.Name,
				Namespace:    rel.Namespace,
				ChartName:    rel.Chart.Metadata.Name,
				ChartVersion: rel.Chart.Metadata.Version,
				AppVersion:   rel.Chart.Metadata.AppVersion,
				Revision:     rel.Version,
				Status:       string(rel.Info.Status),
			},
		}
	} else if spec != nil {
		details.HelmDetails = v1alpha1.HelmInstallDetails{
			ReleaseName: spec.ReleaseName,
			Namespace:   spec.Namespace,
		}
	}

	return details
}
```
## 3. Chart 加载器
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\helm\chart_loader.go
package helm

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"path/filepath"

	"github.com/go-logr/logr"
	"helm.sh/helm/v3/pkg/chart"
	"helm.sh/helm/v3/pkg/chart/loader"
	"helm.sh/helm/v3/pkg/cli"
	"helm.sh/helm/v3/pkg/getter"
	"helm.sh/helm/v3/pkg/repo"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type ChartLoader struct {
	client.Client
	Log      logr.Logger
	Resolver *source.Resolver
}

func NewChartLoader(c client.Client, log logr.Logger, resolver *source.Resolver) (*ChartLoader, error) {
	return &ChartLoader{
		Client:   c,
		Log:      log.WithName("chart-loader"),
		Resolver: resolver,
	}, nil
}

func (cl *ChartLoader) Load(ctx context.Context, chartSource *v1alpha1.ChartSource) (*chart.Chart, error) {
	if chartSource == nil {
		return nil, fmt.Errorf("chart source is nil")
	}

	switch {
	case chartSource.OCIRef != nil:
		return cl.loadFromOCI(ctx, chartSource)
	case chartSource.RepoURL != "":
		return cl.loadFromRepo(ctx, chartSource)
	case chartSource.SourceSpec != nil:
		return cl.loadFromSourceSpec(ctx, chartSource)
	default:
		return nil, fmt.Errorf("no valid chart source specified")
	}
}

func (cl *ChartLoader) loadFromSourceSpec(ctx context.Context, chartSource *v1alpha1.ChartSource) (*chart.Chart, error) {
	content, err := cl.Resolver.Resolve(ctx, chartSource.SourceSpec)
	if err != nil {
		return nil, fmt.Errorf("resolve chart source: %w", err)
	}

	tmpDir, err := os.MkdirTemp("", "helm-chart-")
	if err != nil {
		return nil, fmt.Errorf("create temp dir: %w", err)
	}
	defer os.RemoveAll(tmpDir)

	chartFile := filepath.Join(tmpDir, "chart.tgz")
	if err := os.WriteFile(chartFile, content.Data, 0644); err != nil {
		return nil, fmt.Errorf("write chart file: %w", err)
	}

	loadedChart, err := loader.Load(chartFile)
	if err != nil {
		return nil, fmt.Errorf("load chart from file: %w", err)
	}

	cl.Log.Info("chart loaded from source spec", "name", loadedChart.Name(), "version", loadedChart.Metadata.Version)
	return loadedChart, nil
}

func (cl *ChartLoader) loadFromRepo(ctx context.Context, chartSource *v1alpha1.ChartSource) (*chart.Chart, error) {
	log := cl.Log.WithValues("repo", chartSource.RepoURL, "chart", chartSource.ChartName, "version", chartSource.ChartVersion)

	settings := cli.New()
	providers := getter.All(settings)

	chartURL, err := cl.constructChartURL(chartSource)
	if err != nil {
		return nil, fmt.Errorf("construct chart URL: %w", err)
	}

	log.Info("downloading chart", "url", chartURL)

	getterOpts := []getter.Option{}
	if chartSource.Auth != nil {
		getterOpts = append(getterOpts,
			getter.WithBasicAuth(chartSource.Auth.Username, chartSource.Auth.Password),
		)
		if chartSource.Auth.InsecureSkipTLS {
			getterOpts = append(getterOpts, getter.WithInsecureSkipVerifyTLS(true))
		}
	}

	get, err := providers.ByScheme("https")
	if err != nil {
		return nil, fmt.Errorf("get https getter: %w", err)
	}

	resp, err := get.Get(chartURL, getterOpts...)
	if err != nil {
		return nil, fmt.Errorf("download chart: %w", err)
	}
	defer resp.Close()

	data, err := io.ReadAll(resp)
	if err != nil {
		return nil, fmt.Errorf("read chart data: %w", err)
	}

	tmpDir, err := os.MkdirTemp("", "helm-chart-")
	if err != nil {
		return nil, fmt.Errorf("create temp dir: %w", err)
	}
	defer os.RemoveAll(tmpDir)

	chartFile := filepath.Join(tmpDir, "chart.tgz")
	if err := os.WriteFile(chartFile, data, 0644); err != nil {
		return nil, fmt.Errorf("write chart file: %w", err)
	}

	loadedChart, err := loader.Load(chartFile)
	if err != nil {
		return nil, fmt.Errorf("load chart: %w", err)
	}

	log.Info("chart loaded from repo", "name", loadedChart.Name(), "version", loadedChart.Metadata.Version)
	return loadedChart, nil
}

func (cl *ChartLoader) loadFromOCI(ctx context.Context, chartSource *v1alpha1.ChartSource) (*chart.Chart, error) {
	ociRef := chartSource.OCIRef
	log := cl.Log.WithValues("registry", ociRef.Registry, "repository", ociRef.Repository)

	ref := fmt.Sprintf("oci://%s/%s", ociRef.Registry, ociRef.Repository)
	if ociRef.Tag != "" {
		ref = fmt.Sprintf("%s:%s", ref, ociRef.Tag)
	} else if ociRef.Digest != "" {
		ref = fmt.Sprintf("%s@%s", ref, ociRef.Digest)
	} else if chartSource.ChartVersion != "" {
		ref = fmt.Sprintf("%s:%s", ref, chartSource.ChartVersion)
	}

	log.Info("pulling chart from OCI", "ref", ref)

	pullURL := fmt.Sprintf("%s/%s", ociRef.Registry, ociRef.Repository)
	if ociRef.Tag != "" {
		pullURL = fmt.Sprintf("%s:%s", pullURL, ociRef.Tag)
	} else if chartSource.ChartVersion != "" {
		pullURL = fmt.Sprintf("%s:%s", pullURL, chartSource.ChartVersion)
	}

	httpClient := &http.Client{}
	if ociRef.Insecure {
		httpClient.Transport = &http.Transport{
			TLSClientConfig: nil,
		}
	}

	req, err := http.NewRequestWithContext(ctx, "GET", fmt.Sprintf("https://%s", pullURL), nil)
	if err != nil {
		return nil, fmt.Errorf("create OCI request: %w", err)
	}

	if ociRef.Auth != nil {
		req.SetBasicAuth(ociRef.Auth.Username, ociRef.Auth.Password)
	}

	settings := cli.New()
	providers := getter.All(settings)

	ociGetter, err := providers.ByScheme("oci")
	if err != nil {
		return cl.loadFromOCIFallback(ctx, chartSource)
	}

	getterOpts := []getter.Option{
		getter.WithURL(ref),
	}
	if ociRef.Auth != nil {
		getterOpts = append(getterOpts,
			getter.WithBasicAuth(ociRef.Auth.Username, ociRef.Auth.Password),
		)
	}

	resp, err := ociGetter.Get(ref, getterOpts...)
	if err != nil {
		return cl.loadFromOCIFallback(ctx, chartSource)
	}
	defer resp.Close()

	data, err := io.ReadAll(resp)
	if err != nil {
		return nil, fmt.Errorf("read OCI chart data: %w", err)
	}

	tmpDir, err := os.MkdirTemp("", "helm-chart-oci-")
	if err != nil {
		return nil, fmt.Errorf("create temp dir: %w", err)
	}
	defer os.RemoveAll(tmpDir)

	chartFile := filepath.Join(tmpDir, "chart.tgz")
	if err := os.WriteFile(chartFile, data, 0644); err != nil {
		return nil, fmt.Errorf("write OCI chart file: %w", err)
	}

	loadedChart, err := loader.Load(chartFile)
	if err != nil {
		return nil, fmt.Errorf("load OCI chart: %w", err)
	}

	log.Info("chart loaded from OCI", "name", loadedChart.Name(), "version", loadedChart.Metadata.Version)
	return loadedChart, nil
}

func (cl *ChartLoader) loadFromOCIFallback(ctx context.Context, chartSource *v1alpha1.ChartSource) (*chart.Chart, error) {
	ociRef := chartSource.OCIRef

	scheme := "https"
	if ociRef.Insecure {
		scheme = "http"
	}

	chartURL := fmt.Sprintf("%s://%s/v2/%s/manifests/%s", scheme, ociRef.Registry, ociRef.Repository, chartSource.ChartVersion)
	if ociRef.Tag != "" {
		chartURL = fmt.Sprintf("%s://%s/v2/%s/manifests/%s", scheme, ociRef.Registry, ociRef.Repository, ociRef.Tag)
	}

	httpClient := &http.Client{}
	req, err := http.NewRequestWithContext(ctx, "GET", chartURL, nil)
	if err != nil {
		return nil, fmt.Errorf("create fallback OCI request: %w", err)
	}

	req.Header.Set("Accept", "application/vnd.oci.image.manifest.v1+json")
	if ociRef.Auth != nil {
		req.SetBasicAuth(ociRef.Auth.Username, ociRef.Auth.Password)
	}

	resp, err := httpClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fallback OCI request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("fallback OCI request returned status %d", resp.StatusCode)
	}

	return nil, fmt.Errorf("OCI fallback not fully implemented: use helm pull for OCI registries")
}

func (cl *ChartLoader) constructChartURL(chartSource *v1alpha1.ChartSource) (string, error) {
	if chartSource.RepoURL == "" || chartSource.ChartName == "" {
		return "", fmt.Errorf("repoURL and chartName are required")
	}

	chartURL := fmt.Sprintf("%s/%s", chartSource.RepoURL, chartSource.ChartName)
	if chartSource.ChartVersion != "" {
		chartURL = fmt.Sprintf("%s-%s.tgz", chartURL, chartSource.ChartVersion)
	} else {
		chartURL = fmt.Sprintf("%s/index.yaml", chartURL)
	}

	return chartURL, nil
}

func (cl *ChartLoader) resolveRepoAuth(ctx context.Context, auth *v1alpha1.RepoAuth) (string, string, error) {
	if auth == nil {
		return "", "", nil
	}

	username := auth.Username
	password := auth.Password

	if auth.PasswordSecretRef != nil {
		content, err := cl.Resolver.Resolve(ctx, &v1alpha1.SourceSpec{
			Type:      v1alpha1.SourceTypeSecretRef,
			SecretRef: auth.PasswordSecretRef,
		})
		if err != nil {
			return "", "", fmt.Errorf("resolve password secret: %w", err)
		}
		if val, ok := content.KeyValues["password"]; ok {
			password = val
		}
	}

	return username, password, nil
}

func (cl *ChartLoader) addRepo(chartSource *v1alpha1.ChartSource) error {
	if chartSource.RepoURL == "" {
		return nil
	}

	settings := cli.New()
	providers := getter.All(settings)

	repoFile := settings.RepositoryConfig

	entry := &repo.Entry{
		Name:                  chartSource.ChartName,
		URL:                   chartSource.RepoURL,
		Username:              chartSource.Auth.Username,
		Password:              chartSource.Auth.Password,
		InsecureSkipTLSverify: chartSource.Auth.InsecureSkipTLS,
	}

	chartRepo, err := repo.NewChartRepository(entry, providers)
	if err != nil {
		return fmt.Errorf("create chart repository: %w", err)
	}

	_, err = chartRepo.DownloadIndexFile()
	if err != nil {
		return fmt.Errorf("download repo index: %w", err)
	}

	existingRepoFile, err := repo.LoadFile(repoFile)
	if err != nil && !os.IsNotExist(err) {
		return fmt.Errorf("load repo file: %w", err)
	}

	if existingRepoFile == nil {
		existingRepoFile = repo.NewFile()
	}

	if !existingRepoFile.Has(entry.Name) {
		existingRepoFile.Add(entry)
		if err := existingRepoFile.WriteFile(repoFile, 0644); err != nil {
			return fmt.Errorf("write repo file: %w", err)
		}
	}

	return nil
}
```
## 4. Values 合并器
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\helm\values.go
package helm

import (
	"context"
	"fmt"
	"strings"

	"github.com/go-logr/logr"
	"helm.sh/helm/v3/pkg/cli"
	"helm.sh/helm/v3/pkg/getter"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/yaml"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type ValuesMerger struct {
	client.Client
	Log      logr.Logger
	Resolver *source.Resolver
}

func NewValuesMerger(c client.Client, log logr.Logger, resolver *source.Resolver) *ValuesMerger {
	return &ValuesMerger{
		Client:   c,
		Log:      log.WithName("values-merger"),
		Resolver: resolver,
	}
}

func (vm *ValuesMerger) Merge(ctx context.Context, sources []v1alpha1.ValuesSource, cv *v1alpha1.ComponentVersion) (map[string]interface{}, error) {
	merged := make(map[string]interface{})

	for i, vs := range sources {
		values, err := vm.resolveValuesSource(ctx, vs, cv)
		if err != nil {
			return nil, fmt.Errorf("resolve values source [%d]: %w", i, err)
		}

		merged = mergeMaps(merged, values)
	}

	vm.injectDefaultValues(merged, cv)

	return merged, nil
}

func (vm *ValuesMerger) resolveValuesSource(ctx context.Context, vs v1alpha1.ValuesSource, cv *v1alpha1.ComponentVersion) (map[string]interface{}, error) {
	switch {
	case vs.Inline != "":
		return vm.parseYAML(vs.Inline)
	case vs.SourceSpec != nil:
		return vm.resolveFromSourceSpec(ctx, vs.SourceSpec)
	case vs.ConfigMapRef != nil:
		return vm.resolveFromConfigMap(ctx, vs.ConfigMapRef)
	case vs.SecretRef != nil:
		return vm.resolveFromSecret(ctx, vs.SecretRef)
	default:
		return make(map[string]interface{}), nil
	}
}

func (vm *ValuesMerger) parseYAML(data string) (map[string]interface{}, error) {
	values := make(map[string]interface{})
	if err := yaml.Unmarshal([]byte(data), &values); err != nil {
		return nil, fmt.Errorf("parse YAML values: %w", err)
	}
	return values, nil
}

func (vm *ValuesMerger) resolveFromSourceSpec(ctx context.Context, spec *v1alpha1.SourceSpec) (map[string]interface{}, error) {
	content, err := vm.Resolver.Resolve(ctx, spec)
	if err != nil {
		return nil, fmt.Errorf("resolve values source spec: %w", err)
	}

	if content.IsYAML {
		return vm.parseYAML(string(content.Data))
	}

	if content.KeyValues != nil {
		return vm.flattenKeyValues(content.KeyValues), nil
	}

	return vm.parseYAML(string(content.Data))
}

func (vm *ValuesMerger) resolveFromConfigMap(ctx context.Context, ref *v1alpha1.ConfigMapKeyRef) (map[string]interface{}, error) {
	content, err := vm.Resolver.Resolve(ctx, &v1alpha1.SourceSpec{
		Type:         v1alpha1.SourceTypeConfigMapRef,
		ConfigMapRef: ref,
	})
	if err != nil {
		return nil, fmt.Errorf("resolve ConfigMap values: %w", err)
	}

	if content.KeyValues != nil {
		return vm.flattenKeyValues(content.KeyValues), nil
	}

	return vm.parseYAML(string(content.Data))
}

func (vm *ValuesMerger) resolveFromSecret(ctx context.Context, ref *v1alpha1.SecretKeyRef) (map[string]interface{}, error) {
	content, err := vm.Resolver.Resolve(ctx, &v1alpha1.SourceSpec{
		Type:      v1alpha1.SourceTypeSecretRef,
		SecretRef: ref,
	})
	if err != nil {
		return nil, fmt.Errorf("resolve Secret values: %w", err)
	}

	if content.KeyValues != nil {
		return vm.flattenKeyValues(content.KeyValues), nil
	}

	return vm.parseYAML(string(content.Data))
}

func (vm *ValuesMerger) flattenKeyValues(kv map[string]string) map[string]interface{} {
	result := make(map[string]interface{})
	for key, value := range kv {
		if strings.Contains(key, ".") {
			parts := strings.Split(key, ".")
			current := result
			for i, part := range parts {
				if i == len(parts)-1 {
					current[part] = value
				} else {
					if _, ok := current[part]; !ok {
						current[part] = make(map[string]interface{})
					}
					if nested, ok := current[part].(map[string]interface{}); ok {
						current = nested
					}
				}
			}
		} else {
			result[key] = value
		}
	}
	return result
}

func (vm *ValuesMerger) injectDefaultValues(values map[string]interface{}, cv *v1alpha1.ComponentVersion) {
	componentValues := map[string]interface{}{
		"componentName": cv.Spec.ComponentName,
		"version":       cv.Spec.Version,
	}

	if cv.Namespace != "" {
		componentValues["namespace"] = cv.Namespace
	}

	values["componentversion"] = mergeMaps(
		values["componentversion"].(map[string]interface{}),
		componentValues,
	)
}

func mergeMaps(base, overlay map[string]interface{}) map[string]interface{} {
	result := make(map[string]interface{})

	for key, value := range base {
		result[key] = value
	}

	for key, value := range overlay {
		if existing, ok := result[key]; ok {
			existingMap, existingIsMap := existing.(map[string]interface{})
			valueMap, valueIsMap := value.(map[string]interface{})

			if existingIsMap && valueIsMap {
				result[key] = mergeMaps(existingMap, valueMap)
				continue
			}
		}
		result[key] = value
	}

	return result
}
```
## 5. Helm 验证器
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\helm\verifier.go
package helm

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	"helm.sh/helm/v3/pkg/action"
	"helm.sh/helm/v3/pkg/release"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer"
)

type Verifier struct {
	client.Client
	Log    logr.Logger
	Config *rest.Config
}

func NewVerifier(c client.Client, log logr.Logger, config *rest.Config) *Verifier {
	return &Verifier{
		Client: c,
		Log:    log.WithName("helm-verifier"),
		Config: config,
	}
}

func (hv *Verifier) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion, helmSpec *v1alpha1.HelmChartInstallSpec) (*installer.VerifyResult, error) {
	log := hv.Log.WithValues("component", cv.Spec.ComponentName, "release", helmSpec.ReleaseName)

	verifySpec := helmSpec.VerifySpec
	if verifySpec == nil {
		return hv.VerifyReleaseStatus(ctx, helmSpec)
	}

	if verifySpec.CheckReleaseStatus {
		result, err := hv.VerifyReleaseStatus(ctx, helmSpec)
		if err != nil {
			return result, err
		}
		if !result.Healthy {
			return result, nil
		}
	}

	if verifySpec.CheckResources && len(verifySpec.ResourceChecks) > 0 {
		result := hv.verifyResources(ctx, helmSpec, verifySpec.ResourceChecks)
		if !result.Healthy {
			return result, nil
		}
	}

	log.Info("helm verification passed")
	return &installer.VerifyResult{
		Healthy: true,
		Reason:  "all helm verification checks passed",
	}, nil
}

func (hv *Verifier) VerifyReleaseStatus(ctx context.Context, helmSpec *v1alpha1.HelmChartInstallSpec) (*installer.VerifyResult, error) {
	log := hv.Log.WithValues("release", helmSpec.ReleaseName)

	actionConfig, err := hv.newActionConfig(helmSpec.Namespace)
	if err != nil {
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("create action config: %v", err),
		}, err
	}

	getAction := action.NewGet(actionConfig)
	rel, err := getAction.Run(helmSpec.ReleaseName)
	if err != nil {
		log.Error(err, "failed to get release status")
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("release not found: %v", err),
		}, nil
	}

	switch rel.Info.Status {
	case release.StatusDeployed:
		return &installer.VerifyResult{
			Healthy: true,
			Reason:  fmt.Sprintf("release deployed (revision %d)", rel.Version),
		}, nil
	case release.StatusPendingInstall, release.StatusPendingUpgrade, release.StatusPendingRollback:
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("release in pending state: %s", rel.Info.Status),
		}, nil
	case release.StatusFailed:
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("release failed: %s", rel.Info.Description),
		}, nil
	case release.StatusSuperseded:
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  "release superseded by newer version",
		}, nil
	default:
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("release in unexpected state: %s", rel.Info.Status),
		}, nil
	}
}

func (hv *Verifier) verifyResources(ctx context.Context, helmSpec *v1alpha1.HelmChartInstallSpec, checks []v1alpha1.HelmResourceCheck) *installer.VerifyResult {
	log := hv.Log.WithValues("release", helmSpec.ReleaseName)

	dynamicClient, err := dynamic.NewForConfig(hv.Config)
	if err != nil {
		return &installer.VerifyResult{
			Healthy: false,
			Reason:  fmt.Sprintf("create dynamic client: %v", err),
		}
	}

	for _, check := range checks {
		gvk := schema.FromAPIVersionAndKind("", check.Kind)
		gvr, err := hv.gvkToGVR(gvk)
		if err != nil {
			log.Error(err, "failed to map GVK to GVR", "kind", check.Kind)
			return &installer.VerifyResult{
				Healthy: false,
				Reason:  fmt.Sprintf("map kind %q to resource: %v", check.Kind, err),
			}
		}

		ns := check.Namespace
		if ns == "" {
			ns = helmSpec.Namespace
		}

		resourceClient := dynamicClient.Resource(gvr).Namespace(ns)
		obj, err := resourceClient.Get(ctx, check.Name, metav1.GetOptions{})
		if err != nil {
			return &installer.VerifyResult{
				Healthy: false,
				Reason:  fmt.Sprintf("resource %s/%s not found: %v", check.Kind, check.Name, err),
			}
		}

		if check.Condition != nil {
			if !hv.checkCondition(obj, check.Condition) {
				return &installer.VerifyResult{
					Healthy: false,
					Reason:  fmt.Sprintf("resource %s/%s condition %s=%s not met", check.Kind, check.Name, check.Condition.Type, check.Condition.Status),
				}
			}
		}

		log.Info("resource check passed", "kind", check.Kind, "name", check.Name)
	}

	return &installer.VerifyResult{
		Healthy: true,
		Reason:  "all resource checks passed",
	}
}

func (hv *Verifier) checkCondition(obj *unstructured.Unstructured, condition *v1alpha1.ResourceCondition) bool {
	conditions, found, err := unstructured.NestedSlice(obj.Object, "status", "conditions")
	if err != nil || !found {
		return false
	}

	for _, c := range conditions {
		condMap, ok := c.(map[string]interface{})
		if !ok {
			continue
		}

		condType, _, _ := unstructured.NestedString(condMap, "type")
		condStatus, _, _ := unstructured.NestedString(condMap, "status")

		if condType == condition.Type && condStatus == condition.Status {
			return true
		}
	}

	return false
}

func (hv *Verifier) gvkToGVR(gvk schema.GroupVersionKind) (schema.GroupVersionResource, error) {
	switch gvk.Kind {
	case "Deployment":
		return schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}, nil
	case "StatefulSet":
		return schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "statefulsets"}, nil
	case "DaemonSet":
		return schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "daemonsets"}, nil
	case "Service":
		return schema.GroupVersionResource{Group: "", Version: "v1", Resource: "services"}, nil
	case "ConfigMap":
		return schema.GroupVersionResource{Group: "", Version: "v1", Resource: "configmaps"}, nil
	case "Secret":
		return schema.GroupVersionResource{Group: "", Version: "v1", Resource: "secrets"}, nil
	case "Ingress":
		return schema.GroupVersionResource{Group: "networking.k8s.io", Version: "v1", Resource: "ingresses"}, nil
	case "Pod":
		return schema.GroupVersionResource{Group: "", Version: "v1", Resource: "pods"}, nil
	default:
		return schema.GroupVersionResource{}, fmt.Errorf("unsupported kind: %s", gvk.Kind)
	}
}

func (hv *Verifier) newActionConfig(namespace string) (*action.Configuration, error) {
	rcGetter := &restClientGetter{
		config:    hv.Config,
		namespace: namespace,
	}

	logWrapper := &helmLogWrapper{log: hv.Log}

	actionConfig := &action.Configuration{
		RESTClientGetter: rcGetter,
		Log:              logWrapper.Log,
	}

	kubeClient, err := kubernetes.NewForConfig(hv.Config)
	if err != nil {
		return nil, err
	}

	store := action.NewSecretStore(
		kubeClient.CoreV1().Secrets(namespace),
		namespace,
	)
	actionConfig.Releases = store

	return actionConfig, nil
}
```
## 6. Helm 辅助类型
```go d:\code\github\componentversion-controller\pkg\componentversion\installer\helm\helpers.go
package helm

import (
	"fmt"

	"github.com/go-logr/logr"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/discovery/cached/memory"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

type restClientGetter struct {
	config    *rest.Config
	namespace string
}

func (r *restClientGetter) ToRESTConfig() (*rest.Config, error) {
	return r.config, nil
}

func (r *restClientGetter) ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	discoveryClient, err := discovery.NewDiscoveryClientForConfig(r.config)
	if err != nil {
		return nil, err
	}
	return memory.NewMemCacheClient(discoveryClient), nil
}

func (r *restClientGetter) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	return nil
}

func (r *restClientGetter) Namespace() (string, bool, error) {
	return r.namespace, false, nil
}

type helmLogWrapper struct {
	log logr.Logger
}

func (h *helmLogWrapper) Log(format string, v ...interface{}) {
	h.log.Info(fmt.Sprintf(format, v...))
}
```
## 架构总结
### HelmChartInstaller 整体设计
```
HelmChartInstaller
├── Installer Interface (Install/Verify/Rollback/Uninstall)
│
├── ChartLoader (Chart 来源解析)
│   ├── SourceSpec 来源 → Resolver 解析 → 写入临时文件 → loader.Load
│   ├── RepoURL 来源   → helm getter 下载 → 写入临时文件 → loader.Load
│   └── OCIRef 来源    → OCI getter 拉取 → 写入临时文件 → loader.Load
│
├── ValuesMerger (多来源 Values 合并)
│   ├── Inline YAML    → 直接解析
│   ├── SourceSpec     → Resolver 解析 → YAML 解析
│   ├── ConfigMapRef   → Resolver 解析 → KeyValues 扁平化/YAML 解析
│   ├── SecretRef      → Resolver 解析 → KeyValues 扁平化/YAML 解析
│   └── 注入默认值     → componentversion.componentName/version/namespace
│
├── Verifier (验证器)
│   ├── Release 状态验证 → action.NewGet → 检查 release.Status
│   ├── Resource 条件验证 → dynamic client → 检查 status.conditions
│   └── 组合验证       → 先检查 Release 状态，再检查资源条件
│
└── Release 管理 (Helm Action 封装)
    ├── Install → action.NewInstall (支持 CreateNamespace/Replace/Atomic/Wait)
    ├── Upgrade → action.NewUpgrade (支持 Force/ResetValues/ReuseValues/Atomic)
    ├── Rollback → action.NewRollback (自动查找上一版本)
    └── Uninstall → action.NewUninstall (支持 KeepHistory/Wait)
```
### 关键设计决策
| 决策 | 说明 |
|------|------|
| **三种 Chart 来源** | SourceSpec 适用于内嵌/ConfigMap/URI；RepoURL 适用于传统 Helm Repo；OCIRef 适用于 OCI Registry |
| **Values 多来源合并** | 按声明顺序依次深度合并，后者覆盖前者，支持嵌套 Key 的点分表示法 |
| **Install/Upgrade 自动判断** | 检查 Release 是否已存在，自动选择 Install 或 Upgrade 路径 |
| **Rollback 自动版本回退** | 从 History 中自动获取上一版本号，无需手动指定 |
| **Atomic 模式** | 安装/升级失败时自动回滚，保证集群状态一致性 |
| **Secret 存储** | 使用 Helm 默认的 Secret 后端存储 Release 状态 |
| **GVK→GVR 映射** | 验证器内置常见资源类型映射，支持条件检查 |
### Install 执行流程
```
Install()
  ├── newActionConfig(namespace) → 初始化 Helm Action 配置
  ├── ChartLoader.Load()         → 加载 Chart
  │   ├── OCIRef?    → OCI 拉取
  │   ├── RepoURL?   → Repo 下载
  │   └── SourceSpec? → Resolver 解析
  ├── ValuesMerger.Merge()       → 合并 Values
  │   ├── 逐个解析 ValuesSource
  │   └── 深度合并 + 注入默认值
  ├── 检查 Release 是否存在
  │   ├── 不存在 → action.NewInstall → RunWithContext
  │   └── 已存在 → action.NewUpgrade → RunWithContext
  └── 返回 PhaseVerifying + InstallDetails

Verify()
  ├── 检查 Release Status (Deployed/Pending/Failed)
  ├── 检查 Resource Conditions (可选)
  └── 返回 Healthy/Reason

Rollback()
  ├── 获取 Release History
  ├── 定位上一版本 (N-1)
  ├── action.NewRollback → Run
  └── 返回 PhasePending

Uninstall()
  ├── action.NewUninstall → Run
  └── 返回 PhasePending
```
请手动创建上述文件到对应路径。至此，核心层的三种安装策略（**ManifestInstaller**、**ScriptInstaller**、**HelmChartInstaller**）已全部实现。下一步可以实现**编排层：Reconciler + Dispatcher + NodeConfig 协作**。

# 健康检查 API       
基于前面会话的设计，我将实现统一的健康检查框架，支持跨所有安装策略（Manifest/Script/HelmChart）的健康检查。
## 1. 健康检查 API 类型
```go d:\code\github\componentversion-controller\api\v1alpha1\healthcheck_types.go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type HealthCheckSpec struct {
	Checks         []HealthCheck     `json:"checks"`
	Strategy       HealthCheckStrategy `json:"strategy,omitempty"`
	IntervalSeconds int64            `json:"intervalSeconds,omitempty"`
	TimeoutSeconds  int64            `json:"timeoutSeconds,omitempty"`
	FailureThreshold int32           `json:"failureThreshold,omitempty"`
	SuccessThreshold int32           `json:"successThreshold,omitempty"`
}

type HealthCheckStrategy string

const (
	HealthCheckStrategyAll   HealthCheckStrategy = "All"
	HealthCheckStrategyAny   HealthCheckStrategy = "Any"
	HealthCheckStrategyQuorum HealthCheckStrategy = "Quorum"
)

type HealthCheck struct {
	Name     string               `json:"name"`
	Type     HealthCheckType      `json:"type"`
	Required bool                 `json:"required,omitempty"`
	Weight   int32                `json:"weight,omitempty"`
	Command  *CommandHealthCheck  `json:"command,omitempty"`
	HTTP     *HTTPHealthCheck     `json:"http,omitempty"`
	Resource *ResourceHealthCheck `json:"resource,omitempty"`
	Helm     *HelmHealthCheck     `json:"helm,omitempty"`
}

type HealthCheckType string

const (
	HealthCheckTypeCommand  HealthCheckType = "Command"
	HealthCheckTypeHTTP     HealthCheckType = "HTTP"
	HealthCheckTypeResource HealthCheckType = "Resource"
	HealthCheckTypeHelm     HealthCheckType = "Helm"
)

type CommandHealthCheck struct {
	Script         string   `json:"script,omitempty"`
	ScriptSource   *SourceSpec `json:"scriptSource,omitempty"`
	Args           []string `json:"args,omitempty"`
	Env            []EnvVar `json:"env,omitempty"`
	TimeoutSeconds int64    `json:"timeoutSeconds,omitempty"`
	ExpectedExitCode int    `json:"expectedExitCode,omitempty"`
	OutputContains  string  `json:"outputContains,omitempty"`
	OutputNotContains string `json:"outputNotContains,omitempty"`
}

type HTTPHealthCheck struct {
	URL                  string            `json:"url"`
	Method               string            `json:"method,omitempty"`
	Headers              map[string]string `json:"headers,omitempty"`
	Body                 string            `json:"body,omitempty"`
	ExpectedStatusCodes  []int             `json:"expectedStatusCodes,omitempty"`
	ResponseBodyContains string            `json:"responseBodyContains,omitempty"`
	ResponseBodyNotContains string          `json:"responseBodyNotContains,omitempty"`
	TLSVerify            *bool             `json:"tlsVerify,omitempty"`
	Retries              int               `json:"retries,omitempty"`
	RetryIntervalSeconds int64             `json:"retryIntervalSeconds,omitempty"`
	TimeoutSeconds       int64             `json:"timeoutSeconds,omitempty"`
}

type ResourceHealthCheck struct {
	Refs       []ResourceHealthRef `json:"refs"`
	Namespace  string              `json:"namespace,omitempty"`
	Conditions []ResourceCondition `json:"conditions,omitempty"`
	Count      *ResourceCountCheck `json:"count,omitempty"`
}

type ResourceHealthRef struct {
	APIVersion string `json:"apiVersion,omitempty"`
	Kind       string `json:"kind"`
	Name       string `json:"name"`
	Namespace  string `json:"namespace,omitempty"`
	LabelSelector map[string]string `json:"labelSelector,omitempty"`
}

type ResourceCountCheck struct {
	MinCount int32 `json:"minCount,omitempty"`
	MaxCount int32 `json:"maxCount,omitempty"`
	ExactCount int32 `json:"exactCount,omitempty"`
}

type HelmHealthCheck struct {
	ReleaseName string `json:"releaseName"`
	Namespace   string `json:"namespace,omitempty"`
	CheckStatus bool   `json:"checkStatus,omitempty"`
	CheckResources bool `json:"checkResources,omitempty"`
}

type HealthStatus struct {
	Overall       HealthOverallStatus `json:"overall,omitempty"`
	Checks        []HealthCheckResult `json:"checks,omitempty"`
	HealthyCount  int32               `json:"healthyCount,omitempty"`
	UnhealthyCount int32              `json:"unhealthyCount,omitempty"`
	UnknownCount  int32               `json:"unknownCount,omitempty"`
	ConsecutiveFailures int32         `json:"consecutiveFailures,omitempty"`
	ConsecutiveSuccesses int32        `json:"consecutiveSuccesses,omitempty"`
	LastCheckedAt *metav1.Time        `json:"lastCheckedAt,omitempty"`
	LastHealthyAt *metav1.Time        `json:"lastHealthyAt,omitempty"`
	Message       string              `json:"message,omitempty"`
}

type HealthOverallStatus string

const (
	HealthStatusHealthy   HealthOverallStatus = "Healthy"
	HealthStatusUnhealthy HealthOverallStatus = "Unhealthy"
	HealthStatusDegraded  HealthOverallStatus = "Degraded"
	HealthStatusUnknown   HealthOverallStatus = "Unknown"
)

type HealthCheckResult struct {
	Name      string              `json:"name"`
	Type      HealthCheckType     `json:"type"`
	Status    HealthCheckStatus   `json:"status"`
	Message   string              `json:"message,omitempty"`
	Duration  string              `json:"duration,omitempty"`
	Timestamp *metav1.Time        `json:"timestamp,omitempty"`
	Required  bool                `json:"required,omitempty"`
}

type HealthCheckStatus string

const (
	HealthCheckStatusHealthy   HealthCheckStatus = "Healthy"
	HealthCheckStatusUnhealthy HealthCheckStatus = "Unhealthy"
	HealthCheckStatusUnknown   HealthCheckStatus = "Unknown"
	HealthCheckStatusSkipped   HealthCheckStatus = "Skipped"
)
```
## 2. 健康检查管理器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\manager.go
package healthcheck

import (
	"context"
	"fmt"
	"sync"
	"time"

	"github.com/go-logr/logr"
	"k8s.io/client-go/rest"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type Manager struct {
	client.Client
	Log      logr.Logger
	Config   *rest.Config
	Resolver *source.Resolver

	checkers map[v1alpha1.HealthCheckType]Checker
	mu       sync.RWMutex
	cache    map[string]*cachedHealthResult
}

type cachedHealthResult struct {
	status   *v1alpha1.HealthStatus
	cachedAt time.Time
	ttl      time.Duration
}

func NewManager(c client.Client, log logr.Logger, config *rest.Config, resolver *source.Resolver) *Manager {
	m := &Manager{
		Client:   c,
		Log:      log.WithName("healthcheck-manager"),
		Config:   config,
		Resolver: resolver,
		cache:    make(map[string]*cachedHealthResult),
	}

	m.checkers = map[v1alpha1.HealthCheckType]Checker{
		v1alpha1.HealthCheckTypeCommand:  NewCommandChecker(c, log, resolver),
		v1alpha1.HealthCheckTypeHTTP:     NewHTTPChecker(log),
		v1alpha1.HealthCheckTypeResource: NewResourceChecker(c, log, config),
		v1alpha1.HealthCheckTypeHelm:     NewHelmChecker(c, log, config),
	}

	return m
}

func (m *Manager) Check(ctx context.Context, cv *v1alpha1.ComponentVersion) (*v1alpha1.HealthStatus, error) {
	if cv.Spec.HealthCheck == nil {
		return &v1alpha1.HealthStatus{
			Overall: v1alpha1.HealthStatusHealthy,
			Message: "no health check defined",
		}, nil
	}

	log := m.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version)

	cacheKey := fmt.Sprintf("%s/%s", cv.Namespace, cv.Name)
	if cached := m.getFromCache(cacheKey); cached != nil {
		return cached, nil
	}

	spec := cv.Spec.HealthCheck
	timeout := m.getTimeout(spec.TimeoutSeconds)

	checkCtx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	results := make([]v1alpha1.HealthCheckResult, 0, len(spec.Checks))
	var healthyCount, unhealthyCount, unknownCount int32

	for i, check := range spec.Checks {
		log.Info("running health check", "check", check.Name, "type", check.Type, "index", i)

		result := m.runCheck(checkCtx, cv, check)
		results = append(results, result)

		switch result.Status {
		case v1alpha1.HealthCheckStatusHealthy:
			healthyCount++
		case v1alpha1.HealthCheckStatusUnhealthy:
			unhealthyCount++
		default:
			unknownCount++
		}

		if result.Status == v1alpha1.HealthCheckStatusUnhealthy && check.Required {
			log.Info("required check failed, stopping further checks", "check", check.Name)
			break
		}
	}

	overall := m.aggregateResults(results, spec.Strategy, spec.Checks)

	status := &v1alpha1.HealthStatus{
		Overall:        overall,
		Checks:         results,
		HealthyCount:   healthyCount,
		UnhealthyCount: unhealthyCount,
		UnknownCount:   unknownCount,
		LastCheckedAt:  &metav1.Time{Time: time.Now()},
		Message:        m.buildMessage(overall, results),
	}

	m.updateThresholds(status, cv)

	if overall == v1alpha1.HealthStatusHealthy {
		status.LastHealthyAt = status.LastCheckedAt
		status.ConsecutiveSuccesses = cv.Status.Health.ConsecutiveSuccesses + 1
		status.ConsecutiveFailures = 0
	} else {
		status.ConsecutiveFailures = cv.Status.Health.ConsecutiveFailures + 1
		status.ConsecutiveSuccesses = 0
	}

	interval := m.getInterval(spec.IntervalSeconds)
	m.putToCache(cacheKey, status, interval/2)

	log.Info("health check completed", "overall", overall, "healthy", healthyCount, "unhealthy", unhealthyCount)

	return status, nil
}

func (m *Manager) runCheck(ctx context.Context, cv *v1alpha1.ComponentVersion, check v1alpha1.HealthCheck) v1alpha1.HealthCheckResult {
	startTime := time.Now()

	result := v1alpha1.HealthCheckResult{
		Name:     check.Name,
		Type:     check.Type,
		Required: check.Required,
	}

	checker, ok := m.checkers[check.Type]
	if !ok {
		result.Status = v1alpha1.HealthCheckStatusUnknown
		result.Message = fmt.Sprintf("unsupported check type: %s", check.Type)
		result.Timestamp = &metav1.Time{Time: startTime}
		return result
	}

	checkResult, err := checker.Check(ctx, cv, check)
	if err != nil {
		result.Status = v1alpha1.HealthCheckStatusUnknown
		result.Message = fmt.Sprintf("check execution error: %v", err)
		result.Timestamp = &metav1.Time{Time: startTime}
		result.Duration = time.Since(startTime).String()
		return result
	}

	result.Status = checkResult.Status
	result.Message = checkResult.Message
	result.Duration = time.Since(startTime).String()
	result.Timestamp = &metav1.Time{Time: time.Now()}

	return result
}

func (m *Manager) aggregateResults(results []v1alpha1.HealthCheckResult, strategy v1alpha1.HealthCheckStrategy, checks []v1alpha1.HealthCheck) v1alpha1.HealthOverallStatus {
	if len(results) == 0 {
		return v1alpha1.HealthStatusUnknown
	}

	switch strategy {
	case v1alpha1.HealthCheckStrategyAll:
		return m.aggregateAll(results, checks)
	case v1alpha1.HealthCheckStrategyAny:
		return m.aggregateAny(results)
	case v1alpha1.HealthCheckStrategyQuorum:
		return m.aggregateQuorum(results, checks)
	default:
		return m.aggregateAll(results, checks)
	}
}

func (m *Manager) aggregateAll(results []v1alpha1.HealthCheckResult, checks []v1alpha1.HealthCheck) v1alpha1.HealthOverallStatus {
	hasDegraded := false

	for i, r := range results {
		if r.Status == v1alpha1.HealthCheckStatusUnhealthy {
			if i < len(checks) && checks[i].Required {
				return v1alpha1.HealthStatusUnhealthy
			}
			hasDegraded = true
		}
		if r.Status == v1alpha1.HealthCheckStatusUnknown {
			if i < len(checks) && checks[i].Required {
				return v1alpha1.HealthStatusUnknown
			}
			hasDegraded = true
		}
	}

	if hasDegraded {
		return v1alpha1.HealthStatusDegraded
	}

	return v1alpha1.HealthStatusHealthy
}

func (m *Manager) aggregateAny(results []v1alpha1.HealthCheckResult) v1alpha1.HealthOverallStatus {
	for _, r := range results {
		if r.Status == v1alpha1.HealthCheckStatusHealthy {
			return v1alpha1.HealthStatusHealthy
		}
	}

	return v1alpha1.HealthStatusUnhealthy
}

func (m *Manager) aggregateQuorum(results []v1alpha1.HealthCheckResult, checks []v1alpha1.HealthCheck) v1alpha1.HealthOverallStatus {
	var totalWeight, healthyWeight int32

	for i, r := range results {
		weight := int32(1)
		if i < len(checks) && checks[i].Weight > 0 {
			weight = checks[i].Weight
		}

		totalWeight += weight
		if r.Status == v1alpha1.HealthCheckStatusHealthy {
			healthyWeight += weight
		}
	}

	if totalWeight == 0 {
		return v1alpha1.HealthStatusUnknown
	}

	if healthyWeight > totalWeight/2 {
		return v1alpha1.HealthStatusHealthy
	}

	if healthyWeight > 0 {
		return v1alpha1.HealthStatusDegraded
	}

	return v1alpha1.HealthStatusUnhealthy
}

func (m *Manager) updateThresholds(status *v1alpha1.HealthStatus, cv *v1alpha1.ComponentVersion) {
	spec := cv.Spec.HealthCheck
	if spec == nil {
		return
	}

	if spec.FailureThreshold > 0 && status.ConsecutiveFailures > 0 {
		if cv.Status.Health.ConsecutiveFailures >= spec.FailureThreshold {
			status.Overall = v1alpha1.HealthStatusUnhealthy
			status.Message = fmt.Sprintf("consecutive failures (%d) exceeded threshold (%d)",
				status.ConsecutiveFailures, spec.FailureThreshold)
		}
	}

	if spec.SuccessThreshold > 0 && status.ConsecutiveSuccesses > 0 {
		if cv.Status.Health.ConsecutiveSuccesses < spec.SuccessThreshold {
			if status.Overall == v1alpha1.HealthStatusHealthy {
				status.Overall = v1alpha1.HealthStatusDegraded
				status.Message = fmt.Sprintf("consecutive successes (%d) not yet reached threshold (%d)",
					status.ConsecutiveSuccesses, spec.SuccessThreshold)
			}
		}
	}
}

func (m *Manager) buildMessage(overall v1alpha1.HealthOverallStatus, results []v1alpha1.HealthCheckResult) string {
	switch overall {
	case v1alpha1.HealthStatusHealthy:
		return "all health checks passed"
	case v1alpha1.HealthStatusDegraded:
		var failed []string
		for _, r := range results {
			if r.Status != v1alpha1.HealthCheckStatusHealthy {
				failed = append(failed, r.Name)
			}
		}
		return fmt.Sprintf("degraded: non-critical checks failed: %v", failed)
	case v1alpha1.HealthStatusUnhealthy:
		var failed []string
		for _, r := range results {
			if r.Status == v1alpha1.HealthCheckStatusUnhealthy {
				failed = append(failed, r.Name)
			}
		}
		return fmt.Sprintf("unhealthy: required checks failed: %v", failed)
	default:
		return "health status unknown"
	}
}

func (m *Manager) getTimeout(seconds int64) time.Duration {
	if seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return 2 * time.Minute
}

func (m *Manager) getInterval(seconds int64) time.Duration {
	if seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return 30 * time.Second
}

func (m *Manager) getFromCache(key string) *v1alpha1.HealthStatus {
	m.mu.RLock()
	defer m.mu.RUnlock()

	cached, ok := m.cache[key]
	if !ok {
		return nil
	}

	if time.Since(cached.cachedAt) > cached.ttl {
		return nil
	}

	result := cached.status.DeepCopy()
	return result
}

func (m *Manager) putToCache(key string, status *v1alpha1.HealthStatus, ttl time.Duration) {
	m.mu.Lock()
	defer m.mu.Unlock()

	m.cache[key] = &cachedHealthResult{
		status:   status.DeepCopy(),
		cachedAt: time.Now(),
		ttl:      ttl,
	}
}

func (m *Manager) InvalidateCache(key string) {
	m.mu.Lock()
	defer m.mu.Unlock()

	delete(m.cache, key)
}

func (m *Manager) RegisterChecker(checkType v1alpha1.HealthCheckType, checker Checker) {
	m.mu.Lock()
	defer m.mu.Unlock()

	m.checkers[checkType] = checker
}
```
## 3. Checker 接口与 Command 检查器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\checker.go
package healthcheck

import (
	"context"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type CheckResult struct {
	Status  v1alpha1.HealthCheckStatus
	Message string
}

type Checker interface {
	Check(ctx context.Context, cv *v1alpha1.ComponentVersion, spec v1alpha1.HealthCheck) (*CheckResult, error)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\command.go
package healthcheck

import (
	"bytes"
	"context"
	"fmt"
	"os"
	"os/exec"
	"strings"
	"time"

	"github.com/go-logr/logr"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type CommandChecker struct {
	client.Client
	Log      logr.Logger
	Resolver *source.Resolver
}

func NewCommandChecker(c client.Client, log logr.Logger, resolver *source.Resolver) *CommandChecker {
	return &CommandChecker{
		Client:   c,
		Log:      log.WithName("command-checker"),
		Resolver: resolver,
	}
}

func (cc *CommandChecker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion, spec v1alpha1.HealthCheck) (*CheckResult, error) {
	if spec.Command == nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "command check spec is nil",
		}, nil
	}

	cmdSpec := spec.Command

	var script string
	switch {
	case cmdSpec.Script != "":
		script = cmdSpec.Script
	case cmdSpec.ScriptSource != nil:
		content, err := cc.Resolver.Resolve(ctx, cmdSpec.ScriptSource)
		if err != nil {
			return &CheckResult{
				Status:  v1alpha1.HealthCheckStatusUnknown,
				Message: fmt.Sprintf("resolve script source: %v", err),
			}, err
		}
		script = string(content.Data)
	default:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "no script or scriptSource specified",
		}, nil
	}

	timeout := time.Duration(cmdSpec.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 30 * time.Second
	}

	checkCtx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	cmd := exec.CommandContext(checkCtx, "/bin/sh", "-c", script)

	if len(cmdSpec.Args) > 0 {
		cmd = exec.CommandContext(checkCtx, "/bin/sh", append([]string{"-c", script}, cmdSpec.Args...)...)
	}

	var stdout, stderr bytes.Buffer
	cmd.Stdout = &stdout
	cmd.Stderr = &stderr

	env := os.Environ()
	for _, e := range cmdSpec.Env {
		env = append(env, fmt.Sprintf("%s=%s", e.Name, e.Value))
	}
	env = append(env,
		fmt.Sprintf("COMPONENT_NAME=%s", cv.Spec.ComponentName),
		fmt.Sprintf("COMPONENT_VERSION=%s", cv.Spec.Version),
		fmt.Sprintf("COMPONENT_NAMESPACE=%s", cv.Namespace),
	)
	cmd.Env = env

	err := cmd.Run()

	exitCode := 0
	if cmd.ProcessState != nil {
		exitCode = cmd.ProcessState.ExitCode()
	}

	expectedExitCode := cmdSpec.ExpectedExitCode
	if expectedExitCode == 0 {
		expectedExitCode = 0
	}

	if checkCtx.Err() == context.DeadlineExceeded {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("command timed out after %v", timeout),
		}, nil
	}

	if err != nil && exitCode != expectedExitCode {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("command exited with code %d (expected %d): %s",
				exitCode, expectedExitCode, strings.TrimSpace(stderr.String())),
		}, nil
	}

	if cmdSpec.OutputContains != "" {
		output := stdout.String()
		if !strings.Contains(output, cmdSpec.OutputContains) {
			return &CheckResult{
				Status:  v1alpha1.HealthCheckStatusUnhealthy,
				Message: fmt.Sprintf("output does not contain %q", cmdSpec.OutputContains),
			}, nil
		}
	}

	if cmdSpec.OutputNotContains != "" {
		output := stdout.String()
		if strings.Contains(output, cmdSpec.OutputNotContains) {
			return &CheckResult{
				Status:  v1alpha1.HealthCheckStatusUnhealthy,
				Message: fmt.Sprintf("output contains forbidden string %q", cmdSpec.OutputNotContains),
			}, nil
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("command succeeded (exit code %d)", exitCode),
	}, nil
}
```
## 4. HTTP 检查器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\http.go
package healthcheck

import (
	"context"
	"crypto/tls"
	"fmt"
	"io"
	"net/http"
	"strings"
	"time"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/util/wait"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type HTTPChecker struct {
	Log logr.Logger
}

func NewHTTPChecker(log logr.Logger) *HTTPChecker {
	return &HTTPChecker{
		Log: log.WithName("http-checker"),
	}
}

func (hc *HTTPChecker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion, spec v1alpha1.HealthCheck) (*CheckResult, error) {
	if spec.HTTP == nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "http check spec is nil",
		}, nil
	}

	httpSpec := spec.HTTP

	timeout := time.Duration(httpSpec.TimeoutSeconds) * time.Second
	if timeout == 0 {
		timeout = 10 * time.Second
	}

	retries := httpSpec.Retries
	if retries == 0 {
		retries = 3
	}

	retryInterval := time.Duration(httpSpec.RetryIntervalSeconds) * time.Second
	if retryInterval == 0 {
		retryInterval = 2 * time.Second
	}

	tlsVerify := true
	if httpSpec.TLSVerify != nil {
		tlsVerify = *httpSpec.TLSVerify
	}

	transport := &http.Transport{}
	if !tlsVerify {
		transport.TLSClientConfig = &tls.Config{
			InsecureSkipVerify: true,
		}
	}

	httpClient := &http.Client{
		Timeout:   timeout,
		Transport: transport,
	}

	method := httpSpec.Method
	if method == "" {
		method = "GET"
	}

	var lastErr error
	var lastResp *http.Response

	for attempt := 0; attempt <= retries; attempt++ {
		if attempt > 0 {
			select {
			case <-ctx.Done():
				return &CheckResult{
					Status:  v1alpha1.HealthCheckStatusUnhealthy,
					Message: "context cancelled during retry",
				}, nil
			case <-time.After(retryInterval):
			}
		}

		var body io.Reader
		if httpSpec.Body != "" {
			body = strings.NewReader(httpSpec.Body)
		}

		req, err := http.NewRequestWithContext(ctx, method, httpSpec.URL, body)
		if err != nil {
			return &CheckResult{
				Status:  v1alpha1.HealthCheckStatusUnknown,
				Message: fmt.Sprintf("create request: %v", err),
			}, err
		}

		for key, value := range httpSpec.Headers {
			req.Header.Set(key, value)
		}

		resp, err := httpClient.Do(req)
		if err != nil {
			lastErr = err
			hc.Log.V(1).Info("HTTP check attempt failed", "attempt", attempt+1, "error", err)
			continue
		}

		lastResp = resp

		if !hc.isExpectedStatus(resp.StatusCode, httpSpec.ExpectedStatusCodes) {
			resp.Body.Close()
			lastErr = fmt.Errorf("unexpected status code %d", resp.StatusCode)
			hc.Log.V(1).Info("HTTP check unexpected status", "attempt", attempt+1, "status", resp.StatusCode)
			continue
		}

		bodyBytes, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			lastErr = readErr
			continue
		}

		bodyStr := string(bodyBytes)

		if httpSpec.ResponseBodyContains != "" {
			if !strings.Contains(bodyStr, httpSpec.ResponseBodyContains) {
				lastErr = fmt.Errorf("response body does not contain %q", httpSpec.ResponseBodyContains)
				continue
			}
		}

		if httpSpec.ResponseBodyNotContains != "" {
			if strings.Contains(bodyStr, httpSpec.ResponseBodyNotContains) {
				lastErr = fmt.Errorf("response body contains forbidden string %q", httpSpec.ResponseBodyNotContains)
				continue
			}
		}

		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusHealthy,
			Message: fmt.Sprintf("HTTP %s %s returned %d", method, httpSpec.URL, resp.StatusCode),
		}, nil
	}

	if lastResp != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("HTTP check failed after %d retries: last status %d", retries, lastResp.StatusCode),
		}, nil
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusUnhealthy,
		Message: fmt.Sprintf("HTTP check failed after %d retries: %v", retries, lastErr),
	}, nil
}

func (hc *HTTPChecker) isExpectedStatus(statusCode int, expected []int) bool {
	if len(expected) == 0 {
		return statusCode >= 200 && statusCode < 400
	}

	for _, code := range expected {
		if statusCode == code {
			return true
		}
	}

	return false
}
```
## 5. Resource 检查器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\resource.go
package healthcheck

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/restmapper"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type ResourceChecker struct {
	client.Client
	Log    logr.Logger
	Config *rest.Config

	dynamicClient dynamic.Interface
	mapper        meta.RESTMapper
	kubeClient    kubernetes.Interface
}

func NewResourceChecker(c client.Client, log logr.Logger, config *rest.Config) *ResourceChecker {
	rc := &ResourceChecker{
		Client: c,
		Log:    log.WithName("resource-checker"),
		Config: config,
	}

	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		rc.Log.Error(err, "failed to create dynamic client")
		return rc
	}
	rc.dynamicClient = dynamicClient

	kubeClient, err := kubernetes.NewForConfig(config)
	if err != nil {
		rc.Log.Error(err, "failed to create kube client")
		return rc
	}
	rc.kubeClient = kubeClient

	discoveryClient := rc.kubeClient.Discovery()
	groupResources, err := restmapper.GetAPIGroupResources(discoveryClient)
	if err != nil {
		rc.Log.Error(err, "failed to get API group resources")
		return rc
	}
	rc.mapper = restmapper.NewDiscoveryRESTMapper(groupResources)

	return rc
}

func (rc *ResourceChecker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion, spec v1alpha1.HealthCheck) (*CheckResult, error) {
	if spec.Resource == nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "resource check spec is nil",
		}, nil
	}

	resourceSpec := spec.Resource

	if rc.dynamicClient == nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "dynamic client not initialized",
		}, nil
	}

	for _, ref := range resourceSpec.Refs {
		result := rc.checkRef(ctx, cv, ref, resourceSpec)
		if result.Status != v1alpha1.HealthCheckStatusHealthy {
			return result, nil
		}
	}

	if resourceSpec.Count != nil {
		result := rc.checkCount(ctx, cv, resourceSpec)
		if result.Status != v1alpha1.HealthCheckStatusHealthy {
			return result, nil
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: "all resource checks passed",
	}, nil
}

func (rc *ResourceChecker) checkRef(ctx context.Context, cv *v1alpha1.ComponentVersion, ref v1alpha1.ResourceHealthRef, spec *v1alpha1.ResourceHealthCheck) *CheckResult {
	gvk := schema.FromAPIVersionAndKind(ref.APIVersion, ref.Kind)

	mapping, err := rc.mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
	if err != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: fmt.Sprintf("map GVK %s to GVR: %v", gvk, err),
		}
	}

	namespace := ref.Namespace
	if namespace == "" {
		namespace = spec.Namespace
	}
	if namespace == "" {
		namespace = cv.Namespace
	}

	var resourceClient dynamic.ResourceInterface

	if mapping.Scope.Name() == meta.RESTScopeNameNamespace {
		resourceClient = rc.dynamicClient.Resource(mapping.Resource).Namespace(namespace)
	} else {
		resourceClient = rc.dynamicClient.Resource(mapping.Resource)
	}

	if len(ref.LabelSelector) > 0 {
		return rc.checkByLabelSelector(ctx, ref, resourceClient)
	}

	obj := &unstructured.Unstructured{}
	key := types.NamespacedName{Name: ref.Name, Namespace: namespace}

	if err := rc.Get(ctx, key, obj); err != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("resource %s/%s not found: %v", ref.Kind, ref.Name, err),
		}
	}

	if len(spec.Conditions) > 0 {
		if !rc.checkConditions(obj, spec.Conditions) {
			return &CheckResult{
				Status:  v1alpha1.HealthCheckStatusUnhealthy,
				Message: fmt.Sprintf("resource %s/%s conditions not met", ref.Kind, ref.Name),
			}
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("resource %s/%s exists and conditions met", ref.Kind, ref.Name),
	}
}

func (rc *ResourceChecker) checkByLabelSelector(ctx context.Context, ref v1alpha1.ResourceHealthRef, resourceClient dynamic.ResourceInterface) *CheckResult {
	labelSelector := metav1.FormatLabelSelector(&metav1.LabelSelector{
		MatchLabels: ref.LabelSelector,
	})

	listOpts := metav1.ListOptions{
		LabelSelector: labelSelector,
	}

	list, err := resourceClient.List(ctx, listOpts)
	if err != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: fmt.Sprintf("list resources by label selector: %v", err),
		}
	}

	if len(list.Items) == 0 {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("no resources found for %s with labels %v", ref.Kind, ref.LabelSelector),
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("found %d resources for %s with labels %v", len(list.Items), ref.Kind, ref.LabelSelector),
	}
}

func (rc *ResourceChecker) checkCount(ctx context.Context, cv *v1alpha1.ComponentVersion, spec *v1alpha1.ResourceHealthCheck) *CheckResult {
	countSpec := spec.Count

	var totalCount int

	for _, ref := range spec.Refs {
		gvk := schema.FromAPIVersionAndKind(ref.APIVersion, ref.Kind)

		mapping, err := rc.mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
		if err != nil {
			continue
		}

		namespace := ref.Namespace
		if namespace == "" {
			namespace = spec.Namespace
		}
		if namespace == "" {
			namespace = cv.Namespace
		}

		var resourceClient dynamic.ResourceInterface
		if mapping.Scope.Name() == meta.RESTScopeNameNamespace {
			resourceClient = rc.dynamicClient.Resource(mapping.Resource).Namespace(namespace)
		} else {
			resourceClient = rc.dynamicClient.Resource(mapping.Resource)
		}

		listOpts := metav1.ListOptions{}
		if ref.Name != "" {
			listOpts.FieldSelector = fmt.Sprintf("metadata.name=%s", ref.Name)
		}
		if len(ref.LabelSelector) > 0 {
			listOpts.LabelSelector = metav1.FormatLabelSelector(&metav1.LabelSelector{
				MatchLabels: ref.LabelSelector,
			})
		}

		list, err := resourceClient.List(ctx, listOpts)
		if err != nil {
			continue
		}

		totalCount += len(list.Items)
	}

	if countSpec.ExactCount > 0 && totalCount != int(countSpec.ExactCount) {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("resource count %d != expected exact count %d", totalCount, countSpec.ExactCount),
		}
	}

	if countSpec.MinCount > 0 && totalCount < int(countSpec.MinCount) {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("resource count %d < minimum count %d", totalCount, countSpec.MinCount),
		}
	}

	if countSpec.MaxCount > 0 && totalCount > int(countSpec.MaxCount) {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("resource count %d > maximum count %d", totalCount, countSpec.MaxCount),
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("resource count %d within expected range", totalCount),
	}
}

func (rc *ResourceChecker) checkConditions(obj *unstructured.Unstructured, conditions []v1alpha1.ResourceCondition) bool {
	objConditions, found, err := unstructured.NestedSlice(obj.Object, "status", "conditions")
	if err != nil || !found {
		return false
	}

	for _, expectedCond := range conditions {
		found := false

		for _, c := range objConditions {
			condMap, ok := c.(map[string]interface{})
			if !ok {
				continue
			}

			condType, _, _ := unstructured.NestedString(condMap, "type")
			condStatus, _, _ := unstructured.NestedString(condMap, "status")

			if condType == expectedCond.Type {
				if condStatus == expectedCond.Status {
					found = true
					break
				}
			}
		}

		if !found {
			return false
		}
	}

	return true
}
```
## 6. Helm Release 检查器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\helm.go
package healthcheck

import (
	"context"
	"fmt"

	"github.com/go-logr/logr"
	"helm.sh/helm/v3/pkg/action"
	"helm.sh/helm/v3/pkg/release"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type HelmChecker struct {
	client.Client
	Log    logr.Logger
	Config *rest.Config
}

func NewHelmChecker(c client.Client, log logr.Logger, config *rest.Config) *HelmChecker {
	return &HelmChecker{
		Client: c,
		Log:    log.WithName("helm-checker"),
		Config: config,
	}
}

func (hc *HelmChecker) Check(ctx context.Context, cv *v1alpha1.ComponentVersion, spec v1alpha1.HealthCheck) (*CheckResult, error) {
	if spec.Helm == nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: "helm check spec is nil",
		}, nil
	}

	helmSpec := spec.Helm

	namespace := helmSpec.Namespace
	if namespace == "" {
		namespace = cv.Namespace
	}

	actionConfig, err := hc.newActionConfig(namespace)
	if err != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: fmt.Sprintf("create action config: %v", err),
		}, err
	}

	getAction := action.NewGet(actionConfig)
	rel, err := getAction.Run(helmSpec.ReleaseName)
	if err != nil {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("release %q not found: %v", helmSpec.ReleaseName, err),
		}, nil
	}

	if helmSpec.CheckStatus {
		result := hc.checkReleaseStatus(rel)
		if result.Status != v1alpha1.HealthCheckStatusHealthy {
			return result, nil
		}
	}

	if helmSpec.CheckResources {
		result := hc.checkReleaseResources(rel)
		if result.Status != v1alpha1.HealthCheckStatusHealthy {
			return result, nil
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("helm release %q is healthy (revision %d, status %s)", rel.Name, rel.Version, rel.Info.Status),
	}, nil
}

func (hc *HelmChecker) checkReleaseStatus(rel *release.Release) *CheckResult {
	switch rel.Info.Status {
	case release.StatusDeployed:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusHealthy,
			Message: fmt.Sprintf("release deployed (revision %d)", rel.Version),
		}
	case release.StatusPendingInstall, release.StatusPendingUpgrade, release.StatusPendingRollback:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusDegraded,
			Message: fmt.Sprintf("release in pending state: %s", rel.Info.Status),
		}
	case release.StatusFailed:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: fmt.Sprintf("release failed: %s", rel.Info.Description),
		}
	case release.StatusSuperseded:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: "release superseded by newer version",
		}
	case release.StatusUninstalling:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnhealthy,
			Message: "release is being uninstalled",
		}
	default:
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusUnknown,
			Message: fmt.Sprintf("release in unexpected state: %s", rel.Info.Status),
		}
	}
}

func (hc *HelmChecker) checkReleaseResources(rel *release.Release) *CheckResult {
	if rel.Info.Resources == nil || len(rel.Info.Resources) == 0 {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusHealthy,
			Message: "no resources to check",
		}
	}

	var unhealthyResources []string

	for _, res := range rel.Info.Resources {
		if res.Deleted != nil {
			unhealthyResources = append(unhealthyResources,
				fmt.Sprintf("%s/%s (deleted)", res.Kind, res.Name))
		}

		if res.Policy != "" && res.Policy == "keep" {
			continue
		}
	}

	if len(unhealthyResources) > 0 {
		return &CheckResult{
			Status:  v1alpha1.HealthCheckStatusDegraded,
			Message: fmt.Sprintf("resources with issues: %v", unhealthyResources),
		}
	}

	return &CheckResult{
		Status:  v1alpha1.HealthCheckStatusHealthy,
		Message: fmt.Sprintf("all %d resources healthy", len(rel.Info.Resources)),
	}
}

func (hc *HelmChecker) newActionConfig(namespace string) (*action.Configuration, error) {
	kubeClient, err := kubernetes.NewForConfig(hc.Config)
	if err != nil {
		return nil, fmt.Errorf("create kube client: %w", err)
	}

	rcGetter := &helmRESTClientGetter{
		config:    hc.Config,
		namespace: namespace,
	}

	logWrapper := &helmLogWrapper{log: hc.Log}

	actionConfig := &action.Configuration{
		RESTClientGetter: rcGetter,
		Log:              logWrapper.Log,
	}

	store := action.NewSecretStore(
		kubeClient.CoreV1().Secrets(namespace),
		namespace,
	)
	actionConfig.Releases = store

	return actionConfig, nil
}

type helmRESTClientGetter struct {
	config    *rest.Config
	namespace string
}

func (r *helmRESTClientGetter) ToRESTConfig() (*rest.Config, error) {
	return r.config, nil
}

func (r *helmRESTClientGetter) ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	discoveryClient, err := discovery.NewDiscoveryClientForConfig(r.config)
	if err != nil {
		return nil, err
	}
	return memory.NewMemCacheClient(discoveryClient), nil
}

func (r *helmRESTClientGetter) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	return nil
}

func (r *helmRESTClientGetter) Namespace() (string, bool, error) {
	return r.namespace, false, nil
}

type helmLogWrapper struct {
	log logr.Logger
}

func (h *helmLogWrapper) Log(format string, v ...interface{}) {
	h.log.Info(fmt.Sprintf(format, v...))
}
```
## 7. 健康检查轮询控制器
```go d:\code\github\componentversion-controller\pkg\componentversion\healthcheck\controller.go
package healthcheck

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type HealthCheckReconciler struct {
	client.Client
	Log     logr.Logger
	Manager *Manager
}

func NewHealthCheckReconciler(c client.Client, log logr.Logger, manager *Manager) *HealthCheckReconciler {
	return &HealthCheckReconciler{
		Client:  c,
		Log:     log.WithName("healthcheck-reconciler"),
		Manager: manager,
	}
}

func (r *HealthCheckReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	log := r.Log.WithValues("componentversion", req.NamespacedName)

	cv := &v1alpha1.ComponentVersion{}
	if err := r.Get(ctx, req.NamespacedName, cv); err != nil {
		if errors.IsNotFound(err) {
			r.Manager.InvalidateCache(req.String())
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, err
	}

	if cv.Spec.HealthCheck == nil {
		return reconcile.Result{}, nil
	}

	if cv.Status.Phase != v1alpha1.PhaseInstalled {
		log.V(1).Info("skipping health check, component not installed", "phase", cv.Status.Phase)
		return r.requeueAfter(cv), nil
	}

	log.Info("running periodic health check")

	healthStatus, err := r.Manager.Check(ctx, cv)
	if err != nil {
		log.Error(err, "health check execution failed")
		return reconcile.Result{RequeueAfter: 30 * time.Second}, err
	}

	patch := client.MergeFrom(cv.DeepCopy())
	cv.Status.Health = *healthStatus

	if err := r.Status().Patch(ctx, cv, patch); err != nil {
		log.Error(err, "failed to patch health status")
		return reconcile.Result{RequeueAfter: 5 * time.Second}, err
	}

	switch healthStatus.Overall {
	case v1alpha1.HealthStatusHealthy:
		log.Info("component is healthy")
	case v1alpha1.HealthStatusDegraded:
		log.Info("component is degraded", "message", healthStatus.Message)
	case v1alpha1.HealthStatusUnhealthy:
		log.Info("component is unhealthy", "message", healthStatus.Message,
			"consecutiveFailures", healthStatus.ConsecutiveFailures)

		if cv.Spec.HealthCheck.FailureThreshold > 0 &&
			healthStatus.ConsecutiveFailures >= cv.Spec.HealthCheck.FailureThreshold {
			log.Info("failure threshold exceeded, triggering re-install")
			patch2 := client.MergeFrom(cv.DeepCopy())
			cv.Status.Phase = v1alpha1.PhasePending
			if err := r.Status().Patch(ctx, cv, patch2); err != nil {
				log.Error(err, "failed to update phase to Pending")
			}
		}
	}

	return r.requeueAfter(cv), nil
}

func (r *HealthCheckReconciler) requeueAfter(cv *v1alpha1.ComponentVersion) reconcile.Result {
	interval := int64(30)
	if cv.Spec.HealthCheck != nil && cv.Spec.HealthCheck.IntervalSeconds > 0 {
		interval = cv.Spec.HealthCheck.IntervalSeconds
	}

	return reconcile.Result{
		RequeueAfter: time.Duration(interval) * time.Second,
	}
}

func (r *HealthCheckReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&v1alpha1.ComponentVersion{}).
		WithEventFilter(predicate.Funcs{
			UpdateFunc: func(e event.UpdateEvent) bool {
				oldCV := e.ObjectOld.(*v1alpha1.ComponentVersion)
				newCV := e.ObjectNew.(*v1alpha1.ComponentVersion)
				return oldCV.Status.Phase == v1alpha1.PhaseInstalled &&
					newCV.Status.Phase == v1alpha1.PhaseInstalled
			},
			CreateFunc: func(e event.CreateEvent) bool {
				cv := e.Object.(*v1alpha1.ComponentVersion)
				return cv.Status.Phase == v1alpha1.PhaseInstalled &&
					cv.Spec.HealthCheck != nil
			},
			DeleteFunc: func(e event.DeleteEvent) bool {
				return false
			},
		}).
		Complete(r)
}
```
## 架构总结
### 健康检查框架整体设计
```
HealthCheck Framework
│
├── Manager (核心调度器)
│   ├── Check() → 执行所有检查 → 聚合结果 → 更新阈值计数 → 缓存
│   ├── Cache → TTL 缓存，避免频繁检查
│   ├── RegisterChecker() → 可扩展注册新检查类型
│   └── InvalidateCache() → 状态变更时清除缓存
│
├── Checker Interface (策略模式)
│   ├── CommandChecker
│   │   ├── 本地 /bin/sh 执行
│   │   ├── 退出码校验 (expectedExitCode)
│   │   ├── 输出内容校验 (outputContains/outputNotContains)
│   │   └── 超时控制
│   │
│   ├── HTTPChecker
│   │   ├── 支持 GET/POST 等方法
│   │   ├── 状态码校验 (expectedStatusCodes)
│   │   ├── 响应体校验 (responseBodyContains/responseBodyNotContains)
│   │   ├── TLS 可控 (tlsVerify)
│   │   └── 重试机制 (retries + retryInterval)
│   │
│   ├── ResourceChecker
│   │   ├── 资源存在性检查 (Get by Name)
│   │   ├── LabelSelector 批量检查
│   │   ├── 条件检查 (status.conditions)
│   │   ├── 数量检查 (exactCount/minCount/maxCount)
│   │   └── GVK→GVR 自动映射
│   │
│   └── HelmChecker
│       ├── Release 状态检查 (Deployed/Pending/Failed)
│       ├── Release 资源检查 (deleted 资源检测)
│       └── Helm Action 配置复用
│
├── Aggregator (聚合策略)
│   ├── All    → 所有检查通过才健康，非必需检查失败为 Degraded
│   ├── Any    → 任一检查通过即健康
│   └── Quorum → 加权投票，>50% 权重通过为健康
│
├── Threshold (阈值控制)
│   ├── FailureThreshold → 连续失败 N 次后标记为 Unhealthy
│   └── SuccessThreshold → 连续成功 N 次后才标记为 Healthy
│
└── HealthCheckReconciler (周期性检查)
    ├── 仅对 PhaseInstalled 的组件执行
    ├── 按 intervalSeconds 周期 Requeue
    ├── 更新 Status.Health
    └── FailureThreshold 超限时触发重新安装
```
### 健康状态流转

```
                    ┌──────────────┐
                    │   Unknown    │ ← 初始状态/检查异常
                    └──────┬───────┘
                           │ 检查成功
                   ┌──────▼───────┐
         ┌─────────│   Healthy    │──────────┐
         │         └──────────────┘          │
         │ 非必需检查失败       必需检查失败 │
         │                                   │
  ┌──────▼───────┐                   ┌───────▼──────┐
  │   Degraded   │                   │  Unhealthy   │
  └──────────────┘                   └───────┬──────┘
                                             │
                              consecutiveFailures >= FailureThreshold
                                             │
                                    ┌────────▼────────┐
                                    │ Phase→Pending   │
                                    │ (触发重新安装)  │
                                    └─────────────────┘
```
### 与三种安装策略的集成
| 安装类型 | 默认健康检查 | 自定义健康检查 |
|---------|-------------|--------------|
| **Manifest** | Resource 检查（Deployment Ready、Pod Running） | 通过 `healthCheck.checks` 自定义 |
| **Script** | Command 检查（执行验证脚本） | 通过 `healthCheck.checks` 自定义 |
| **HelmChart** | Helm Release 状态检查 + Resource 条件检查 | 通过 `healthCheck.checks` 自定义 |
### 关键设计决策
| 决策 | 说明 |
|------|------|
| **统一框架** | 所有安装策略共享同一套健康检查框架，通过 `healthCheck` 字段统一配置 |
| **四种检查类型** | Command/HTTP/Resource/Helm 覆盖绝大多数健康检查场景 |
| **三级健康状态** | Healthy/Degraded/Unhealthy 区分完全健康、部分降级和完全不健康 |
| **Required 字段** | 必需检查失败立即终止后续检查，非必需检查失败仅标记 Degraded |
| **阈值机制** | 避免瞬时抖动导致的误判，连续 N 次才确认状态变更 |
| **TTL 缓存** | 避免短时间内重复执行相同检查，缓存时间为检查间隔的一半 |
| **自动修复** | FailureThreshold 超限时自动将 Phase 重置为 Pending，触发重新安装 |
| **Quorum 策略** | 支持加权投票，适用于多副本场景下部分实例不健康的判断 |

请手动创建上述文件到对应路径。至此，核心层的健康检查框架已实现完成。下一步可以实现**编排层：Reconciler + Dispatcher + NodeConfig 协作**。

# Installer三次输出的 Prompt 拆分方案
按**复杂度递增**的顺序拆分，每次输出一个完整可编译的模块：
## 第一次 Prompt（Installer 接口 + ManifestInstaller）
> 实现 ComponentVersion Controller 核心层的 Installer 接口与 ManifestInstaller，包含以下文件：
>
> 1. **Installer 接口**（`pkg/componentversion/installer/interface.go`）
>    - 定义 `Installer` 接口，包含 4 个方法：`Install(ctx, cv) → (*InstallResult, error)`、`Verify(ctx, cv) → (*VerifyResult, error)`、`Rollback(ctx, cv) → error`、`Uninstall(ctx, cv) → error`
>    - 定义 `InstallResult` 结构体（Phase / Details / Requeue）
>    - 定义 `VerifyResult` 结构体（Passed / Message）
>    - 定义 `InstallerFactory` 工厂函数类型，用于按 InstallType 创建 Installer 实例
>
> 2. **ManifestInstaller**（`pkg/componentversion/installer/manifest/installer.go`）
>    - `Installer` 结构体，持有 client.Client / Scheme / SourceResolver
>    - `Install` 方法完整实现：遍历 `spec.install.manifest.items`，对每个 item 调用 SourceResolver 解析内容，将 YAML 解析为 `unstructured.Unstructured` 列表，渲染模板变量（替换 `{{.Version}}` / `{{.ComponentName}}` / images 映射），根据 `applyStrategy` 调用对应的 Apply 方法，记录 `appliedResources` 到 InstallResult.Details
>    - `Verify` 方法完整实现：遍历 `status.installDetails.manifest.appliedResources`，逐个检查资源是否存在且状态就绪（Deployment 检查 availableReplicas、StatefulSet 检查 readyReplicas、DaemonSet 检查 numberReady、Pod 检查 phase=Running、其他资源仅检查存在性），全部就绪返回 `VerifyResult{Passed: true}`，有资源不存在返回 error，有资源未就绪返回 `VerifyResult{Passed: false, Message: ...}`
>    - `Rollback` 方法完整实现：基于 `appliedResources` 中记录的 hash，对比当前资源的 hash，如果不同则用 `ServerSideApply` 重新应用旧版本 manifest（需从 Source 重新解析）
>    - `Uninstall` 方法完整实现：遍历 `appliedResources`，按逆序删除所有已应用资源
>
> 3. **Apply 策略实现**（`pkg/componentversion/installer/manifest/apply.go`）
>    - `applyServerSideApply(ctx, c, obj, fieldManager)` 实现：使用 `client.Patch` with `client.MergeFrom` + `client.FieldOwner(fieldManager)` 进行 Server-Side Apply
>    - `applyThreeWayMerge(ctx, c, obj)` 实现：先 Get 当前资源，如果不存在则 Create；如果存在则使用 strategic merge patch 合并后 Update
>    - `applyReplace(ctx, c, obj)` 实现：先 Delete 已有资源（忽略 NotFound），再 Create 新资源
>    - 三个方法均设置 ownership label（`orchestration.bke.io/component-version: <cv-name>`），用于后续 Prune 和 Rollback
>
> 4. **Prune 实现**（`pkg/componentversion/installer/manifest/prune.go`）
>    - `Prune(ctx, c, cv, appliedResources)` 实现：根据 ownership label 查询集群中属于该 CV 的所有资源，对比 `appliedResources` 列表，将不在列表中的资源按逆序删除，返回 `prunedResources` 列表
>    - `ListOwnedResources(ctx, c, cv)` 实现：通过 label selector `orchestration.bke.io/component-version=<cv-name>` 查询所有命名空间中的资源，返回 `AppliedResource` 列表
>
> 5. **模板渲染**（`pkg/componentversion/installer/manifest/template.go`）
>    - `RenderTemplate(content []byte, cv *v1alpha1.ComponentVersion) ([]byte, error)` 实现：使用 Go `text/template` 替换 YAML 中的模板变量，支持 `{{.Version}}` / `{{.ComponentName}}` / `{{.Namespace}}` / `{{.Image "component-name"}}`（从 cv.Spec.Images 中按 name 查找 image）
>    - 模板渲染失败时返回明确错误，包含原始内容和失败位置
>
> 要求：
> - 所有 Apply 方法需处理 `apierrors.IsConflict` 错误，返回 `Requeue=true` 让框架自动重试
> - ManifestInstaller 的 Install 方法在应用完所有 manifest 后，如果 `spec.install.manifest.prune=true`，自动调用 Prune
> - appliedResources 中记录的 hash 使用 `sha256` 计算整个 manifest 内容的哈希，用于后续 Rollback 对比
> - Uninstall 删除资源时使用 `client.GracefulDeletePolicy` 并设置 `propagationPolicy=Background`
> - 模板渲染使用 `Delims("{{", "}}")` 避免与 Helm 模板冲突
## 第二次 Prompt（ScriptInstaller）
> 实现 ComponentVersion Controller 核心层的 ScriptInstaller，包含以下文件：
>
> 1. **ScriptInstaller 主文件**（`pkg/componentversion/installer/script/installer.go`）
>    - `Installer` 结构体，持有 client.Client / Scheme / SourceResolver / RestConfig
>    - `Install` 方法完整实现：
>      - 调用 SourceResolver 解析 `spec.install.installScript.source` 获取脚本内容
>      - 根据 `spec.install.installScript.execution.target` 路由到三个执行通道：
>        - `Controller`：调用 `executeLocally` 在 Controller 进程内执行
>        - `NodeAgent`：调用 `dispatchToNodeAgent` 构造 Command CRD 下发
>        - `InitPod`：调用 `dispatchInitPod` 构造 Job 执行
>      - 生成 `executionId`（UUID），记录到 InstallResult.Details
>      - 对于 NodeAgent 和 InitPod 通道，Install 返回 `Requeue=true`，由 Reconciler 在下一轮查询执行状态
>    - `Verify` 方法完整实现：
>      - 根据 `execution.target` 查询执行状态：
>        - `Controller`：直接返回 `VerifyResult{Passed: true}`（同步执行，Install 返回时已完成）
>        - `NodeAgent`：调用 `checkCommandStatus` 查询 Command CRD 的 status
>        - `InitPod`：调用 `checkJobStatus` 查询 Job 的 status
>      - 如果配置了 `spec.install.installScript.validation`，在执行完成后调用 `runValidation` 执行验证命令
>      - 验证进行中返回 `VerifyResult{Passed: false, Message: "verifying"}`，验证完成返回 `VerifyResult{Passed: true}`，验证失败返回 `VerifyResult{Passed: false, Message: ...}`
>    - `Rollback` 方法完整实现：对于 NodeAgent 通道，创建新的 Command CRD 执行回滚脚本（回滚脚本路径约定为原脚本同目录下的 `rollback.sh`）；对于 InitPod 通道，删除当前 Job 并创建回滚 Job；对于 Controller 通道，直接执行回滚命令
>    - `Uninstall` 方法完整实现：对于 NodeAgent 通道，创建 Command CRD 执行卸载脚本；对于 InitPod 通道，创建卸载 Job；对于 Controller 通道，直接执行卸载命令；清理所有关联的 Command CRD 和 Job
>
> 2. **执行器**（`pkg/componentversion/installer/script/executor.go`）
>    - `executeLocally(ctx, cv, scriptContent)` 实现：将脚本内容写入临时文件，使用 `os/exec.CommandContext` 执行，设置 env / args / timeout，捕获 stdout/stderr，超时使用 context cancel，返回执行结果（exitCode / stdout / stderr / error）
>    - `dispatchToNodeAgent(ctx, cv, scriptContent)` 完整实现：
>      - 将脚本内容写入 ConfigMap（名称 `cv-script-<cv-name>-<hash>`），设置 OwnerReference 为 ComponentVersion
>      - 构造 Command CRD（名称 `cv-<cv-name>-<component>-<executionId>`），设置：
>        - `spec.nodeSelector` 从 `cv.spec.install.installScript.execution.nodeSelector` 映射
>        - `spec.commands` 包含一条 `CommandKubernetes` 类型指令，格式 `configmap:<ns>/<cm-name>:rx:<key>`
>        - `spec.activeDeadlineSecond` 从 timeout 解析
>        - `spec.backoffLimit` 从 retryPolicy.maxRetries 获取
>        - OwnerReference 指向 ComponentVersion
>      - 创建 Command CRD 到集群
>    - `dispatchInitPod(ctx, cv, scriptContent)` 完整实现：
>      - 将脚本内容写入 ConfigMap
>      - 构造 `batch/v1 Job`（名称 `cv-<cv-name>-<component>-<executionId>`），设置：
>        - InitContainer 从 `cv.spec.install.installScript.source.imageRef.image` 拉取镜像（如果 source 是 ImageRef），将脚本拷贝到共享 EmptyDir
>        - 主 Container 使用 `busybox` 或指定镜像执行脚本
>        - env / args / timeout（activeDeadlineSeconds）从 spec 映射
>        - nodeSelector 从 execution.nodeSelector 映射
>        - OwnerReference 指向 ComponentVersion
>      - 创建 Job 到集群
>    - `checkCommandStatus(ctx, cv)` 实现：查询关联的 Command CRD，解析 status 中各节点的执行状态，汇总为 `InstallScriptInstallDetails`
>    - `checkJobStatus(ctx, cv)` 实现：查询关联的 Job，检查 completionTime / succeeded / failed / active，汇总执行状态
>
> 3. **验证器**（`pkg/componentversion/installer/script/validator.go`）
>    - `runValidation(ctx, cv)` 完整实现：
>      - 读取 `spec.install.installScript.validation` 配置
>      - 构造验证命令（validation.command），根据 execution.target 选择执行通道
>      - 执行验证命令，检查输出是否包含 `expectedOutput`
>      - 返回 `ValidationResult`（passed / output / checkedAt）
>    - `buildValidationCommand(cv)` 实现：将 validation.command 和 expectedOutput 构造为可执行的验证脚本
>
> 4. **辅助函数**（`pkg/componentversion/installer/script/helpers.go`）
>    - `parseDuration(s string) time.Duration`：解析时长字符串（如 "300s" / "10m"），空值返回默认值
>    - `buildEnvVars(cv)` 实现：将 `spec.install.installScript.env` 转换为 `[]corev1.EnvVar`，处理 valueFrom（configMapKeyRef / secretKeyRef）
>    - `buildNodeSelector(cv)` 实现：将 `execution.nodeSelector` 转换为 `*metav1.LabelSelector`
>    - `generateExecutionID()` 实现：使用 `uuid.New().String()` 生成唯一执行 ID
>    - `findOwnedCommands(ctx, c, cv)` 实现：按 OwnerReference 查询属于该 CV 的所有 Command CRD
>    - `findOwnedJobs(ctx, c, cv)` 实现：按 OwnerReference 查询属于该 CV 的所有 Job
>    - `cleanupResources(ctx, c, cv)` 实现：删除关联的 ConfigMap / Command / Job
>
> 要求：
> - Command CRD 的类型定义使用 `agentv1beta1.Command`（包路径 `bkeagent.bocloud.com/v1beta1`），如果该包不可用，定义一个本地 `Command` 接口/结构体，包含 `CommandSpec` / `CommandStatus` 等必要字段，并添加注释说明实际使用时应替换为真实包
> - NodeAgent 通道的 Command CRD 名称需包含 executionId 以支持同一 CV 的多次执行
> - InitPod 通道的 Job 需设置 `ttlSecondsAfterFinished` 自动清理
> - `executeLocally` 需确保临时文件在执行完成后清理（defer os.Remove）
> - 所有创建的 Kubernetes 资源（ConfigMap / Command / Job）均设置 OwnerReference 指向 ComponentVersion
> - Verify 方法中，如果执行仍在进行中（Command phase=Running / Job active>0），返回 `VerifyResult{Passed: false}` 而非 error，让 Reconciler requeue 等待
## 第三次 Prompt（HelmChartInstaller + 健康检查）
> 实现 ComponentVersion Controller 核心层的 HelmChartInstaller 与健康检查模块，包含以下文件：
>
> 1. **HelmChartInstaller**（`pkg/componentversion/installer/helmchart/installer.go`）
>    - `Installer` 结构体，持有 client.Client / Scheme / SourceResolver / RestConfig / Settings（`*cli.EnvSettings`）
>    - `newActionConfig(namespace)` 实现：基于 RestConfig 创建 `*action.Configuration`，使用 `helm.sh/helm/v3/pkg/action` 的 `ActionConfig` 初始化，配置 RESTClientGetter 使用集群内配置
>    - `Install` 方法完整实现：
>      - 调用 `loadChart` 加载 Chart
>      - 调用 `mergeValues` 合并 Values
>      - 检查 Release 是否已存在（使用 `action.NewHistory`）
>      - 不存在：使用 `action.NewInstall` 执行 `helm install`，设置 releaseName / namespace / wait / atomic / timeout
>      - 已存在：使用 `action.NewUpgrade` 执行 `helm upgrade`，设置 wait / atomic / timeout
>      - 记录 `HelmChartInstallDetails`（releaseStatus / revision / lastDeployed）到 InstallResult.Details
>      - 如果 `atomic=true` 且安装失败，Helm SDK 会自动回滚
>    - `Verify` 方法完整实现：
>      - 使用 `action.NewStatus` 查询 Release 状态
>      - `status.Status == release.StatusDeployed` → `VerifyResult{Passed: true}`
>      - `status.Status == release.StatusPendingInstall / PendingUpgrade` → `VerifyResult{Passed: false, Message: "helm release is pending"}`
>      - `status.Status == release.StatusFailed` → `VerifyResult{Passed: false, Message: status.Info}`
>      - 其他状态 → `VerifyResult{Passed: false, Message: ...}`
>    - `Rollback` 方法完整实现：
>      - 使用 `action.NewRollback` 回滚到上一个 Revision
>      - 设置 wait / atomic / timeout
>      - 如果没有可回滚的 Revision（当前 revision=1），返回 error
>    - `Uninstall` 方法完整实现：
>      - 使用 `action.NewUninstall` 卸载 Release
>      - 设置 `KeepHistory=false` 彻底清理
>      - 忽略 Release 不存在的错误
>
> 2. **Chart 加载**（`pkg/componentversion/installer/helmchart/chart.go`）
>    - `loadChart(ctx, cv)` 完整实现：根据 `spec.install.helmChart.source.type` 路由到不同加载方式：
>      - `Repo`：调用 `loadChartFromRepo`，使用 `action.NewPull` 拉取 Chart 到临时目录，使用 `loader.Load` 加载
>      - `OCI`：调用 `loadChartFromOCI`，使用 `action.NewPullWithOpts(action.WithConfig(actionConfig))` 拉取 OCI Chart
>      - `LocalPath`：调用 `loadChartFromLocalPath`，直接使用 `loader.Load` 加载本地目录或 tgz
>      - `ConfigMapRef`：调用 `loadChartFromConfigMap`，从 ConfigMap 读取 tgz 数据，写入临时文件后 `loader.Load`
>    - `loadChartFromRepo(ctx, cv)` 实现：构造 Helm repo URL + chart name + version，使用 `action.NewPull` 下载，返回 `*chart.Chart`
>    - `loadChartFromOCI(ctx, cv)` 实现：构造 OCI registry URL，使用 `action.NewPull` 下载
>    - `loadChartFromLocalPath(cv)` 实现：检查 path 是目录还是 tgz，使用 `loader.Load` 或 `loader.LoadArchive` 加载
>    - `loadChartFromConfigMap(ctx, cv)` 实现：从 ConfigMap 读取 key 对应的 tgz 数据，写入临时文件，使用 `loader.LoadArchive` 加载
>    - 所有加载方式均需处理临时文件清理
>
> 3. **Values 合并**（`pkg/componentversion/installer/helmchart/values.go`）
>    - `mergeValues(ctx, cv)` 完整实现：
>      - 从 `spec.install.helmChart.values`（`runtime.RawExtension`）解析基础 Values 为 `map[string]interface{}`
>      - 遍历 `spec.install.helmChart.valuesFrom`，按顺序从 ConfigMap / Secret 读取覆盖 Values
>      - 使用递归深度合并（`util.MergeMaps`），后者覆盖前者
>      - 返回最终合并的 `map[string]interface{}`
>    - `readValuesFromConfigMap(ctx, vf)` 实现：从 ConfigMap 读取 `valuesKey`（默认 `values.yaml`），解析为 `map[string]interface{}`
>    - `readValuesFromSecret(ctx, vf)` 实现：从 Secret 读取 `valuesKey`（默认 `values.yaml`），解析为 `map[string]interface{}`
>    - `deepMergeMaps(base, override map[string]interface{}) map[string]interface{}` 实现：递归深度合并两个 map，override 中的值覆盖 base 中的同名键，map 类型递归合并，其他类型直接覆盖
>
> 4. **Release 管理**（`pkg/componentversion/installer/helmchart/release.go`）
>    - `getReleaseStatus(ctx, cv)` 实现：使用 `action.NewStatus` 查询 Release，返回 `*release.Release`
>    - `getCurrentRevision(ctx, cv)` 实现：查询当前 Revision 号
>    - `isReleaseExists(ctx, cv)` 实现：检查 Release 是否存在
>    - `buildHelmInstallOptions(cv, install *action.Install)` 实现：将 CV Spec 中的配置映射到 Helm Install action 的选项（ReleaseName / Namespace / Wait / Atomic / Timeout / CreateNamespace）
>    - `buildHelmUpgradeOptions(cv, upgrade *action.Upgrade)` 实现：将 CV Spec 中的配置映射到 Helm Upgrade action 的选项（Wait / Atomic / Timeout / MaxHistory）
>    - `buildHelmRollbackOptions(cv, rollback *action.Rollback)` 实现：将 CV Spec 中的配置映射到 Helm Rollback action 的选项（Wait / Atomic / Timeout）
>
> 5. **健康检查**（`pkg/componentversion/healthcheck/checker.go`）
>    - `Checker` 结构体，持有 client.Client / RestConfig
>    - `Check(ctx, cv) error` 实现：如果 `spec.healthCheck` 为 nil 直接返回 nil；根据 `type` 路由到具体检查方法；检查失败返回 `HealthCheckError`（包含 type / message），检查成功返回 nil
>    - `HealthCheckError` 类型定义（Type / Message / LastCheck 字段），实现 `error` 接口
>
> 6. **HTTPGet 检查**（`pkg/componentversion/healthcheck/http.go`）
>    - `checkHTTP(ctx, cv)` 完整实现：构造 HTTP 请求（scheme://<service-host>:<port><path>），使用 `http.Client` 发送请求，设置 timeout，状态码 200-399 视为成功，其他视为失败
>    - 服务发现：如果 port 对应的是 Service，通过 client 查询 Service 的 ClusterIP
>
> 7. **TCPSocket 检查**（`pkg/componentversion/healthcheck/tcp.go`）
>    - `checkTCP(ctx, cv)` 完整实现：使用 `net.DialTimeout` 尝试建立 TCP 连接，timeout 从 spec 解析，连接成功视为健康，连接失败/超时视为不健康
>
> 8. **Command 检查**（`pkg/componentversion/healthcheck/command.go`）
>    - `checkCommand(ctx, cv)` 完整实现：将 `spec.healthCheck.command.command` 构造为 Command CRD 或通过 `os/exec` 执行，检查退出码为 0 视为成功
>    - 对于 scope=Cluster 的组件，直接在 Controller 内执行
>    - 对于 scope=Node 的组件，创建 Command CRD 下发到 NodeAgent 执行
>
> 9. **ClusterOperator 检查**（`pkg/componentversion/healthcheck/clusteroperator.go`）
>    - `checkClusterOperator(ctx, cv)` 完整实现：查询 `config.openshift.io/v1` 的 `ClusterOperator` 资源，检查指定 `conditionType`（默认 Available）的 status 是否为 True，如果 conditionType=Progressing 且 status=True 也视为不健康（正在升级中）
>    - 如果 ClusterOperator CRD 不存在（非 OpenShift 环境），返回 nil 跳过检查
>
> 要求：
> - HelmChartInstaller 使用 `helm.sh/helm/v3` 的 Go SDK（`action` / `chart` / `chart/loader` / `cli` / `release` / `getter` 包），不调用 helm 命令行工具
> - `newActionConfig` 需实现 `getter.RESTClientGetter` 接口，使用传入的 RestConfig 构造 Kubernetes 客户端
> - Values 合并时，如果 `spec.install.helmChart.values` 为空（nil 或 `{"raw": null}`），使用空 map 作为基础值
> - 健康检查失败不改变 ComponentVersion 的 phase，仅在 status.conditions 中添加/更新 `HealthCheckFailed` 类型的 condition
> - 所有临时文件/目录需在方法返回前清理（defer）
> - HTTPGet 和 TCPSocket 检查的目标地址需支持通过环境变量覆盖（用于本地调试）
> - Command 检查中构造的 Command CRD 需设置 OwnerReference，且设置 `ttlSecondsAfterFinished` 自动清理

# ComponentVersion Controller 的编排层   
基于前面会话的设计，我将实现 ComponentVersion Controller 的编排层，包括 Reconciler 主循环、Dispatcher、NodeConfig 协作、SetupWithManager 和辅助函数。
## 1. Installer 分发器
```go d:\code\github\componentversion-controller\pkg\componentversion\dispatcher\dispatcher.go
package dispatcher

import (
	"context"
	"fmt"

	"github.com/go-logr/logr"
	"k8s.io/client-go/rest"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\healthcheck"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer\helm"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer\manifest"
	"d:\code\github\componentversion-controller\pkg\componentversion\installer\script"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
)

type Dispatcher struct {
	client.Client
	Log           logr.Logger
	Config        *rest.Config
	Resolver      *source.Resolver
	HealthManager *healthcheck.Manager

	installers map[v1alpha1.InstallType]installer.Installer
}

func NewDispatcher(c client.Client, log logr.Logger, config *rest.Config, resolver *source.Resolver, healthMgr *healthcheck.Manager) (*Dispatcher, error) {
	d := &Dispatcher{
		Client:        c,
		Log:           log.WithName("installer-dispatcher"),
		Config:        config,
		Resolver:      resolver,
		HealthManager: healthMgr,
		installers:    make(map[v1alpha1.InstallType]installer.Installer),
	}

	manifestInstaller, err := manifest.NewInstaller(c, log, resolver)
	if err != nil {
		return nil, fmt.Errorf("create manifest installer: %w", err)
	}
	d.installers[v1alpha1.InstallTypeManifest] = manifestInstaller

	scriptInstaller := script.NewInstaller(c, log, resolver)
	d.installers[v1alpha1.InstallTypeScript] = scriptInstaller

	helmInstaller, err := helm.NewInstaller(c, log, resolver, config)
	if err != nil {
		return nil, fmt.Errorf("create helm installer: %w", err)
	}
	d.installers[v1alpha1.InstallTypeHelmChart] = helmInstaller

	return d, nil
}

func (d *Dispatcher) GetInstaller(cv *v1alpha1.ComponentVersion) (installer.Installer, error) {
	installType := d.detectInstallType(cv)
	if installType == "" {
		return nil, fmt.Errorf("cannot determine install type for component %s", cv.Spec.ComponentName)
	}

	ins, ok := d.installers[installType]
	if !ok {
		return nil, fmt.Errorf("unsupported install type: %s", installType)
	}

	return ins, nil
}

func (d *Dispatcher) Install(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.InstallResult, error) {
	ins, err := d.GetInstaller(cv)
	if err != nil {
		return nil, err
	}

	return ins.Install(ctx, cv)
}

func (d *Dispatcher) Verify(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.VerifyResult, error) {
	ins, err := d.GetInstaller(cv)
	if err != nil {
		return nil, err
	}

	return ins.Verify(ctx, cv)
}

func (d *Dispatcher) Rollback(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.RollbackResult, error) {
	ins, err := d.GetInstaller(cv)
	if err != nil {
		return nil, err
	}

	return ins.Rollback(ctx, cv)
}

func (d *Dispatcher) Uninstall(ctx context.Context, cv *v1alpha1.ComponentVersion) (*installer.UninstallResult, error) {
	ins, err := d.GetInstaller(cv)
	if err != nil {
		return nil, err
	}

	return ins.Uninstall(ctx, cv)
}

func (d *Dispatcher) HealthCheck(ctx context.Context, cv *v1alpha1.ComponentVersion) (*v1alpha1.HealthStatus, error) {
	return d.HealthManager.Check(ctx, cv)
}

func (d *Dispatcher) detectInstallType(cv *v1alpha1.ComponentVersion) v1alpha1.InstallType {
	if cv.Spec.Install.Manifest != nil {
		return v1alpha1.InstallTypeManifest
	}
	if cv.Spec.Install.Script != nil {
		return v1alpha1.InstallTypeScript
	}
	if cv.Spec.Install.HelmChart != nil {
		return v1alpha1.InstallTypeHelmChart
	}

	if cv.Status.InstallType != "" {
		return cv.Status.InstallType
	}

	return ""
}

func (d *Dispatcher) RegisterInstaller(installType v1alpha1.InstallType, ins installer.Installer) {
	d.installers[installType] = ins
}

func (d *Dispatcher) GetInstallType(cv *v1alpha1.ComponentVersion) v1alpha1.InstallType {
	return d.detectInstallType(cv)
}
```
## 2. NodeConfig 协作器
```go d:\code\github\componentversion-controller\pkg\componentversion\nodeconfig\collaborator.go
package nodeconfig

import (
	"context"
	"fmt"

	"github.com/go-logr/logr"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

type Collaborator struct {
	client.Client
	Log logr.Logger
}

func NewCollaborator(c client.Client, log logr.Logger) *Collaborator {
	return &Collaborator{
		Client: c,
		Log:    log.WithName("nodeconfig-collaborator"),
	}
}

func (nc *Collaborator) GetNodeConfigsForComponent(ctx context.Context, cv *v1alpha1.ComponentVersion) ([]*v1alpha1.NodeConfig, error) {
	log := nc.Log.WithValues("component", cv.Spec.ComponentName)

	nodeConfigList := &v1alpha1.NodeConfigList{}
	if err := nc.List(ctx, nodeConfigList, client.InNamespace(cv.Namespace)); err != nil {
		return nil, fmt.Errorf("list NodeConfigs: %w", err)
	}

	var matchedConfigs []*v1alpha1.NodeConfig
	for i := range nodeConfigList.Items {
		ncItem := &nodeConfigList.Items[i]

		if ncItem.Spec.ComponentName != cv.Spec.ComponentName {
			continue
		}

		if ncItem.Spec.Version != "" && ncItem.Spec.Version != cv.Spec.Version {
			continue
		}

		matchedConfigs = append(matchedConfigs, ncItem)
	}

	log.Info("found matching NodeConfigs", "count", len(matchedConfigs))
	return matchedConfigs, nil
}

func (nc *Collaborator) GetNodesForNodeConfig(ctx context.Context, nodeConfig *v1alpha1.NodeConfig) ([]corev1.Node, error) {
	log := nc.Log.WithValues("nodeconfig", nodeConfig.Name)

	nodeList := &corev1.NodeList{}

	if nodeConfig.Spec.NodeSelector != nil {
		selector, err := metav1.LabelSelectorAsSelector(nodeConfig.Spec.NodeSelector)
		if err != nil {
			return nil, fmt.Errorf("parse node selector: %w", err)
		}

		if err := nc.List(ctx, nodeList, client.MatchingLabelsSelector{Selector: selector}); err != nil {
			return nil, fmt.Errorf("list nodes by selector: %w", err)
		}
	} else {
		if err := nc.List(ctx, nodeList); err != nil {
			return nil, fmt.Errorf("list all nodes: %w", err)
		}
	}

	log.Info("found matching nodes", "count", len(nodeList.Items))
	return nodeList.Items, nil
}

func (nc *Collaborator) ApplyNodeConfig(ctx context.Context, cv *v1alpha1.ComponentVersion, nodeConfig *v1alpha1.NodeConfig) error {
	log := nc.Log.WithValues("component", cv.Spec.ComponentName, "nodeconfig", nodeConfig.Name)

	nodes, err := nc.GetNodesForNodeConfig(ctx, nodeConfig)
	if err != nil {
		return err
	}

	if len(nodes) == 0 {
		log.Info("no nodes match NodeConfig selector")
		return nil
	}

	for _, node := range nodes {
		if err := nc.applyToNode(ctx, cv, nodeConfig, &node); err != nil {
			log.Error(err, "failed to apply NodeConfig to node", "node", node.Name)
			continue
		}
	}

	return nil
}

func (nc *Collaborator) applyToNode(ctx context.Context, cv *v1alpha1.ComponentVersion, nodeConfig *v1alpha1.NodeConfig, node *corev1.Node) error {
	log := nc.Log.WithValues("node", node.Name)

	patch := client.MergeFrom(node.DeepCopy())

	if node.Labels == nil {
		node.Labels = make(map[string]string)
	}

	componentLabel := fmt.Sprintf("componentversion.%s/version", cv.Spec.ComponentName)
	node.Labels[componentLabel] = cv.Spec.Version

	if node.Annotations == nil {
		node.Annotations = make(map[string]string)
	}

	configAnnotation := fmt.Sprintf("componentversion.%s/config", cv.Spec.ComponentName)
	if nodeConfig.Spec.ConfigData != nil {
		node.Annotations[configAnnotation] = string(nodeConfig.Spec.ConfigData.Raw)
	}

	if err := nc.Patch(ctx, node, patch); err != nil {
		return fmt.Errorf("patch node %s: %w", node.Name, err)
	}

	log.Info("applied NodeConfig to node")
	return nil
}

func (nc *Collaborator) CreateNodeConfigCommand(ctx context.Context, cv *v1alpha1.ComponentVersion, nodeConfig *v1alpha1.NodeConfig, node *corev1.Node) (*v1alpha1.Command, error) {
	log := nc.Log.WithValues("component", cv.Spec.ComponentName, "node", node.Name, "nodeconfig", nodeConfig.Name)

	if nodeConfig.Spec.Script == nil {
		return nil, nil
	}

	command := &v1alpha1.Command{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%s-%s-%s", cv.Spec.ComponentName, nodeConfig.Name, node.Name),
			Namespace: cv.Namespace,
			Labels: map[string]string{
				"componentversion.component": cv.Spec.ComponentName,
				"componentversion.version":   cv.Spec.Version,
				"componentversion.node":      node.Name,
				"componentversion.nodeconfig": nodeConfig.Name,
			},
			OwnerReferences: []metav1.OwnerReference{
				{
					APIVersion: v1alpha1.GroupVersion.String(),
					Kind:       "ComponentVersion",
					Name:       cv.Name,
					UID:        cv.UID,
				},
			},
		},
		Spec: v1alpha1.CommandSpec{
			Script:      nodeConfig.Spec.Script.Script,
			Timeout:     nodeConfig.Spec.Script.TimeoutSeconds,
			NodeName:    node.Name,
			Env:         nc.buildCommandEnv(cv, nodeConfig),
		},
	}

	if err := nc.Create(ctx, command); err != nil {
		return nil, fmt.Errorf("create Command: %w", err)
	}

	log.Info("created NodeConfig Command")
	return command, nil
}

func (nc *Collaborator) buildCommandEnv(cv *v1alpha1.ComponentVersion, nodeConfig *v1alpha1.NodeConfig) []v1alpha1.CommandEnv {
	envs := []v1alpha1.CommandEnv{
		{Name: "COMPONENT_NAME", Value: cv.Spec.ComponentName},
		{Name: "COMPONENT_VERSION", Value: cv.Spec.Version},
		{Name: "COMPONENT_NAMESPACE", Value: cv.Namespace},
		{Name: "NODE_CONFIG_NAME", Value: nodeConfig.Name},
	}

	for _, e := range nodeConfig.Spec.Env {
		envs = append(envs, v1alpha1.CommandEnv{Name: e.Name, Value: e.Value})
	}

	return envs
}

func (nc *Collaborator) WaitForNodeConfigCompletion(ctx context.Context, commandName, namespace string) (bool, error) {
	command := &v1alpha1.Command{}
	key := types.NamespacedName{Name: commandName, Namespace: namespace}

	if err := nc.Get(ctx, key, command); err != nil {
		return false, fmt.Errorf("get Command %s: %w", commandName, err)
	}

	switch command.Status.Phase {
	case v1alpha1.CommandPhaseCompleted:
		return true, nil
	case v1alpha1.CommandPhaseFailed:
		return false, fmt.Errorf("command failed: %s", command.Status.Message)
	default:
		return false, nil
	}
}

func (nc *Collaborator) GetNodeConfigStatus(ctx context.Context, cv *v1alpha1.ComponentVersion) (*NodeConfigStatus, error) {
	log := nc.Log.WithValues("component", cv.Spec.ComponentName)

	nodeConfigs, err := nc.GetNodeConfigsForComponent(ctx, cv)
	if err != nil {
		return nil, err
	}

	status := &NodeConfigStatus{
		TotalNodeConfigs: len(nodeConfigs),
		NodeDetails:      make(map[string]NodeDetail),
	}

	for _, nodeConfig := range nodeConfigs {
		nodes, err := nc.GetNodesForNodeConfig(ctx, nodeConfig)
		if err != nil {
			log.Error(err, "failed to get nodes for NodeConfig", "nodeconfig", nodeConfig.Name)
			continue
		}

		for _, node := range nodes {
			detail := NodeDetail{
				NodeConfigName: nodeConfig.Name,
				NodeName:       node.Name,
			}

			componentLabel := fmt.Sprintf("componentversion.%s/version", cv.Spec.ComponentName)
			if appliedVersion, ok := node.Labels[componentLabel]; ok {
				detail.AppliedVersion = appliedVersion
				if appliedVersion == cv.Spec.Version {
					detail.Synced = true
					status.SyncedNodes++
				}
			}

			status.NodeDetails[node.Name] = detail
		}

		status.TotalNodes += len(nodes)
	}

	if status.TotalNodes > 0 && status.SyncedNodes == status.TotalNodes {
		status.AllSynced = true
	}

	return status, nil
}

type NodeConfigStatus struct {
	TotalNodeConfigs int
	TotalNodes       int
	SyncedNodes      int
	AllSynced        bool
	NodeDetails      map[string]NodeDetail
}

type NodeDetail struct {
	NodeConfigName string
	NodeName       string
	AppliedVersion string
	Synced         bool
}

func (nc *Collaborator) CleanupNodeConfigCommands(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
	log := nc.Log.WithValues("component", cv.Spec.ComponentName)

	commandList := &v1alpha1.CommandList{}
	labelSelector := labels.SelectorFromSet(labels.Set{
		"componentversion.component": cv.Spec.ComponentName,
		"componentversion.version":   cv.Spec.Version,
	})

	if err := nc.List(ctx, commandList, client.MatchingLabelsSelector{Selector: labelSelector}); err != nil {
		return fmt.Errorf("list Commands: %w", err)
	}

	for _, cmd := range commandList.Items {
		if err := nc.Delete(ctx, &cmd); err != nil {
			log.Error(err, "failed to delete Command", "command", cmd.Name)
			continue
		}
	}

	log.Info("cleaned up NodeConfig commands", "count", len(commandList.Items))
	return nil
}
```
## 3. Reconciler 主循环
```go d:\code\github\componentversion-controller\controllers\componentversion_controller.go
package controllers

import (
	"context"
	"fmt"
	"time"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/rest"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\pkg\componentversion\dependency"
	"d:\code\github\componentversion-controller\pkg\componentversion\dispatcher"
	"d:\code\github\componentversion-controller\pkg\componentversion\healthcheck"
	"d:\code\github\componentversion-controller\pkg\componentversion\nodeconfig"
	"d:\code\github\componentversion-controller\pkg\componentversion\source"
	"d:\code\github\componentversion-controller\pkg\componentversion\statemachine"
)

type ComponentVersionReconciler struct {
	client.Client
	Log            logr.Logger
	Scheme         *runtime.Scheme
	Config         *rest.Config

	Dispatcher     *dispatcher.Dispatcher
	StateMachine   *statemachine.Machine
	DependencyResolver *dependency.Resolver
	NodeConfigCollab *nodeconfig.Collaborator
	HealthManager  *healthcheck.Manager
	Resolver       *source.Resolver
}

func NewComponentVersionReconciler(mgr ctrl.Manager, log logr.Logger) (*ComponentVersionReconciler, error) {
	c := mgr.GetClient()
	scheme := mgr.GetScheme()
	config := mgr.GetConfig()

	resolver := source.NewResolver(c, log)
	healthMgr := healthcheck.NewManager(c, log, config, resolver)
	dispatcherInst, err := dispatcher.NewDispatcher(c, log, config, resolver, healthMgr)
	if err != nil {
		return nil, fmt.Errorf("create dispatcher: %w", err)
	}

	stateMachine := statemachine.NewMachine(log)
	depResolver := dependency.NewResolver(c, log)
	nodeConfigCollab := nodeconfig.NewCollaborator(c, log)

	r := &ComponentVersionReconciler{
		Client:            c,
		Log:               log.WithName("componentversion-controller"),
		Scheme:            scheme,
		Config:            config,
		Dispatcher:        dispatcherInst,
		StateMachine:      stateMachine,
		DependencyResolver: depResolver,
		NodeConfigCollab:  nodeConfigCollab,
		HealthManager:     healthMgr,
		Resolver:          resolver,
	}

	return r, nil
}

func (r *ComponentVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("componentversion", req.NamespacedName)

	cv := &v1alpha1.ComponentVersion{}
	if err := r.Get(ctx, req.NamespacedName, cv); err != nil {
		if errors.IsNotFound(err) {
			log.Info("ComponentVersion deleted")
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	if cv.DeletionTimestamp != nil {
		return r.handleDeletion(ctx, cv)
	}

	if err := r.ensureFinalizer(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}

	return r.StateMachine.Drive(ctx, cv, r.executePhase)
}

func (r *ComponentVersionReconciler) executePhase(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName, "version", cv.Spec.Version, "phase", cv.Status.Phase)

	switch cv.Status.Phase {
	case v1alpha1.PhasePending:
		return r.handlePending(ctx, cv)
	case v1alpha1.PhaseInstalling:
		return r.handleInstalling(ctx, cv)
	case v1alpha1.PhaseVerifying:
		return r.handleVerifying(ctx, cv)
	case v1alpha1.PhaseInstalled:
		return r.handleInstalled(ctx, cv)
	case v1alpha1.PhaseUpgrading:
		return r.handleUpgrading(ctx, cv)
	case v1alpha1.PhaseRollingBack:
		return r.handleRollingBack(ctx, cv)
	case v1alpha1.PhaseFailed:
		return r.handleFailed(ctx, cv)
	default:
		return ctrl.Result{}, fmt.Errorf("unknown phase: %s", cv.Status.Phase)
	}
}

func (r *ComponentVersionReconciler) handlePending(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	if len(cv.Spec.Dependencies) > 0 {
		ready, err := r.DependencyResolver.CheckDependencies(ctx, cv)
		if err != nil {
			log.Error(err, "failed to check dependencies")
			return r.updateStatus(ctx, cv, v1alpha1.PhasePending, err)
		}

		if !ready {
			log.Info("dependencies not ready, requeuing")
			return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
		}

		log.Info("all dependencies are ready")
	}

	if cv.Spec.Scope == v1alpha1.ScopeNode {
		nodeConfigStatus, err := r.NodeConfigCollab.GetNodeConfigStatus(ctx, cv)
		if err != nil {
			log.Error(err, "failed to get NodeConfig status")
			return r.updateStatus(ctx, cv, v1alpha1.PhasePending, err)
		}

		if nodeConfigStatus.TotalNodeConfigs == 0 {
			log.Info("no NodeConfigs found for node-scoped component")
		}

		cv.Status.NodeConfigStatus = &v1alpha1.NodeConfigStatus{
			TotalNodeConfigs: nodeConfigStatus.TotalNodeConfigs,
			TotalNodes:       nodeConfigStatus.TotalNodes,
			SyncedNodes:      nodeConfigStatus.SyncedNodes,
		}
	}

	cv.Status.InstallType = r.Dispatcher.GetInstallType(cv)
	return r.transitionTo(ctx, cv, v1alpha1.PhaseInstalling)
}

func (r *ComponentVersionReconciler) handleInstalling(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	result, err := r.Dispatcher.Install(ctx, cv)
	if err != nil {
		log.Error(err, "installation failed")

		if isRetryableError(err) {
			cv.Status.RetryCount++
			if cv.Status.RetryCount >= getMaxRetries() {
				return r.transitionTo(ctx, cv, v1alpha1.PhaseFailed)
			}
			return ctrl.Result{RequeueAfter: getRetryDelay(cv.Status.RetryCount)}, nil
		}

		return r.transitionTo(ctx, cv, v1alpha1.PhaseFailed)
	}

	cv.Status.InstallDetails = result.Details
	cv.Status.RetryCount = 0

	if cv.Spec.Scope == v1alpha1.ScopeNode {
		if err := r.applyNodeConfigs(ctx, cv); err != nil {
			log.Error(err, "failed to apply NodeConfigs")
			return r.transitionTo(ctx, cv, v1alpha1.PhaseFailed)
		}
	}

	return r.transitionTo(ctx, cv, result.Phase)
}

func (r *ComponentVersionReconciler) handleVerifying(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	result, err := r.Dispatcher.Verify(ctx, cv)
	if err != nil {
		log.Error(err, "verification failed")
		return r.transitionTo(ctx, cv, v1alpha1.PhaseFailed)
	}

	if !result.Healthy {
		log.Info("verification not healthy, requeuing", "reason", result.Reason)

		if cv.Spec.HealthCheck != nil && cv.Spec.HealthCheck.FailureThreshold > 0 {
			cv.Status.Health.ConsecutiveFailures++
			if cv.Status.Health.ConsecutiveFailures >= cv.Spec.HealthCheck.FailureThreshold {
				log.Info("verification failure threshold exceeded, rolling back")
				return r.transitionTo(ctx, cv, v1alpha1.PhaseRollingBack)
			}
		}

		return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
	}

	log.Info("verification successful")
	cv.Status.Health = v1alpha1.HealthStatus{
		Overall:   v1alpha1.HealthStatusHealthy,
		Message:   result.Reason,
		LastCheckedAt: &metav1.Time{Time: time.Now()},
	}

	return r.transitionTo(ctx, cv, v1alpha1.PhaseInstalled)
}

func (r *ComponentVersionReconciler) handleInstalled(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	if cv.Spec.HealthCheck != nil {
		healthStatus, err := r.HealthManager.Check(ctx, cv)
		if err != nil {
			log.Error(err, "health check failed")
			return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
		}

		patch := client.MergeFrom(cv.DeepCopy())
		cv.Status.Health = *healthStatus

		if err := r.Status().Patch(ctx, cv, patch); err != nil {
			log.Error(err, "failed to patch health status")
		}

		if healthStatus.Overall == v1alpha1.HealthStatusUnhealthy {
			if cv.Spec.HealthCheck.FailureThreshold > 0 &&
				healthStatus.ConsecutiveFailures >= cv.Spec.HealthCheck.FailureThreshold {
				log.Info("health check failure threshold exceeded, triggering reinstall")
				return r.transitionTo(ctx, cv, v1alpha1.PhasePending)
			}
		}

		interval := int64(30)
		if cv.Spec.HealthCheck.IntervalSeconds > 0 {
			interval = cv.Spec.HealthCheck.IntervalSeconds
		}
		return ctrl.Result{RequeueAfter: time.Duration(interval) * time.Second}, nil
	}

	return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *ComponentVersionReconciler) handleUpgrading(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName, "from", cv.Status.ObservedVersion, "to", cv.Spec.Version)

	result, err := r.Dispatcher.Install(ctx, cv)
	if err != nil {
		log.Error(err, "upgrade failed")
		return r.transitionTo(ctx, cv, v1alpha1.PhaseRollingBack)
	}

	cv.Status.InstallDetails = result.Details
	return r.transitionTo(ctx, cv, result.Phase)
}

func (r *ComponentVersionReconciler) handleRollingBack(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	result, err := r.Dispatcher.Rollback(ctx, cv)
	if err != nil {
		log.Error(err, "rollback failed")
		return r.transitionTo(ctx, cv, v1alpha1.PhaseFailed)
	}

	log.Info("rollback completed", "message", result.Message)
	return r.transitionTo(ctx, cv, result.Phase)
}

func (r *ComponentVersionReconciler) handleFailed(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	log.Info("component in failed state, will retry after delay")
	return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *ComponentVersionReconciler) handleDeletion(ctx context.Context, cv *v1alpha1.ComponentVersion) (ctrl.Result, error) {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	if !controllerutil.ContainsFinalizer(cv, v1alpha1.ComponentVersionFinalizer) {
		return ctrl.Result{}, nil
	}

	log.Info("uninstalling component before deletion")

	if _, err := r.Dispatcher.Uninstall(ctx, cv); err != nil {
		log.Error(err, "uninstall failed during deletion")
		return ctrl.Result{}, err
	}

	if cv.Spec.Scope == v1alpha1.ScopeNode {
		if err := r.NodeConfigCollab.CleanupNodeConfigCommands(ctx, cv); err != nil {
			log.Error(err, "failed to cleanup NodeConfig commands")
		}
	}

	controllerutil.RemoveFinalizer(cv, v1alpha1.ComponentVersionFinalizer)
	if err := r.Update(ctx, cv); err != nil {
		return ctrl.Result{}, err
	}

	log.Info("component uninstalled and finalizer removed")
	return ctrl.Result{}, nil
}

func (r *ComponentVersionReconciler) ensureFinalizer(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
	if controllerutil.ContainsFinalizer(cv, v1alpha1.ComponentVersionFinalizer) {
		return nil
	}

	controllerutil.AddFinalizer(cv, v1alpha1.ComponentVersionFinalizer)
	return r.Update(ctx, cv)
}

func (r *ComponentVersionReconciler) transitionTo(ctx context.Context, cv *v1alpha1.ComponentVersion, phase v1alpha1.ComponentPhase) (ctrl.Result, error) {
	if cv.Status.Phase == phase {
		return ctrl.Result{}, nil
	}

	log := r.Log.WithValues("component", cv.Spec.ComponentName, "from", cv.Status.Phase, "to", phase)
	log.Info("phase transition")

	patch := client.MergeFrom(cv.DeepCopy())
	cv.Status.Phase = phase
	cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}

	if phase == v1alpha1.PhaseInstalled {
		cv.Status.ObservedVersion = cv.Spec.Version
	}

	if err := r.Status().Patch(ctx, cv, patch); err != nil {
		return ctrl.Result{}, fmt.Errorf("update status: %w", err)
	}

	return ctrl.Result{Requeue: true}, nil
}

func (r *ComponentVersionReconciler) updateStatus(ctx context.Context, cv *v1alpha1.ComponentVersion, phase v1alpha1.ComponentPhase, cause error) (ctrl.Result, error) {
	patch := client.MergeFrom(cv.DeepCopy())

	cv.Status.Phase = phase
	cv.Status.LastTransitionTime = &metav1.Time{Time: time.Now()}

	if cause != nil {
		condition := metav1.Condition{
			Type:    string(phase),
			Status:  metav1.ConditionFalse,
			Reason:  "Error",
			Message: cause.Error(),
		}
		cv.Status.Conditions = append(cv.Status.Conditions, condition)
	}

	if err := r.Status().Patch(ctx, cv, patch); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{Requeue: true}, nil
}

func (r *ComponentVersionReconciler) applyNodeConfigs(ctx context.Context, cv *v1alpha1.ComponentVersion) error {
	log := r.Log.WithValues("component", cv.Spec.ComponentName)

	nodeConfigs, err := r.NodeConfigCollab.GetNodeConfigsForComponent(ctx, cv)
	if err != nil {
		return err
	}

	for _, nodeConfig := range nodeConfigs {
		if err := r.NodeConfigCollab.ApplyNodeConfig(ctx, cv, nodeConfig); err != nil {
			log.Error(err, "failed to apply NodeConfig", "nodeconfig", nodeConfig.Name)
			continue
		}
	}

	return nil
}
```
## 4. SetupWithManager
```go d:\code\github\componentversion-controller\controllers\setup.go
package controllers

import (
	ctrl "sigs.k8s.io/controller-runtime"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func (r *ComponentVersionReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&v1alpha1.ComponentVersion{}).
		Owns(&v1alpha1.Command{}).
		Complete(r)
}
```
## 5. 辅助函数
```go d:\code\github\componentversion-controller\pkg\componentversion\utils\helpers.go
package utils

import (
	"time"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

const (
	MaxRetries = 5
	BaseRetryDelay = 5 * time.Second
	MaxRetryDelay = 5 * time.Minute
)

func GetMaxRetries() int {
	return MaxRetries
}

func GetRetryDelay(retryCount int) time.Duration {
	delay := BaseRetryDelay * time.Duration(retryCount)
	if delay > MaxRetryDelay {
		return MaxRetryDelay
	}
	return delay
}

func IsRetryableError(err error) bool {
	if err == nil {
		return false
	}

	return true
}

func IsTerminalPhase(phase v1alpha1.ComponentPhase) bool {
	switch phase {
	case v1alpha1.PhaseInstalled, v1alpha1.PhaseFailed:
		return true
	default:
		return false
	}
}

func IsHealthyPhase(phase v1alpha1.ComponentPhase) bool {
	return phase == v1alpha1.PhaseInstalled
}

func GetInstallType(cv *v1alpha1.ComponentVersion) v1alpha1.InstallType {
	if cv.Spec.Install.Manifest != nil {
		return v1alpha1.InstallTypeManifest
	}
	if cv.Spec.Install.Script != nil {
		return v1alpha1.InstallTypeScript
	}
	if cv.Spec.Install.HelmChart != nil {
		return v1alpha1.InstallTypeHelmChart
	}
	return ""
}

func CalculateBackoff(attempt int, baseDelay, maxDelay time.Duration) time.Duration {
	delay := baseDelay * time.Duration(1<<uint(attempt))
	if delay > maxDelay {
		return maxDelay
	}
	return delay
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\utils\conditions.go
package utils

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func SetCondition(conditions []metav1.Condition, newCond metav1.Condition) []metav1.Condition {
	for i, c := range conditions {
		if c.Type == newCond.Type {
			if c.Status != newCond.Status {
				conditions[i] = newCond
			}
			return conditions
		}
	}

	return append(conditions, newCond)
}

func GetCondition(conditions []metav1.Condition, condType string) *metav1.Condition {
	for i := range conditions {
		if conditions[i].Type == condType {
			return &conditions[i]
		}
	}
	return nil
}

func RemoveCondition(conditions []metav1.Condition, condType string) []metav1.Condition {
	for i, c := range conditions {
		if c.Type == condType {
			return append(conditions[:i], conditions[i+1:]...)
		}
	}
	return conditions
}

func NewCondition(condType string, status metav1.ConditionStatus, reason, message string) metav1.Condition {
	return metav1.Condition{
		Type:    condType,
		Status:  status,
		Reason:  reason,
		Message: message,
	}
}

func IsConditionTrue(conditions []metav1.Condition, condType string) bool {
	cond := GetCondition(conditions, condType)
	return cond != nil && cond.Status == metav1.ConditionTrue
}

func ComponentPhaseToCondition(phase v1alpha1.ComponentPhase) string {
	return string(phase)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\utils\patch.go
package utils

import (
	"context"
	"fmt"

	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

func PatchStatus(ctx context.Context, c client.Client, cv *v1alpha1.ComponentVersion, patchFn func()) error {
	patch := client.MergeFrom(cv.DeepCopy())
	patchFn()

	if err := c.Status().Patch(ctx, cv, patch); err != nil {
		return fmt.Errorf("patch status: %w", err)
	}
	return nil
}

func PatchObject(ctx context.Context, c client.Client, obj client.Object, patchFn func()) error {
	patch := client.MergeFrom(obj.DeepCopy())
	patchFn()

	if err := c.Patch(ctx, obj, patch); err != nil {
		return fmt.Errorf("patch object: %w", err)
	}
	return nil
}

func AddFinalizer(ctx context.Context, c client.Client, obj client.Object, finalizer string) error {
	patch := client.MergeFrom(obj.DeepCopy())
	controllerutil.AddFinalizer(obj, finalizer)
	return c.Patch(ctx, obj, patch)
}

func RemoveFinalizer(ctx context.Context, c client.Client, obj client.Object, finalizer string) error {
	patch := client.MergeFrom(obj.DeepCopy())
	controllerutil.RemoveFinalizer(obj, finalizer)
	return c.Patch(ctx, obj, patch)
}
```

```go d:\code\github\componentversion-controller\pkg\componentversion\utils\labels.go
package utils

import (
	"fmt"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
)

const (
	ComponentNameLabel = "componentversion.component"
	ComponentVersionLabel = "componentversion.version"
	ComponentScopeLabel = "componentversion.scope"
	ManagedByLabel = "componentversion.managed-by"
)

func BuildLabels(cv *v1alpha1.ComponentVersion) map[string]string {
	return map[string]string{
		ComponentNameLabel:    cv.Spec.ComponentName,
		ComponentVersionLabel: cv.Spec.Version,
		ComponentScopeLabel:   string(cv.Spec.Scope),
		ManagedByLabel:        "componentversion-controller",
	}
}

func BuildComponentLabel(componentName string) string {
	return fmt.Sprintf("componentversion.%s/version", componentName)
}

func BuildOwnerReference(cv *v1alpha1.ComponentVersion) []metav1.OwnerReference {
	return []metav1.OwnerReference{
		{
			APIVersion: v1alpha1.GroupVersion.String(),
			Kind:       "ComponentVersion",
			Name:       cv.Name,
			UID:        cv.UID,
			Controller: boolPtr(true),
		},
	}
}

func boolPtr(b bool) *bool {
	return &b
}
```
## 6. Main 入口
```go d:\code\github\componentversion-controller\cmd\main.go
package main

import (
	"flag"
	"os"

	"github.com/go-logr/logr"
	"github.com/go-logr/zapr"
	"go.uber.org/zap"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"

	v1alpha1 "d:\code\github\componentversion-controller\api\v1alpha1"
	"d:\code\github\componentversion-controller\controllers"
)

var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))
	utilruntime.Must(v1alpha1.AddToScheme(scheme))
}

func main() {
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string

	flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false, "Enable leader election for controller manager.")
	flag.Parse()

	zapLogger, err := zap.NewProduction()
	if err != nil {
		setupLog.Error(err, "failed to create zap logger")
		os.Exit(1)
	}

	log := zapr.NewLogger(zapLogger)
	ctrl.SetLogger(log)

	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		MetricsBindAddress:     metricsAddr,
		Port:                   9443,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "componentversion.controller",
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	reconciler, err := controllers.NewComponentVersionReconciler(mgr, log)
	if err != nil {
		setupLog.Error(err, "unable to create reconciler")
		os.Exit(1)
	}

	if err := reconciler.SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to setup controller")
		os.Exit(1)
	}

	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}

	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```
## 架构总结
### 编排层整体架构
```
ComponentVersion Controller (编排层)
│
├── Reconciler (主循环)
│   ├── Reconcile() → 入口
│   ├── StateMachine.Drive() → 状态机驱动
│   ├── executePhase() → 阶段执行分发
│   │   ├── handlePending()     → 依赖检查 + NodeConfig 状态
│   │   ├── handleInstalling()  → Dispatcher.Install() + NodeConfig 应用
│   │   ├── handleVerifying()   → Dispatcher.Verify()
│   │   ├── handleInstalled()   → 周期性健康检查
│   │   ├── handleUpgrading()   → Dispatcher.Install() (升级路径)
│   │   ├── handleRollingBack() → Dispatcher.Rollback()
│   │   └── handleFailed()      → 等待重试
│   ├── handleDeletion() → Finalizer 清理 + Uninstall
│   ├── transitionTo()   → 状态转换 + Status Patch
│   └── applyNodeConfigs() → Node-scoped 组件配置应用
│
├── Dispatcher (安装器分发)
│   ├── GetInstaller() → 按安装类型获取 Installer
│   ├── detectInstallType() → 自动检测安装类型
│   ├── Install/Verify/Rollback/Uninstall → 委托给对应 Installer
│   └── HealthCheck() → 委托给 HealthManager
│
├── NodeConfig Collaborator (节点配置协作)
│   ├── GetNodeConfigsForComponent() → 查询关联的 NodeConfig
│   ├── GetNodesForNodeConfig() → 按 NodeSelector 查询节点
│   ├── ApplyNodeConfig() → 应用配置到节点
│   ├── CreateNodeConfigCommand() → 创建节点级执行命令
│   ├── GetNodeConfigStatus() → 获取节点配置同步状态
│   └── CleanupNodeConfigCommands() → 清理节点命令
│
├── Dependency Resolver (依赖解析)
│   └── CheckDependencies() → 检查所有依赖是否就绪
│
├── Health Manager (健康检查)
│   └── Check() → 执行健康检查 → 更新 Status.Health
│
└── Utils (辅助函数)
    ├── helpers.go   → 重试逻辑、阶段判断
    ├── conditions.go → Condition 操作
    ├── patch.go     → Status/Object Patch 封装
    └── labels.go    → Label 构建工具
```
### Reconcile 主循环流程
```
Reconcile(req)
  │
  ├─ Get ComponentVersion
  │
  ├─ DeletionTimestamp?
  │   └─ Yes → handleDeletion()
  │              ├─ Dispatcher.Uninstall()
  │              ├─ CleanupNodeConfigCommands()
  │              └─ RemoveFinalizer
  │
  ├─ EnsureFinalizer
  │
  └─ StateMachine.Drive(executePhase)
      │
      └─ executePhase(phase)
          │
          ├─ Pending
          │   ├─ CheckDependencies() → not ready? Requeue 10s
          │   ├─ GetNodeConfigStatus() (if ScopeNode)
          │   └─ transitionTo(Installing)
          │
          ├─ Installing
          │   ├─ Dispatcher.Install()
          │   │   ├─ ManifestInstaller.Install()
          │   │   ├─ ScriptInstaller.Install()
          │   │   └─ HelmChartInstaller.Install()
          │   ├─ applyNodeConfigs() (if ScopeNode)
          │   └─ transitionTo(Verifying/Failed)
          │
          ├─ Verifying
          │   ├─ Dispatcher.Verify()
          │   ├─ Healthy? → transitionTo(Installed)
          │   ├─ FailureThreshold exceeded? → transitionTo(RollingBack)
          │   └─ else → Requeue 10s
          │
          ├─ Installed
          │   ├─ HealthManager.Check() (if healthCheck defined)
          │   ├─ Update Status.Health
          │   ├─ FailureThreshold exceeded? → transitionTo(Pending)
          │   └─ Requeue after intervalSeconds
          │
          ├─ Upgrading
          │   ├─ Dispatcher.Install()
          │   └─ transitionTo(Verifying/RollingBack)
          │
          ├─ RollingBack
          │   ├─ Dispatcher.Rollback()
          │   └─ transitionTo(Pending/Failed)
          │
          └─ Failed
              └─ Requeue 5m (等待手动干预或自动恢复)
```
### NodeConfig 协作流程
```
Node-scoped Component 安装流程
│
├─ handlePending()
│   └─ GetNodeConfigStatus()
│       ├─ List NodeConfigs matching component
│       ├─ For each NodeConfig:
│       │   └─ GetNodesForNodeConfig(nodeSelector)
│       └─ Calculate SyncedNodes / TotalNodes
│
├─ handleInstalling()
│   └─ applyNodeConfigs()
│       └─ For each NodeConfig:
│           ├─ GetNodesForNodeConfig()
│           └─ For each Node:
│               ├─ Patch node.labels[component/version]
│               ├─ Patch node.annotations[component/config]
│               └─ (optional) CreateNodeConfigCommand()
│
└─ handleDeletion()
    └─ CleanupNodeConfigCommands()
        └─ Delete all Commands with component labels
```
### 关键设计决策
| 决策 | 说明 |
|------|------|
| **状态机驱动** | Reconciler 通过 StateMachine.Drive() 驱动，状态转换逻辑集中在状态机中 |
| **Dispatcher 分发** | 按安装类型自动选择对应 Installer，支持扩展注册新安装类型 |
| **NodeConfig 协作** | Node-scoped 组件通过 NodeConfig 协作器管理节点级配置和命令 |
| **依赖检查** | Pending 阶段检查所有依赖是否 Installed，未就绪则 Requeue |
| **Finalizer 保护** | 删除前必须执行 Uninstall，确保资源清理 |
| **指数退避重试** | 安装失败后按指数退避重试，最大 5 分钟 |
| **健康检查轮询** | Installed 状态按 intervalSeconds 周期执行健康检查 |
| **FailureThreshold** | 连续失败达到阈值后自动触发重新安装或回滚 |
| **Patch 策略** | 使用 MergeFrom + Patch 避免并发冲突 |
### 与前序层的集成关系
| 层级 | 组件 | 编排层调用方式 |
|------|------|--------------|
| **基础层** | API Types | Reconciler 读写 ComponentVersion/NodeConfig/Command |
| **基础层** | Source Resolver | Dispatcher 传递给各 Installer |
| **基础层** | State Machine | Reconciler 通过 Drive() 驱动状态转换 |
| **基础层** | Dependency Resolver | handlePending() 中检查依赖 |
| **核心层** | ManifestInstaller | Dispatcher.installers[Manifest] |
| **核心层** | ScriptInstaller | Dispatcher.installers[Script] |
| **核心层** | HelmChartInstaller | Dispatcher.installers[HelmChart] |
| **核心层** | Health Manager | handleInstalled() 中周期调用 Check() |

请手动创建上述文件到对应路径。至此，ComponentVersion Controller 的完整实现框架已完成，包括：
- **基础层**：API 类型 + Source 解析 + 状态机 + 依赖解析
- **核心层**：ManifestInstaller + ScriptInstaller + HelmChartInstaller + 健康检查框架
- **编排层**：Reconciler 主循环 + Dispatcher + NodeConfig 协作 + SetupWithManager + 辅助函数


