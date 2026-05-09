# conversion
**Kubernetes 的 CRD conversion 机制用于在同一个自定义资源定义（CRD）支持多个版本时，保证 API Server 能够在不同版本之间透明地转换对象，从而实现平滑升级与兼容。它通过 `served`/`storage` 标志和可选的 conversion webhook 来完成。**
## 核心机制
### 1. 多版本支持
- **`spec.versions`**：CRD 可以定义多个版本（如 v1alpha1、v1beta1、v1）。  
- **`served: true`**：该版本可通过 API Server 提供给客户端。  
- **`storage: true`**：该版本是唯一的存储版本，所有对象在 etcd 中以此版本保存。
- ### 2. 转换策略
- **None**：仅修改 `apiVersion` 字段，不做 schema 转换。适用于字段完全兼容的情况。  
- **Webhook**：调用外部 webhook 服务执行自定义转换逻辑，适用于字段结构变化或语义变化。
### 3. Hub-and-Spoke 模型
- **Hub version**：定义一个中心版本作为转换的“枢纽”。  
- **Spoke versions**：其他版本通过转换 webhook与 Hub 版本互相转换。  
- **好处**：避免版本之间两两转换的复杂性，只需维护 Hub ↔ Spoke 的转换逻辑。
## 工作流程
1. **客户端请求**：用户通过 API 请求某个版本的 CRD 对象。  
2. **API Server**：如果请求版本不是存储版本，API Server 会调用 conversion 逻辑。  
3. **转换执行**：  
   - 若策略为 None → 自动转换 `apiVersion`。  
   - 若策略为 Webhook → 调用 webhook 服务进行字段映射与转换。  
4. **返回结果**：API Server 将转换后的对象返回给客户端。
## 示例场景
- **字段重命名**：`spec.region` → `spec.primaryRegion` + `spec.replicaRegions`。  
- **旧对象存储**：仍以旧版本存储在 etcd。  
- **新客户端请求**：通过 conversion webhook 转换为新版本结构返回。  
- **兼容性保证**：旧客户端仍可使用旧版本 API，不会因 schema 更新而报错。
## 关键价值
- **零停机升级**：支持在生产环境中平滑引入新版本。  
- **兼容性**：允许不同客户端同时使用不同版本。  
- **灵活性**：通过 webhook 实现复杂的字段转换逻辑。
## 总结
Kubernetes CRD conversion 机制通过 **served/storage 标志**与**conversion webhook**，实现了 **多版本共存、透明转换、零停机升级**。在设计 CRD 时，推荐采用 **Hub-and-Spoke 模型**，并在 schema 变更时实现 webhook 转换逻辑，以保证兼容性和稳定性。 
- [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) 
- [Medium](https://medium.com/@rameshavutu/crd-versioning-conversion-webhooks-evolving-your-api-without-breaking-production-42f7b95724ee)


# 对CRD进行了不兼容的变更，同时版本号没有改变，如何进行平滑的升级
**结论：最稳妥的做法是把 CRD 变更做成多版本兼容（新增新版本并使用 Conversion Webhook 做双向/单向转换），同时让控制器兼容旧版与新版对象，逐步迁移并最终下线旧版本。**  
### 核心步骤（按优先级执行）
1. **新增 CRD 版本**：在 `spec.versions` 中添加新版本（`served:true`），暂时保留旧版本为 `served:true`，并把 **storage** 指向一个版本（通常先不改 storage）。**（Kubernetes 官方推荐）**。  [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)  
2. **实现 Conversion Webhook**：若字段/结构不兼容，部署 conversion webhook 做 **on‑the‑fly** 转换（建议实现双向或 hub 模式），保证 API server 能在不同版本间透明转换。  [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)  [oneuptime.com](https://oneuptime.com/blog/post/2026-02-09-crd-version-upgrades-conversion/view)  
3. **控制器兼容性**：更新控制器逻辑以同时处理旧版与新版 CR（或以 hub 版本为中心做转换），避免仅靠 API server 转换导致运行时语义差异。  
4. **渐进迁移**：在非高峰环境逐步把资源写入/读取切换到新版本（或把 CRD 的 `storage` 切换到新版本并触发后台转换），同时运行数据迁移脚本以把 etcd 中对象转换为 storage 版本。  [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)  
5. **下线旧版本**：确认所有控制器与外部系统不再使用旧版本后，移除旧版本的 `served:true` 并清理兼容代码。
### 可选/补充策略对比
| 策略 | 优点 | 缺点 | 适用场景 |
|---|---:|---|---|
| **Conversion Webhook（推荐）** | 无需停机，API 层透明转换 | 需实现并维护 webhook 逻辑 | 字段重命名/结构变更 |
| **Dual‑write（控制器同时写两种版本）** | 快速过渡，回滚简单 | 复杂度高，易出不一致 | 短期兼容过渡 |
| **离线迁移脚本（一次性转换 etcd）** | 简单直接 | 需停机或强一致保证 | 小规模集群或维护窗口可控 |
| **Adapter 层（兼容层）** | 最小改动客户端 | 增加运行时复杂度 | 外部系统难改时使用 |
### 风险与缓解（必须强调）
- **数据丢失/语义错配**：先做快照与回滚计划；在沙箱多轮演练。**强烈使用 etcd 备份**。  
- **Webhook 可用性**：conversion webhook 必须高可用，建议部署多副本并加健康探针。  
- **控制器未兼容**：在变更前把控制器升级为兼容双版本模式，避免运行时异常。  
### 验证清单（快速执行）
- **部署 conversion webhook 并通过 `kubectl get <cr> -o yaml --api-version=<ver>` 验证转换**。  [Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)  
- **在测试集群做 storage 切换演练并验证 etcd 对象**。  
- **准备回滚脚本与 etcd 快照恢复步骤**。

# Conversion Webhook的详细方案
## 概述
下面是一份**整合版 Conversion Webhook 方案**，覆盖 **served/storage 设计、conversion webhook 开发与部署、控制器兼容两版的代码设计、切换 `storage:true` 到 v2 的执行逻辑**，并附带一个 **可运行的最小完整代码样例**（教学级，生产需补 TLS、RBAC、证书管理、日志与健壮性增强）。目标：在对 CRD 做不兼容变更且版本号未改变的情况下，实现平滑、可回滚的升级路径。
## 1 served 与 storage 设计（版本策略）
**目标**：新增 v2，保证 API 可同时服务 v1 与 v2，最终把 etcd 存储（storage）切换到 v2（或 hub）。

**设计要点**
- **初始阶段**
  - `v1.served: true, v1.storage: true`
  - `v2.served: true, v2.storage: false`
  - CRD `conversion.strategy: Webhook`，指向 conversion webhook 服务
- **中间阶段（控制器兼容并验证）**
  - 保持两版 `served:true`，conversion webhook 提供 `v1 ⇄ v2`（或 hub）转换
  - 控制器同时兼容 v1 与 v2（或统一转换到 hub）
- **切换阶段**
  - 把 `storage:true` 从 v1 切换到 v2（或 hub），API server 在后台把 etcd 对象转换为 storage 版本
- **下线阶段**
  - 确认所有对象已转换且系统稳定后，把 `v1.served:false` 并移除兼容代码

**CRD 版本片段示例**
```yaml
spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: conversion-ns
          name: foo-conversion-svc
          path: /convert
        caBundle: <base64-CA>
  versions:
    - name: v1
      served: true
      storage: true
    - name: v2
      served: true
      storage: false
```

**hub 说明**
- **hub** 是可选的内部中介模型。推荐 hub 模式（v1↔hub↔v2）以降低转换复杂度（O(n) 而非 O(n²)）。hub 可以是 `v2`（直接以 v2 为 hub）或单独定义的内部版本（不 `served`）。
## 2 conversion webhook 的开发与部署设计
**功能要求**
- 支持 `ConversionReview` v1，批量转换 `objects`。
- 支持 `DesiredAPIVersion` 到 `ConvertedObjects` 的转换。
- 幂等、可重入、错误返回明确（在 `ConversionResponse.Result` 中返回错误信息）。
- 保留 `metadata`（尤其 `metadata.uid`、`metadata.resourceVersion`、`metadata.annotations/labels`）并正确处理缺失字段与默认值。

**实现策略**
- **Hub 模式**：实现 `v1 ↔ hub` 与 `v2 ↔ hub` 的转换函数；控制器以 hub 为内部模型。
- **转换函数**：每个版本到 hub 的转换与 hub 到每个版本的转换都应有单元测试，覆盖字段缺失、默认值、向后兼容策略。
- **批量处理**：一次请求可能包含多个对象，逐个转换并返回数组。
- **错误处理**：单个对象转换失败应返回 Failure 并包含错误信息；可选策略为部分成功/部分失败需在设计中明确。

**部署要求**
- **高可用**：Deployment 多副本（建议 ≥2），PodDisruptionBudget，readiness/liveness 探针。
- **TLS**：Webhook 必须使用 HTTPS；使用 cert-manager 或手动签发证书并把 CA 填入 CRD 的 `caBundle`。
- **Service**：ClusterIP Service 暴露 `/convert` 路径。
- **RBAC**：若 webhook 需要访问 Kubernetes API（例如读取其它资源），配置最小权限的 Role/RoleBinding。
- **监控与日志**：导出 metrics（转换计数、失败率、延迟）并记录转换日志与请求 UID。

**健康与可用性**
- 提供 `/healthz` 与 `/readyz`，API server 在调用 webhook 前会检查可用性。
- 在切换 `storage` 期间 webhook 必须稳定可用，否则切换会失败或阻塞。
## 3 控制器兼容两版的代码设计
**总体策略**
- 控制器接收任意 `served` 版本对象后，**统一转换到 hub 内部结构**（typed Go struct），然后执行业务逻辑。这样控制器只需实现一套业务逻辑，版本转换集中在转换函数中。

**关键点**
- 使用 `controller-runtime` 的 `Unstructured` 或 typed client：
  - 若使用 `Unstructured`：在 Reconcile 中根据 `u.GetAPIVersion()` 反序列化到对应版本的 Go struct，再转换到 hub。
  - 若使用 typed client：可以直接 watch 指定版本的 Go 类型（需要在 scheme 中注册多个版本），但仍建议统一到 hub。
- 实现转换函数：`V1ToHub`, `V2ToHub`, `HubToV1`, `HubToV2`。
- 在 Reconcile 中只处理 hub 类型：`reconcileHub(ctx, hub)`。
- 兼容期内保留 v1 转换代码；在下线 v1 后移除。

**Reconcile 伪代码**
```go
u := &unstructured.Unstructured{}
u.SetGroupVersionKind(schema.GroupVersionKind{Group:"example.com", Version:"v1", Kind:"Foo"})
r.Get(ctx, req.NamespacedName, u)
apiVersion := u.GetAPIVersion()
var hub FooHub
if apiVersion == "example.com/v1" {
  var v1 FooV1
  runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v1)
  hub = V1ToHub(&v1)
} else if apiVersion == "example.com/v2" {
  var v2 FooV2
  runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v2)
  hub = V2ToHub(&v2)
}
r.reconcileHub(ctx, &hub)
```
## 4 切换 storage:true 到 v2 的执行逻辑（操作步骤与注意）
**前置准备**
- **备份 etcd 快照** 并验证恢复流程。
- **Webhook 高可用**：至少 2 副本，健康探针通过。
- **控制器已部署** 并能兼容 v1 与 v2（或 hub）。
- **测试环境演练**：在 dev/预生产先做完整演练。

**切换步骤**
1. **导出当前 CRD**：
   ```bash
   kubectl get crd foos.example.com -o yaml > crd-current.yaml
   ```
2. **确保 v2.served:true**（若尚未）并部署 webhook 与兼容控制器。
3. **编辑 CRD**：把 `storage:true` 从 v1 改为 v2（或改为 hub）：
   ```yaml
   # 修改 versions 中
   - name: v1
     served: true
     storage: false
   - name: v2
     served: true
     storage: true
   ```
4. **应用 CRD 变更**：
   ```bash
   kubectl apply -f crd-current.yaml
   ```
5. **监控后台转换**：
   - 观察 API server 日志与 conversion webhook 日志。
   - 抽样检查对象 `apiVersion`：
     ```bash
     kubectl get foos -A -o json | jq '.items[] | {name:.metadata.name, apiVersion:.apiVersion}'
     ```
   - 监控 conversion webhook metrics（错误率、延迟）。
6. **验证控制器行为**：确认 reconcile 正常、无异常重试或数据不一致。
7. **下线旧版本**：确认所有对象已转换且稳定后，把 `v1.served` 设为 `false` 并移除兼容代码：
   ```bash
   # 编辑 crd-current.yaml 把 v1.served: false
   kubectl apply -f crd-current.yaml
   ```
8. **回滚策略**：
   - 若转换失败或 webhook 不可用，立即恢复 CRD 到原始 `storage:true` 配置并 apply，或使用 etcd 快照恢复。
   - 回滚控制器镜像到兼容版本。

**注意**
- 切换 storage 期间 API server 会调用 conversion webhook；若 webhook 不可用，切换会失败或阻塞。
- 后台转换可能耗时，取决于对象数量与 webhook 性能。
## 5 完整代码样例（最小可运行教学版）
包含：CRD（简化）、conversion webhook（Go，HTTP 教学版）、控制器兼容片段、切换脚本示例。**生产必须补 TLS、RBAC、证书管理、日志、错误处理与构建脚本**。
### 5.1 CRD YAML（`crd-foos.yaml`）
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: foos.example.com
spec:
  group: example.com
  names:
    kind: Foo
    plural: foos
  scope: Namespaced
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: conversion-ns
          name: foo-conversion-svc
          path: /convert
        caBundle: <base64-CA>
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                image:
                  type: string
    - name: v2
      served: true
      storage: false
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                count:
                  type: integer
                container:
                  type: object
                  properties:
                    image:
                      type: string
```
### 5.2 Conversion Webhook 教学版（`cmd/webhook/main.go`）
> **说明**：示例为 HTTP（教学），生产必须用 HTTPS/TLS（`ListenAndServeTLS`），并把 CA 填入 CRD 的 `caBundle`。需在 `go.mod` 中添加 k8s 依赖。
```go
package main

import (
  "encoding/json"
  "flag"
  "io/ioutil"
  "log"
  "net/http"

  conversionv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
  metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/apimachinery/pkg/runtime"
)

// Simplified typed structs
type FooV1 struct {
  APIVersion string            `json:"apiVersion,omitempty"`
  Kind       string            `json:"kind,omitempty"`
  Metadata   metav1.ObjectMeta `json:"metadata,omitempty"`
  Spec       struct {
    Replicas int    `json:"replicas,omitempty"`
    Image    string `json:"image,omitempty"`
  } `json:"spec,omitempty"`
}

type FooV2 struct {
  APIVersion string            `json:"apiVersion,omitempty"`
  Kind       string            `json:"kind,omitempty"`
  Metadata   metav1.ObjectMeta `json:"metadata,omitempty"`
  Spec       struct {
    Count     int `json:"count,omitempty"`
    Container struct {
      Image string `json:"image,omitempty"`
    } `json:"container,omitempty"`
  } `json:"spec,omitempty"`
}

// hub model
type FooHub struct {
  Metadata metav1.ObjectMeta `json:"metadata,omitempty"`
  Spec     struct {
    Replicas int    `json:"replicas,omitempty"`
    Image    string `json:"image,omitempty"`
  } `json:"spec,omitempty"`
}

func v1ToHub(obj map[string]interface{}) (*FooHub, error) {
  b, _ := json.Marshal(obj)
  var v1 FooV1
  if err := json.Unmarshal(b, &v1); err != nil { return nil, err }
  var hub FooHub
  hub.Metadata = v1.Metadata
  hub.Spec.Replicas = v1.Spec.Replicas
  hub.Spec.Image = v1.Spec.Image
  return &hub, nil
}

func hubToV2(hub *FooHub) (map[string]interface{}, error) {
  var v2 FooV2
  v2.APIVersion = "example.com/v2"
  v2.Kind = "Foo"
  v2.Metadata = hub.Metadata
  v2.Spec.Count = hub.Spec.Replicas
  v2.Spec.Container.Image = hub.Spec.Image
  b, _ := json.Marshal(v2)
  var out map[string]interface{}
  json.Unmarshal(b, &out)
  return out, nil
}

func v2ToHub(obj map[string]interface{}) (*FooHub, error) {
  b, _ := json.Marshal(obj)
  var v2 FooV2
  if err := json.Unmarshal(b, &v2); err != nil { return nil, err }
  var hub FooHub
  hub.Metadata = v2.Metadata
  hub.Spec.Replicas = v2.Spec.Count
  hub.Spec.Image = v2.Spec.Container.Image
  return &hub, nil
}

func hubToV1(hub *FooHub) (map[string]interface{}, error) {
  var v1 FooV1
  v1.APIVersion = "example.com/v1"
  v1.Kind = "Foo"
  v1.Metadata = hub.Metadata
  v1.Spec.Replicas = hub.Spec.Replicas
  v1.Spec.Image = hub.Spec.Image
  b, _ := json.Marshal(v1)
  var out map[string]interface{}
  json.Unmarshal(b, &out)
  return out, nil
}

func convertHandler(w http.ResponseWriter, r *http.Request) {
  body, err := ioutil.ReadAll(r.Body)
  if err != nil {
    http.Error(w, "bad request", http.StatusBadRequest)
    return
  }
  var review conversionv1.ConversionReview
  if err := json.Unmarshal(body, &review); err != nil {
    http.Error(w, "invalid conversion review", http.StatusBadRequest)
    return
  }

  req := review.Request
  resp := conversionv1.ConversionResponse{
    UID: req.UID,
    Result: metav1.Status{
      Status: "Success",
    },
  }

  converted := make([]runtime.RawExtension, 0, len(req.Objects))
  for _, raw := range req.Objects {
    var obj map[string]interface{}
    if err := json.Unmarshal(raw.Raw, &obj); err != nil {
      resp.Result = metav1.Status{Status: "Failure", Message: err.Error()}
      review.Response = &resp
      b, _ := json.Marshal(review)
      w.Write(b)
      return
    }

    desired := req.DesiredAPIVersion // e.g. "example.com/v2"
    var out map[string]interface{}
    var hub *FooHub
    var convErr error

    srcVersion, _ := obj["apiVersion"].(string)

    if srcVersion == desired {
      out = obj
    } else {
      if srcVersion == "example.com/v1" {
        hub, convErr = v1ToHub(obj)
      } else if srcVersion == "example.com/v2" {
        hub, convErr = v2ToHub(obj)
      } else {
        hub = &FooHub{} // fallback empty hub
      }
      if convErr != nil {
        resp.Result = metav1.Status{Status: "Failure", Message: convErr.Error()}
        review.Response = &resp
        b, _ := json.Marshal(review)
        w.Write(b)
        return
      }

      if desired == "example.com/v2" {
        out, convErr = hubToV2(hub)
      } else if desired == "example.com/v1" {
        out, convErr = hubToV1(hub)
      } else {
        out = obj
      }
      if convErr != nil {
        resp.Result = metav1.Status{Status: "Failure", Message: convErr.Error()}
        review.Response = &resp
        b, _ := json.Marshal(review)
        w.Write(b)
        return
      }
    }

    bb, _ := json.Marshal(out)
    converted = append(converted, runtime.RawExtension{Raw: bb})
  }

  resp.ConvertedObjects = converted
  review.Response = &resp
  b, _ := json.Marshal(review)
  w.Header().Set("Content-Type", "application/json")
  w.Write(b)
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
  w.WriteHeader(http.StatusOK)
  w.Write([]byte("ok"))
}

func main() {
  var addr string
  flag.StringVar(&addr, "addr", ":8080", "listen address")
  flag.Parse()
  http.HandleFunc("/convert", convertHandler)
  http.HandleFunc("/healthz", healthHandler)
  log.Printf("starting conversion webhook on %s", addr)
  if err := http.ListenAndServe(addr, nil); err != nil {
    log.Fatalf("server failed: %v", err)
  }
}
```
### 5.3 控制器兼容两版核心片段（`controllers/foo_controller.go`）
```go
// 省略 imports
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  var u unstructured.Unstructured
  u.SetGroupVersionKind(schema.GroupVersionKind{Group:"example.com", Version:"v1", Kind:"Foo"})
  if err := r.Get(ctx, req.NamespacedName, &u); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err)
  }

  apiVersion := u.GetAPIVersion() // example.com/v1 or example.com/v2
  var hub FooHub
  if apiVersion == "example.com/v1" {
    var v1 FooV1
    if err := runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v1); err != nil {
      return ctrl.Result{}, err
    }
    hub = FooHubFromV1(&v1)
  } else if apiVersion == "example.com/v2" {
    var v2 FooV2
    if err := runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v2); err != nil {
      return ctrl.Result{}, err
    }
    hub = FooHubFromV2(&v2)
  } else {
    return ctrl.Result{}, fmt.Errorf("unsupported version %s", apiVersion)
  }

  if err := r.reconcileHub(ctx, &hub); err != nil {
    return ctrl.Result{}, err
  }
  return ctrl.Result{}, nil
}
```
### 5.4 切换 storage 的脚本示例（`scripts/switch-storage.sh`）
```bash
#!/usr/bin/env bash
set -euo pipefail

CRD=foos.example.com
BACKUP=crd-backup.yaml

kubectl get crd ${CRD} -o yaml > ${BACKUP}
echo "Backed up CRD to ${BACKUP}"

# Edit: change storage from v1 to v2 (this step can be automated with yq)
# Example using yq (ensure yq v4 installed)
yq eval '.spec.versions |= map(if .name=="v1" then .storage=false elif .name=="v2" then .storage=true else . end)' -i ${BACKUP}

kubectl apply -f ${BACKUP}
echo "Applied CRD with storage switched to v2. Monitor conversion webhook and API server logs."
```
## 6 验证、监控与回滚建议
**验证**
- 在切换前后用 `kubectl get foos -A -o json | jq '.items[] | {name:.metadata.name, apiVersion:.apiVersion}'` 抽样检查。
- 在切换期间监控 conversion webhook 错误率与延迟、API server 错误日志、控制器重试次数。

**回滚**
- 若出现严重错误，先把 CRD 恢复为备份文件（`kubectl apply -f crd-backup.yaml`），并回滚控制器镜像或使用 etcd 快照恢复。

**测试矩阵**
- 单对象转换测试（v1→v2、v2→v1）
- 批量对象压力测试（数千对象）
- webhook 宕机/延迟场景测试（模拟网络分区）
- 控制器兼容性测试（同时存在 v1/v2 对象）
## 7 最佳实践与注意事项
- **证书管理**：使用 cert-manager 自动签发 webhook server 证书并把 CA 填入 CRD。
- **使用 immutable image digest**：避免镜像 tag 被重写导致行为不一致。
- **日志与审计**：在转换点记录事件（包含对象 UID、请求 UID、转换结果）。
- **分阶段发布**：先在 dev → staging → prod，逐步扩大范围。
- **预留回滚窗口**：切换 storage 前后至少保留监控与回滚窗口 24–72 小时（视对象数量与业务影响而定）。

如果你需要，我可以把上述教学示例：
- **扩展为生产级模板**（TLS、cert-manager、RBAC、Deployment、Service、PDB、HPA、metrics）；或
- **生成一套自动化切换脚本**（pre-check、切换、post-check、超时与自动回滚）；或
- **把转换函数的单元测试用例**写出来，便于直接运行 CI 验证。

# 如果在CRD的修改中进行了不兼容变更，而没有增加多版本（如删除了字段，修改了字段名），如果进行处理？
## 要点总结
**如果已经对 CRD 做了不兼容变更但没有新增版本号，最稳妥的处理路径是：**  
1. **立即做缓解**（避免新写入破坏现有数据）；  
2. **尽快补上多版本支持或兼容层**（新增 `v2` 或引入 conversion/adaptor）；  
3. **逐步迁移并验证**，最后下线旧语义。  

下面给出可执行的分步方案、命令片段、代码示例与回滚策略，覆盖短期应急到中长期稳健迁移的全流程。
## 1 立即缓解（0–24 小时）
**目标：阻止进一步破坏并保留恢复点。**

- **冻结写入到该 CRD**（如果业务允许）：
  ```bash
  # 给所有用户/系统打上只读注记（示例：用 NetworkPolicy/Admission 或临时 RBAC）
  kubectl create rolebinding freeze-foos --clusterrole=view --user=system:serviceaccount:default:default -n default
  ```
  或者临时用 Admission webhook 拦截 `CREATE`/`UPDATE` 请求返回 429/403（短期手段）。

- **立即备份 etcd**（必须）：
  - 如果是 managed 集群，使用云厂商快照工具；自管集群用 `etcdctl snapshot save`。
- **导出当前 CRD 与所有对象**：
  ```bash
  kubectl get crd foos.example.com -o yaml > crd-backup.yaml
  kubectl get foos --all-namespaces -o yaml > foos-all-backup.yaml
  ```

- **通知团队并暂停自动化升级/CI 对该 CRD 的变更**。
## 2 可选短期修复（当天到数日）——兼容适配层（不改 etcd）
当无法马上做多版本或 conversion webhook 时，先做一个**兼容适配控制器**（Adapter Controller）或 Admission 层，保证旧客户端/新客户端都能工作。

**方案 A 适配控制器（推荐短期）**
- 新建一个控制器，**监听当前 CRD（不管字段如何变）**，把旧格式对象转换成控制器/下游期望的内部模型或新格式资源（可以是新的 CRD 或同 CRD 的不同字段位置），并维护一个同步副本或状态。
- 优点：无需改 CRD storage，快速上线；缺点：增加运行时复杂度，需保证幂等。

**伪代码（Go）核心逻辑**
```go
// watch Unstructured for foos
u := &unstructured.Unstructured{}
u.SetGroupVersionKind(schema.GroupVersionKind{Group:"example.com", Version:"v1", Kind:"Foo"})
r.Get(ctx, req.NamespacedName, u)
// 解析旧字段（可能已删除或改名）
oldVal, _, _ := unstructured.NestedString(u.Object, "spec", "oldField")
// 构造新模型或更新副本
newObj := map[string]interface{}{ "apiVersion":"example.com/v2", "kind":"Foo", "metadata":u.Object["metadata"], "spec": map[string]interface{}{"newField": oldVal} }
// 写入到另一个 namespace 或 CRD（或更新 annotation/status）
```

**方案 B Admission Webhook（临时拦截）**
- 在 API Server 前端拦截 `CREATE/UPDATE`，把新请求转换成旧格式写入（或拒绝写入并返回友好错误），适合短期阻断错误写入。

## 3 中期稳健方案（几天到数周）——补上多版本与 Conversion Webhook（推荐）
**目标：把 CRD 恢复到多版本兼容流程，长期可维护。**
### 3.1 设计决策
- **新增版本 `v2`**（或定义 hub），把不兼容变更放到 `v2`。  
- **conversion.strategy: Webhook**，实现 `v1 ⇄ v2`（推荐 hub 模式：`v1 ⇄ hub ⇄ v2`）。  
- **初期**：`v1.served:true, v1.storage:true; v2.served:true, v2.storage:false`。  
- **控制器**：升级为兼容两版（见下一节）。
### 3.2 操作步骤（示例命令）
1. **修改 CRD 增加 v2（served:true）但不切换 storage**：
   ```bash
   # 编辑 CRD，添加 v2.served:true storage:false
   kubectl apply -f crd-with-v2.yaml
   ```
2. **部署 conversion webhook 服务并把 CA 填入 CRD.clientConfig.caBundle**（见下方代码样例）。  
3. **升级控制器到兼容版本**（能处理 v1 与 v2 或统一 hub）。  
4. **测试转换**：
   ```bash
   # 验证 kubectl get --api-version=example.com/v2 能正确返回对象
   kubectl get foo myfoo -o yaml --api-version=example.com/v2
   ```
5. **切换 storage 到 v2**（见第 5 节详细步骤）。
## 4 控制器兼容两版的代码设计（实现细节）
**原则**：控制器内部使用 **单一 hub 模型**，所有外部版本先转换为 hub，再执行业务逻辑；写回时按 storage 版本或目标版本转换回去。

**关键函数**
- `FromV1ToHub(v1)`, `FromV2ToHub(v2)`  
- `FromHubToV1(hub)`, `FromHubToV2(hub)`

**Reconcile 流程**
1. 读取对象为 `Unstructured`（或 typed，但 Unstructured 更通用）。  
2. 根据 `u.GetAPIVersion()` 调用对应转换到 hub。  
3. 在 hub 上执行业务逻辑（创建/更新子资源、更新 status）。  
4. 若需要更新 CR 本体，先把 hub 转回 storage 版本再写入（或让 API server 在写入时做转换）。

**示例片段（核心）**
```go
u := &unstructured.Unstructured{}
u.SetGroupVersionKind(schema.GroupVersionKind{Group:"example.com", Version:"v1", Kind:"Foo"})
if err := r.Get(ctx, req.NamespacedName, u); err != nil { ... }
switch u.GetAPIVersion() {
case "example.com/v1":
  var v1 FooV1; runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v1)
  hub := V1ToHub(&v1)
case "example.com/v2":
  var v2 FooV2; runtime.DefaultUnstructuredConverter.FromUnstructured(u.Object, &v2)
  hub := V2ToHub(&v2)
}
r.reconcileHub(ctx, hub)
```
## 5 切换 storage 到 v2 的执行逻辑（详细步骤与命令）
**前提**：Webhook 高可用、控制器兼容、已备份 etcd。

1. **备份 CRD 与 etcd**：
   ```bash
   kubectl get crd foos.example.com -o yaml > crd-backup.yaml
   kubectl get foos --all-namespaces -o yaml > foos-backup.yaml
   # etcd snapshot as per cluster setup
   ```
2. **确认 webhook 健康**：
   ```bash
   kubectl get deploy -n conversion-ns foo-conversion -o wide
   kubectl exec -n conversion-ns deploy/foo-conversion -- curl -sfS http://localhost:8080/healthz
   ```
3. **确保 v2.served:true 已存在**。  
4. **修改 CRD 把 storage 切换到 v2**（编辑 `spec.versions`）：
   ```bash
   kubectl get crd foos.example.com -o yaml > crd-edit.yaml
   # edit crd-edit.yaml: set v1.storage=false, v2.storage=true
   kubectl apply -f crd-edit.yaml
   ```
5. **监控后台转换**：
   - 抽样检查对象 `apiVersion`：
     ```bash
     kubectl get foos -A -o json | jq '.items[] | {name:.metadata.name, apiVersion:.apiVersion}'
     ```
   - 监控 API server 日志与 conversion webhook 日志。  
6. **验证控制器与业务**：确认 reconcile 正常、无异常重试。  
7. **下线 v1.served**（确认无客户端再移除）：
   ```bash
   # edit crd-edit.yaml: set v1.served=false
   kubectl apply -f crd-edit.yaml
   ```

**回滚**
- 若出现严重问题，立即 `kubectl apply -f crd-backup.yaml` 恢复 CRD 配置并根据需要用 etcd 快照恢复数据。
## 6 代码样例与脚本（整合）
下面给出**最小化但可运行的要点代码与脚本**（教学级），供直接参考或改造。
### A CRD 片段（增 v2）
```yaml
# crd-with-v2.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: foos.example.com
spec:
  group: example.com
  names:
    kind: Foo
    plural: foos
  scope: Namespaced
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: conversion-ns
          name: foo-conversion-svc
          path: /convert
        caBundle: <base64-CA>
  versions:
    - name: v1
      served: true
      storage: true
      schema: ...
    - name: v2
      served: true
      storage: false
      schema: ...
```
### B conversion webhook 教学版（核心已在前文给出）  
- 请参考上文 `cmd/webhook/main.go` 示例，生产需改为 `ListenAndServeTLS`、挂载证书 Secret、增加 metrics、并实现批量转换的健壮错误处理。
### C Adapter Controller 伪实现（短期）
```go
// watch Unstructured, convert old fields to new location and write annotation/status
func convertAndAnnotate(u *unstructured.Unstructured) error {
  old, _, _ := unstructured.NestedString(u.Object, "spec", "oldField")
  if old != "" {
    // write annotation to mark converted
    if err := unstructured.SetNestedField(u.Object, old, "metadata", "annotations", "compat.example.com/oldField"); err != nil { return err }
    // optionally update status or create a shadow resource
    _, err := kubeClient.Resource(gvr).Namespace(u.GetNamespace()).Update(context.TODO(), u, metav1.UpdateOptions{})
    return err
  }
  return nil
}
```
### D 切换 storage 自动化脚本（示例）
```bash
#!/usr/bin/env bash
set -euo pipefail
CRD=foos.example.com
BACKUP=crd-backup.yaml
kubectl get crd ${CRD} -o yaml > ${BACKUP}
echo "backup saved ${BACKUP}"
# use yq to flip storage
yq eval '.spec.versions |= map(if .name=="v1" then .storage=false elif .name=="v2" then .storage=true else . end)' -i ${BACKUP}
kubectl apply -f ${BACKUP}
echo "applied CRD with storage switched to v2"
```
## 7 风险清单 与 缓解措施
- **Webhook 不可用导致切换失败**：部署多副本、PDB、readiness probe；切换前做压力测试。  
- **数据语义丢失**（字段被删除）**或误映射**：在转换函数中保留原始字段到 annotation 或 status，便于回滚与审计。  
- **控制器不兼容导致 reconcile 循环或数据损坏**：先在测试环境验证，灰度发布控制器。  
- **回滚复杂度高**：始终先做 etcd 快照并验证恢复流程；保留旧版本 `served:true` 直到确认安全下线。  
## 结论与建议
- **短期**：先冻结写入并备份；上线适配控制器或 Admission 拦截，防止进一步破坏。  
- **中期**：补上 `v2` 与 conversion webhook，升级控制器为兼容 hub 模型，逐步切换 `storage`。  
- **长期**：把转换逻辑、单元/集成测试、监控与回滚流程纳入 CI/CD，避免未来再出现未版本化的不兼容变更。  
