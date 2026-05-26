# Cluster API Runtime Hooks

## Cluster API Runtime Hooks 功能列表及开发指南

### 一、Runtime Hooks 功能列表
Runtime Hooks 共 **17 个 Hook**，分为 5 大类：

#### 1. Discovery Hook（1个）
| Hook | 用途 |
|------|------|
| `Discovery` | ExtensionConfig 被 reconciled 时调用，扩展通过此 Hook 向 CAPI runtime 注册自己支持的所有 handlers |

#### 2. Lifecycle Hooks（9个）
| Hook | 触发时机 | 是否阻塞 |
|------|----------|----------|
| `BeforeClusterCreate` | Cluster 创建后，topology 对象创建前 | 是 |
| `AfterControlPlaneInitialized` | Control Plane 首次可达时 | 否 |
| `BeforeClusterUpgrade` | Cluster 版本升级传播前 | 是 |
| `BeforeControlPlaneUpgrade` | Control Plane 版本升级前 | 是 |
| `AfterControlPlaneUpgrade` | Control Plane 升级成功后 | 是 |
| `BeforeWorkersUpgrade` | Workers (MachineDeployments/MachinePools) 升级前 | 是 |
| `AfterWorkersUpgrade` | Workers 升级成功后 | 是 |
| `AfterClusterUpgrade` | 整个集群升级到目标版本后 | 是 |
| `BeforeClusterDelete` | Cluster 删除触发后，对象删除前 | 是 |

#### 3. Topology Mutation Hooks（3个）
| Hook | 用途 |
|------|------|
| `GeneratePatches` | topology reconcile 期间对模板生成 patches |
| `ValidateTopology` | patches 应用后验证 Cluster topology |
| `DiscoverVariables` | ClusterClass reconcile 期间返回扩展定义的变量 schemas |

#### 4. In-Place Update Hooks（3个）
| Hook | 用途 |
|------|------|
| `CanUpdateMachine` | 更新规划期间判断扩展是否能处理 Machine 的就地变更 |
| `CanUpdateMachineSet` | 更新规划期间判断扩展是否能处理 MachineSet 的就地变更 |
| `UpdateMachine` | 对 Machine 执行实际的就地更新（幂等，可重复调用） |

#### 5. Upgrade Plan Hook（1个）
| Hook | 用途 |
|------|------|
| `GenerateUpgradePlan` | 生成升级计划，支持链式升级的中间 Kubernetes 版本 |

### 二、核心架构
```
┌─────────────────────────────────────────────────────────────┐
│                    CAPI Controllers                         │
│  (cluster controller, topology controller, etc.)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Runtime Client                             │
│  (internal/runtime/client/client.go)                        │
│  - CallAllExtensions() / CallExtension()                    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP POST (HTTPS)
                         │ /{group}/{version}/{hook}/{name}
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Extension Server (Your Code)                   │
│  (exp/runtime/server/server.go)                             │
│  - 接收请求，执行业务逻辑，返回响应                            │
└─────────────────────────────────────────────────────────────┘
```
**关键组件文件：**
- Hook 定义：`api/runtime/hooks/v1alpha1/`
- Catalog 注册：`exp/runtime/catalog/catalog.go`
- Server 实现：`exp/runtime/server/server.go`
- Client 实现：`internal/runtime/client/client.go`
- Registry：`internal/runtime/registry/registry.go`
- ExtensionConfig CRD：`api/runtime/v1beta2/extensionconfig_types.go`

### 三、开发指南

#### 步骤 1：创建 Extension Server
```go
package main

import (
    "context"
    ctrl "sigs.k8s.io/controller-runtime"
    runtimehooksv1 "sigs.k8s.io/cluster-api/api/runtime/hooks/v1alpha1"
    runtimecatalog "sigs.k8s.io/cluster-api/exp/runtime/catalog"
    "sigs.k8s.io/cluster-api/exp/runtime/server"
)

var catalog = runtimecatalog.New()

func init() {
    _ = runtimehooksv1.AddToCatalog(catalog)
}

func main() {
    webhookServer, _ := server.New(server.Options{
        Catalog: catalog,
        Port:    9443,
        CertDir: "/tmp/k8s-webhook-server/serving-certs/",
    })

    // 注册 handler
    webhookServer.AddExtensionHandler(server.ExtensionHandler{
        Hook:        runtimehooksv1.BeforeClusterCreate,
        Name:        "before-cluster-create",
        HandlerFunc: DoBeforeClusterCreate,
    })

    ctx := ctrl.SetupSignalHandler()
    webhookServer.Start(ctx)
}

func DoBeforeClusterCreate(ctx context.Context,
    request *runtimehooksv1.BeforeClusterCreateRequest,
    response *runtimehooksv1.BeforeClusterCreateResponse) {

    log := ctrl.LoggerFrom(ctx)
    log.Info("BeforeClusterCreate called")

    // 你的业务逻辑
    response.Status = runtimehooksv1.ResponseStatusSuccess
}
```

#### 步骤 2：部署为 HTTPS 服务
- 必须使用 TLS 证书
- 可在集群内或集群外运行

#### 步骤 3：通过 ExtensionConfig CRD 注册
```yaml
apiVersion: runtime.cluster.x-k8s.io/v1beta2
kind: ExtensionConfig
metadata:
  annotations:
    runtime.cluster.x-k8s.io/inject-ca-from-secret: default/my-extension-cert
  name: my-extension
spec:
  clientConfig:
    service:
      name: my-extension-svc
      namespace: default
      port: 443
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
          - default
```

### 四、参考示例
- **测试 Extension 示例**：`test/extension/main.go`
- **Lifecycle handlers**：`test/extension/handlers/lifecycle/handlers.go`
- **Topology mutation handlers**：`test/extension/handlers/topologymutation/handler.go`
- **In-place update handlers**：`test/extension/handlers/inplaceupdate/handlers.go`
- **Upgrade plan handlers**：`test/extension/handlers/upgradeplan/handlers.go`

### 五、文档
- Runtime SDK 概览：`docs/book/src/tasks/experimental-features/runtime-sdk/index.md`
- 实现 Extensions：`docs/book/src/tasks/experimental-features/runtime-sdk/implement-extensions.md`
- Lifecycle Hooks 指南：`docs/book/src/tasks/experimental-features/runtime-sdk/implement-lifecycle-hooks.md`
- Topology Mutation 指南：`docs/book/src/tasks/experimental-features/runtime-sdk/implement-topology-mutation-hook.md`
- 提案文档：`docs/proposals/20220221-runtime-SDK.md`

## Cluster API Runtime Hooks 关键组件设计思路与功能说明

### 一、整体架构
Runtime Hooks 采用 **Client-Server 模型**，CAPI 作为 Client 调用外部 Extension Server，通过 HTTP/HTTPS 通信。系统由 4 个核心组件构成：
```
┌─────────────────────────────────────────────────────────────────┐
│                     CAPI Controllers                            │
│        (Cluster, Topology, MachineDeployment, etc.)             │
└────────────────────────────┬────────────────────────────────────┘
                             │ 调用
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Runtime Client                               │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   Catalog    │  │   Registry   │  │   HTTP Client         │  │
│  │  (类型系统)   │  │  (扩展注册表) │  │   (TLS/超时/重试)     │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS POST
                             │ /{group}/{version}/{hook}/{name}
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Extension Server (用户实现)                     │
│  ┌──────────────┐  ┌──────────────────────────────────────────┐ │
│  │   Catalog    │  │  Handlers: BeforeClusterCreate, ...      │ │
│  │  (类型验证)   │  │  用户业务逻辑                             │ │
│  └──────────────┘  └──────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 二、核心组件详解

#### 1. Catalog（类型目录）
**文件：** `exp/runtime/catalog/catalog.go`

**设计思路：**
- 作为 **中央类型注册表**，管理所有 Hook 的元数据
- 采用 **双向映射**：Go 函数 ↔ GroupVersionHook (GVH) ↔ GVK
- 基于 Kubernetes `runtime.Scheme` 实现版本转换

**核心数据结构：**
```go
type Catalog struct {
    scheme              *runtime.Scheme
    gvhToType           map[GroupVersionHook]reflect.Type   // GVH → 函数类型
    typeToGVH           map[reflect.Type]GroupVersionHook   // 函数类型 → GVH
    gvhToHookDescriptor map[GroupVersionHook]hookDescriptor // GVH → 元数据
}

type GroupVersionHook struct {
    Group   string  // "hooks.runtime.cluster.x-k8s.io"
    Version string  // "v1alpha1"
    Hook    string  // "BeforeClusterCreate"
}
```
**关键方法：**
| 方法 | 功能 |
|------|------|
| `AddHook(gv, hookFunc, meta)` | 注册 Hook，通过反射从 `func(*Req, *Resp)` 提取类型 |
| `GroupVersionHook(hookFunc)` | 反向查找：Go 函数 → GVH |
| `NewRequest(gvh)` / `NewResponse(gvh)` | 工厂方法，动态创建请求/响应对象 |
| `Convert(in, out, ctx)` | 不同版本间的类型转换 |
| `GVHToPath(gvh, name)` | GVH → HTTP 路径：`/{group}/{version}/{hook}[/{name}]` |

**设计模式：** 注册表模式 + 工厂模式 + 反射类型发现

#### 2. Builder（构建器）
**文件：** `exp/runtime/catalog/builder.go`

**设计思路：**
- 实现 **延迟注册**，允许 Hook 包在 `init()` 时注册，无需 Catalog 已存在
- 镜像 Kubernetes `SchemeBuilder` 模式

**核心结构：**
```go
type Builder struct {
    GroupVersion   schema.GroupVersion
    catalogBuilder []func(*Catalog)     // 延迟注册的闭包
    schemeBuilder  runtime.SchemeBuilder
}
```

**使用方式：**
```go
// 在 groupversion_info.go 中定义全局 Builder
var catalogBuilder = &runtimecatalog.Builder{GroupVersion: GroupVersion}
var AddToCatalog = catalogBuilder.AddToCatalog

// 在各 hook 类型文件中注册
func init() {
    catalogBuilder.RegisterHook(BeforeClusterCreate, &runtimecatalog.HookMeta{...})
}
```
**设计模式：** Builder 模式 + 延迟执行 + 函数式选项

#### 3. Server（扩展服务端）
**文件：** `exp/runtime/server/server.go`

**设计思路：**
- 为 Extension 开发者提供 **开箱即用的 Webhook 框架**
- 基于 `controller-runtime` webhook server 构建
- 自动注入 Discovery Handler，实现自描述 API

**核心结构：**
```go
type Server struct {
    webhook.Server
    catalog  *runtimecatalog.Catalog
    handlers map[string]ExtensionHandler  // path → handler 映射
}

type ExtensionHandler struct {
    Hook        runtimecatalog.Hook     // 处理的 Hook
    Name        string                  // Handler 名称
    HandlerFunc runtimecatalog.Hook     // func(context.Context, *Req, *Resp)
    TimeoutSeconds  *int32
    FailurePolicy   *runtimehooksv1.FailurePolicy
}
```

**关键流程：**
```
1. New() → 创建 Server，配置 TLS
2. AddExtensionHandler() → 注册 Handler，验证签名，计算 HTTP 路径
3. Start() → 自动添加 Discovery Handler，包装所有 Handlers，启动 HTTP Server
4. wrapHandler() → 反序列化请求 → 反射调用 Handler → 序列化响应
```

**Handler 签名约定：**
```go
// Hook 定义（catalog 中）：
func HookName(*RequestType, *ResponseType) {}

// Handler 实现（扩展中）：
func HandlerImpl(ctx context.Context, *RequestType, *ResponseType) {}
```
**设计模式：** 适配器模式 + Handler/Router 模式 + 模板方法模式

#### 4. Client（客户端接口）
**文件：** `exp/runtime/client/client.go`

**设计思路：**
- 定义 CAPI Controller 与扩展交互的 **公共契约**
- 接口隔离，职责清晰

**核心接口：**
```go
type Client interface {
    // 生命周期
    WarmUp(*runtimev1.ExtensionConfigList) error
    IsReady() bool

    // 扩展管理
    Discover(context.Context, *runtimev1.ExtensionConfig) (*runtimev1.ExtensionConfig, error)
    Register(*runtimev1.ExtensionConfig) error
    Unregister(*runtimev1.ExtensionConfig) error

    // Hook 调用
    CallAllExtensions(ctx, hook, forObject, request, response) error
    CallExtension(ctx, hook, forObject, name, request, response, opts...) error
    GetAllExtensions(ctx, hook, forObject) ([]string, error)
}
```

**选项模式：**
```go
type WithCaching struct {
    Cache        cache.Cache[CallExtensionCacheEntry]
    CacheKeyFunc func(extensionName, rv string, request) string
}
```

#### 5. Client Implementation（客户端实现）
**文件：** `internal/runtime/client/client.go`

**设计思路：**
- **编排 HTTP 通信**，实现版本转换、缓存、命名空间过滤
- 内置 **弹性模式**：超时、失败策略、重试

**核心结构：**
```go
type client struct {
    catalog  *runtimecatalog.Catalog
    registry runtimeregistry.ExtensionRegistry
    client   ctrlclient.Client           // K8s client（命名空间查找）
    httpClientsCache cache.Cache[httpClientEntry]  // TLS 配置缓存
}
```

**`CallExtension` 调用流程：**
```
1. Catalog.GroupVersionHook() → 解析 GVH
2. Catalog.ValidateRequest/Response() → 验证对象
3. Registry.Get(name) → 查找注册信息
4. matchNamespace() → 检查 NamespaceSelector
5. 检查缓存（如启用）
6. 构建 HTTP Client（TLS）
7. httpCall() → 执行请求
   ├── Catalog.Convert() → 转换为扩展支持的版本
   ├── HTTP POST（带超时）
   └── Catalog.Convert() → 转换回调用方版本
8. 应用 FailurePolicy
9. 缓存结果（如启用）
```

**`CallAllExtensions` 扇出流程：**
```
1. GetAllExtensions() → 获取所有匹配的扩展
2. 顺序调用每个扩展（快速失败）
3. 聚合响应
4. 合并 RetryAfterSeconds（取最小非零值）
```

**弹性机制：**
```go
// FailurePolicy 应用
if _, ok := err.(errCallingExtensionHandler); ok && ignore {
    response.SetStatus(runtimehooksv1.ResponseStatusSuccess)
    response.SetMessage("")
    return nil
}
```
**设计模式：** 外观模式 + 缓存旁路模式 + 策略模式（失败策略）

#### 6. Registry（扩展注册表）
**文件：** `internal/runtime/registry/registry.go`

**设计思路：**
- **内存级线程安全注册表**，映射 Handler 名称到元数据
- 实现 **预热模式**，确保注册表完全填充后才可用

**核心结构：**
```go
type ExtensionRegistration struct {
    Name                           string
    ExtensionConfigName            string
    GroupVersionHook               runtimecatalog.GroupVersionHook
    NamespaceSelector              labels.Selector
    ClientConfig                   runtimev1.ClientConfig
    TimeoutSeconds                 int32
    FailurePolicy                  runtimev1.FailurePolicy
    Settings                       map[string]string
}

type extensionRegistry struct {
    ready bool
    items map[string]*ExtensionRegistration  // name → registration
    lock  sync.RWMutex
}
```
**关键方法：**
| 方法 | 功能 |
|------|------|
| `WarmUp(list)` | 批量加载所有 ExtensionConfigs，原子操作，设置 `ready=true` |
| `Add(config)` | 添加/更新 Handler（先删除旧条目） |
| `Remove(config)` | 删除 ExtensionConfig 的所有 Handlers |
| `List(groupHook)` | 按 Group+Hook 过滤（忽略版本） |
| `Get(name)` | 按名称直接查找 |

**设计模式：** 读写锁模式 + 预热/初始化模式 + 全有或全无事务

#### 7. Common Types（公共类型）
**文件：** `api/runtime/hooks/v1alpha1/common_types.go`

**设计思路：**
- 定义所有 Hook 请求/响应必须实现的 **基础接口**
- 通过 **嵌入组合** 满足接口，遵循 Go 惯用法

**核心接口：**
```go
type RequestObject interface {
    runtime.Object
    GetSettings() map[string]string
    SetSettings(settings map[string]string)
}

type ResponseObject interface {
    runtime.Object
    GetMessage() string
    GetStatus() ResponseStatus
    SetMessage(message string)
    SetStatus(status ResponseStatus)
}

type RetryResponseObject interface {
    ResponseObject
    GetRetryAfterSeconds() int32
    SetRetryAfterSeconds(int32)
}
```

**公共嵌入类型：**
```go
type CommonRequest struct {
    Settings map[string]string `json:"settings,omitempty"`
}

type CommonResponse struct {
    Status   ResponseStatus `json:"status"`    // "Success" 或 "Failure"
    Message  string         `json:"message,omitempty"`
}

type CommonRetryResponse struct {
    CommonResponse    `json:",inline"`
    RetryAfterSeconds int32 `json:"retryAfterSeconds"`
}
```
**设计模式：** 接口隔离 + 组合优于继承 + 枚举模式

#### 8. ExtensionConfig CRD
**文件：** `api/runtime/v1beta2/extensionconfig_types.go`

**设计思路：**
- **Kubernetes 自定义资源**，代表已注册的 Runtime Extension
- CAPI 与外部扩展之间的 **配置契约**

**核心结构：**
```go
type ExtensionConfigSpec struct {
    ClientConfig      ClientConfig              // 连接方式（URL 或 Service）
    NamespaceSelector *metav1.LabelSelector     // 适用的命名空间
    Settings          map[string]string         // 传递给所有 Hook 的键值对
}

type ClientConfig struct {
    URL      string             // 外部 URL (https://...)
    Service  ServiceReference   // 集群内 Service 引用
    CABundle []byte             // TLS 验证的 CA 证书
}

type ExtensionConfigStatus struct {
    Conditions []metav1.Condition   // Discovered, Paused
    Handlers   []ExtensionHandler   // 从扩展发现的处理程序
}
```

**关键注解：**
```go
const (
    InjectCAFromSecretAnnotation = "runtime.cluster.x-k8s.io/inject-ca-from-secret"
    PendingHooksAnnotation       = "runtime.cluster.x-k8s.io/pending-hooks"
    OkToDeleteAnnotation         = "runtime.cluster.x-k8s.io/ok-to-delete"
)
```
**设计模式：** CRD 模式 + 多态连接 + Label Selector 作用域 + 自发现

### 三、组件协作流程

#### 注册流程
```
1. 用户创建 ExtensionConfig CR
   └── 包含 ClientConfig、NamespaceSelector、Settings

2. CAPI Controller 调用 Client.Discover()
   └── HTTP 调用扩展的 /discovery 端点
   └── 扩展 Server 返回 DiscoveryResponse（列出所有 Handlers）

3. Client 更新 ExtensionConfig.Status.Handlers

4. Client.Register() → Registry.Add() 解析 Handlers，创建注册条目
```

#### Hook 调用流程
```
1. Controller 调用 CallAllExtensions(hook, cluster, request, response)

2. Client 内部流程：
   ├── Catalog.GroupVersionHook() → 解析 GVH
   ├── Catalog.ValidateRequest/Response() → 验证对象
   ├── Registry.List(groupHook) → 获取所有匹配的注册
   └── 对每个注册：
       ├── Registry.Get(name) → 获取 ExtensionRegistration
       ├── matchNamespace() → 检查 NamespaceSelector
       ├── httpCall() → POST 到扩展 URL
       │   ├── Catalog.Convert() → 版本转换
       │   ├── HTTP POST（带超时）
       │   └── Catalog.Convert() → 响应版本转换
       ├── 应用 FailurePolicy
       └── 聚合响应

3. 返回响应给调用方
```

### 四、设计原则总结
| 原则 | 实现方式 |
|------|----------|
| **关注点分离** | Catalog（类型）、Registry（状态）、Client（编排）、Server（扩展侧） |
| **类型安全** | 注册时和调用时均通过反射验证 |
| **版本灵活** | Catalog.Convert() 支持跨版本通信 |
| **弹性设计** | 超时、FailurePolicy（Ignore/Fail）、retry-after 机制 |
| **可扩展性** | Builder 模式添加新 Hook；接口化 Client |
| **安全性** | mTLS（CABundle）、certwatcher 证书轮换 |
| **可观测性** | 指标（请求时长、总数）、Conditions、结构化日志 |
| **Kubernetes 原生** | CRD 配置、Label Selector、标准 Conditions |

## Catalog 与 Registry 的差异分析
**它们功能完全不同，不能互相替代，两者都是必需的。**

### 核心差异对比
| 维度 | **Catalog** | **Registry** |
|------|-------------|--------------|
| **职责** | Hook **类型定义**注册表 | Extension **实例**注册表 |
| **管理内容** | Hook 的元数据（函数签名、Request/Response 类型、GVK、OpenAPI） | 用户部署的 Extension 实例（URL、超时、FailurePolicy、NamespaceSelector） |
| **生命周期** | 编译期/启动期静态注册，**永不变化** | 运行时动态变化，随 ExtensionConfig CR 增删改 |
| **数据来源** | 代码中 `init()` 调用 `Builder.RegisterHook()` | 用户创建的 `ExtensionConfig` CRD 资源 |
| **典型操作** | `AddHook()`, `NewRequest()`, `Convert()`, `ValidateRequest()` | `Add()`, `Remove()`, `List()`, `Get()` |
| **类比** | 编程语言的 **类型系统/函数签名定义** | 运行时的 **服务发现/服务注册中心** |

### 具体差异说明

#### 1. Catalog 管理的是 **"什么 Hook 存在"**
```go
// Catalog 注册的是 Hook 定义（类型级别）
catalog.AddHook(BeforeClusterCreate, &HookMeta{
    Summary: "Called before cluster create",
    // ...
})

// 回答的问题：
// - BeforeClusterCreate 的请求类型是什么？ → BeforeClusterCreateRequest
// - 响应类型是什么？ → BeforeClusterCreateResponse
// - 如何序列化/反序列化？ → GVK 信息
// - 如何版本转换？ → scheme.Convert()
```

#### 2. Registry 管理的是 **"谁实现了这个 Hook"**
```go
// Registry 注册的是 Extension 实例（运行级别）
registry.Add(&ExtensionConfig{
    Spec: ExtensionConfigSpec{
        ClientConfig: ClientConfig{
            Service: &ServiceReference{Name: "my-extension-svc"},
        },
    },
    Status: ExtensionConfigStatus{
        Handlers: []ExtensionHandler{
            {Name: "my-before-create-hook", RequestHook: GroupVersionHook{...}},
        },
    },
})

// 回答的问题：
// - 谁实现了 BeforeClusterCreate？ → my-extension-svc
// - 如何连接？ → https://my-extension-svc.default.svc:443
// - 超时多久？ → 10s
// - 失败策略？ → Fail 或 Ignore
// - 适用于哪些命名空间？ → NamespaceSelector
```

### 协作关系
```
调用 Hook 时的完整流程：

1. Controller 调用 CallAllExtensions(BeforeClusterCreate, ...)
   
2. Client 使用 Catalog：
   ├── Catalog.GroupVersionHook(BeforeClusterCreate)
   │   └→ 得到 GVH: "hooks.runtime.cluster.x-k8s.io/v1alpha1/BeforeClusterCreate"
   ├── Catalog.NewRequest(gvh)
   │   └→ 创建 BeforeClusterCreateRequest 对象
   └── Catalog.ValidateRequest(gvh, request)
       └→ 验证请求类型正确

3. Client 使用 Registry：
   ├── Registry.List(GroupHook{"hooks.runtime.cluster.x-k8s.io", "BeforeClusterCreate"})
   │   └→ 得到所有实现该 Hook 的 Extension 列表
   │       [{Name: "my-extension", ClientConfig: {...}, TimeoutSeconds: 10, ...}]
   └── 对每个 Extension：
       └── 构建 HTTP 请求 → POST 到对应 URL

4. 收到响应后再次使用 Catalog：
   ├── Catalog.ValidateResponse(gvh, response)
   └── Catalog.Convert() (如需版本转换)
```

### 为什么不能合并为一个？
| 原因 | 说明 |
|------|------|
| **关注点分离** | Catalog 是类型系统（静态），Registry 是服务发现（动态），职责完全不同 |
| **生命周期不同** | Catalog 启动后不变；Registry 随 CR 动态变化 |
| **复用性** | Catalog 可被 Server 端（Extension 侧）独立使用，无需 Registry |
| **依赖方向** | Registry 依赖 Catalog（需要 GVH 信息），但 Catalog 不依赖 Registry |
| **线程安全需求** | Registry 需要 `sync.RWMutex` 处理并发读写；Catalog 只需读，无需锁 |

**结论：两者各司其职，缺一不可。**
