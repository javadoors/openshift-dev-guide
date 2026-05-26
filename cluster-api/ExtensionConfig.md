# ExtensionConfig

## ExtensionConfig 功能及开发指南

### 一、ExtensionConfig 是什么
**ExtensionConfig** 是 Cluster API 中用于 **注册 Runtime Extension** 的 Kubernetes CRD 资源。它定义了 CAPI 如何连接到外部扩展服务器、扩展支持哪些 Hook、以及调用策略。

**核心作用：**
- 声明扩展服务器的连接信息（URL 或 Service）
- 通过 Discovery 自动发现扩展支持的 Handlers
- 配置调用策略（超时、失败策略、命名空间范围）
- 传递全局 Settings 到所有 Hook 调用

### 二、CRD 结构详解

#### 1. Spec（期望状态）
```yaml
apiVersion: runtime.cluster.x-k8s.io/v1beta2
kind: ExtensionConfig
metadata:
  name: my-extension
  annotations:
    # 自动从 Secret 注入 CA 证书
    runtime.cluster.x-k8s.io/inject-ca-from-secret: default/my-extension-cert
spec:
  # 连接配置（url 或 service 二选一）
  clientConfig:
    # 方式1：外部 URL（必须 https）
    url: https://my-extension.example.com
    
    # 方式2：集群内 Service（推荐）
    service:
      namespace: default
      name: my-extension-svc
      port: 443          # 默认 443
      path: /optional-prefix  # 可选路径前缀
    
    # TLS CA 证书（PEM 格式）
    caBundle: <base64-encoded-ca>

  # 命名空间选择器（决定哪些 Cluster 会触发此扩展）
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
          - default
          - production

  # 全局 Settings（传递给所有 Hook 调用）
  settings:
    key1: value1
    key2: value2
```

#### 2. Status（观察状态）
```yaml
status:
  # Conditions 观察当前状态
  conditions:
    - type: Discovered
      status: "True"
      reason: Discovered
    - type: Paused
      status: "False"

  # 从扩展自动发现的 Handlers 列表
  handlers:
    - name: before-cluster-create
      requestHook:
        apiVersion: hooks.runtime.cluster.x-k8s.io/v1alpha1
        hook: BeforeClusterCreate
      timeoutSeconds: 10
      failurePolicy: Fail
```

### 三、关键字段说明

#### ClientConfig（连接配置）
| 字段 | 说明 | 必填 |
|------|------|------|
| `url` | 外部 HTTPS URL，格式 `scheme://host:port/path` | 与 service 二选一 |
| `service` | 集群内 Service 引用 | 与 url 二选一 |
| `caBundle` | PEM 编码的 CA 证书，用于验证服务器证书 | 可选 |

**约束：**
- `url` 必须是 `https://`
- `url` 不允许用户认证 `user:password@`、片段 `#...`、查询参数 `?...`
- `service` 的 `namespace` 和 `name` 必填，`port` 默认 443

#### NamespaceSelector（命名空间过滤）
- 基于 **Cluster 对象所在命名空间** 进行匹配
- 默认空选择器（匹配所有命名空间）
- 使用标准 Kubernetes LabelSelector

#### Settings（全局配置）
- 键值对，传递给所有 Hook 调用的 `Request.Settings` 字段
- 可在 ClusterClass 的 `.spec.patches[*].external.settings` 中覆盖
- 扩展开发者应文档化支持的 Settings

#### ExtensionHandler（Handler 配置）
| 字段 | 默认值 | 说明 |
|------|--------|------|
| `name` | - | Handler 唯一名称（RFC 1123） |
| `requestHook` | - | 处理的 Hook（APIVersion + Hook 名） |
| `timeoutSeconds` | 10 | 调用超时（秒），最大 30s |
| `failurePolicy` | Fail | `Fail` 或 `Ignore` |

#### FailurePolicy（失败策略）
| 策略 | 行为 |
|------|------|
| `Fail` | 错误传播为 Controller 错误，触发重试 |
| `Ignore` | 记录日志但继续处理 |

**注意：** 以下错误 **永远不会** 被 Ignore：
- 配置错误（类型不兼容等）
- 扩展明确返回 Status Failure

### 四、开发指南

#### 步骤 1：实现 Extension Server
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
    // 创建 Webhook Server
    webhookServer, _ := server.New(server.Options{
        Catalog: catalog,
        Port:    9443,
        CertDir: "/tmp/k8s-webhook-server/serving-certs/",
    })

    // 注册 Handler
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

    // 读取 Settings
    settings := request.GetSettings()

    // 业务逻辑...

    response.Status = runtimehooksv1.ResponseStatusSuccess
}
```

#### 步骤 2：打包部署
**推荐部署架构：**
```
Deployment (多副本) → Service (ClusterIP) → Certificate (cert-manager)
```

**示例 Deployment + Service：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-extension
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-extension
  template:
    metadata:
      labels:
        app: my-extension
    spec:
      containers:
        - name: extension
          image: my-extension:latest
          ports:
            - containerPort: 9443
          volumeMounts:
            - name: cert
              mountPath: /tmp/k8s-webhook-server/serving-certs
              readOnly: true
      volumes:
        - name: cert
          secret:
            secretName: my-extension-cert

---
apiVersion: v1
kind: Service
metadata:
  name: my-extension-svc
  namespace: default
spec:
  selector:
    app: my-extension
  ports:
    - port: 443
      targetPort: 9443
```

#### 步骤 3：创建 ExtensionConfig
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
    matchLabels:
      use-my-extension: "true"
  settings:
    my-setting: my-value
```

#### 步骤 4：在 ClusterClass 中引用
```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: ClusterClass
metadata:
  name: my-cluster-class
spec:
  patches:
    - name: my-patch
      external:
        discoverVariablesExtension: my-extension
        generateExtension: my-extension
        validateExtension: my-extension
        settings:
          # 覆盖 ExtensionConfig 级别的 settings
          my-setting: override-value
```

### 五、重要指南

#### 1. 超时设计
- Handler 响应应在 **毫秒级**
- 最大超时 30s，默认 10s
- 长时间任务应异步执行，不要阻塞同步调用

#### 2. 幂等性
- Handler 可能被 **多次调用**（即使之前已成功）
- 检查当前状态，仅在需要时变更
- 示例：两个扩展注册同一 Hook，一个阻塞重试，另一个会被重复调用

#### 3. 阻塞 Hook
- 返回非零 `retryAfterSeconds` 阻止操作继续
- 多个扩展返回不同值时，取 **最小非零值**
- 一个阻塞时，**所有** 扩展会再次被调用

#### 4. 错误消息
- 不得包含敏感信息
- 必须确定性（避免时间戳、随机值）
- 外部 API 错误需清洗后再返回

#### 5. 避免跨扩展依赖
- 每个扩展应独立完成其任务
- 依赖其他扩展会使系统脆弱

### 六、调试技巧

#### 手动测试 Discovery
```bash
# 通过 kubectl proxy
kubectl create --raw '/api/v1/namespaces/default/services/https:my-extension-svc:443/proxy/hooks.runtime.cluster.x-k8s.io/v1alpha1/discovery' \
  -f <(echo '{"apiVersion":"hooks.runtime.cluster.x-k8s.io/v1alpha1","kind":"DiscoveryRequest"}') | jq
```

#### 检查 ExtensionConfig 状态
```bash
kubectl get extensionconfig my-extension -o yaml
# 查看 status.handlers 确认发现成功
# 查看 status.conditions 确认 Discovered=True
```

### 七、关键注解
| 注解 | 用途 |
|------|------|
| `runtime.cluster.x-k8s.io/inject-ca-from-secret` | 自动从 Secret 注入 CA 到 `caBundle` |
| `runtime.cluster.x-k8s.io/pending-hooks` | 跟踪待完成的 Hook 调用 |
| `runtime.cluster.x-k8s.io/ok-to-delete` | 标记 Cluster 可安全删除（BeforeClusterDelete 通过后） |

### 八、参考示例
- 完整测试扩展： test/extension/ 目录
- Handler 示例： test/extension/handlers/

- **完整测试扩展：** `test/extension/` 目录
- **Handler 示例：** `test/extension/handlers/`
