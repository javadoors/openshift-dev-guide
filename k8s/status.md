# Conditions 模式

## Kubernetes Conditions 模式的实现

### 一、什么是 Conditions 模式

Conditions 是 Kubernetes API 规范中**记录资源细粒度状态的标准模式**，定义在 [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) 中。

**核心思想**：一个资源可能有多个维度的子状态，用单个 `phase` 字段无法表达，因此用 `conditions` 数组记录各维度的状态。

```yaml
# K8s 标准模式示例（Pod status）
status:
  phase: Running                    # 粗粒度单一状态
  conditions:                       # 细粒度多维度状态
  - type: PodScheduled
    status: "True"
    reason: "Scheduled"
    message: "Successfully assigned to node-1"
  - type: Initialized
    status: "True"
    reason: "PodCompleted"
  - type: Ready
    status: "False"
    reason: "ContainersNotReady"
    message: "containers with unready status"
```

### 二、四元组标准结构

K8s 规范定义 Condition 的标准字段：

| 字段 | 类型 | 含义 | 是否必需 |
|------|------|------|---------|
| **`type`** | `string` | Condition 类型，标识维度名称（如 `Ready`、`PodScheduled`） | 是 |
| **`status`** | `ConditionStatus` | 该条件是否满足，取值 `True`/`False`/`Unknown` | 是 |
| **`reason`** | `string` | 状态原因的简短标识（CamelCase，程序可读） | 推荐 |
| **`message`** | `string` | 状态详情的人类可读描述 | 推荐 |

**附加标准字段**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `lastTransitionTime` | `*metav1.Time` | 最后一次状态转换的时间 |
| `lastUpdateTime` | `*metav1.Time` | 最后一次更新时间（已被 lastTransitionTime 取代） |
| `observedGeneration` | `int64` | 观察到的资源 generation，用于判断状态是否过期 |

### 三、`status` 三态的语义

`ConditionStatus` 的三个取值有明确语义：

| Status | 语义 | 典型场景 |
|--------|------|---------|
| `True` | 条件已满足 | `Ready=True` 表示 Pod 就绪 |
| `False` | 条件未满足（明确失败） | `Ready=False` 表示 Pod 未就绪 |
| `Unknown` | 无法判断 | 通常在 controller 异常时出现 |

**关键理解**：`False` **不等于错误**，仅代表"该条件未满足"。例如 `PodScheduled=False` 是正常中间状态（正在调度中），只有长时间保持 `False` 才是问题。

# k8s中status/condition设计最佳实践
**在 Kubernetes 中，`status/conditions` 的最佳实践是：保持条件语义清晰、可扩展、可监控，并避免过度依赖单一 `phase` 字段。条件应当独立描述对象的不同状态维度（如 Ready、Scheduled、Initialized），并且遵循一致的命名、布尔化和时间戳更新规则。**  
## 📑 设计最佳实践要点
### 1. 条件 vs. Phase
- **Phase**：提供对象生命周期的高层次摘要（如 Pod 的 Pending、Running、Succeeded）。  
- **Conditions**：提供更细粒度的状态标记，例如 Pod 是否已调度、容器是否就绪。  
- **最佳实践**：不要仅依赖 `phase`，应通过多个条件组合来反映真实状态  [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-condition/)。  
### 2. 条件字段设计
- **Type**：条件类型应简洁、语义明确（如 `Ready`、`Scheduled`、`Initialized`）。  
- **Status**：仅使用 `True`、`False`、`Unknown` 三值，避免复杂枚举。  
- **Reason**：提供机器可读的简短字符串，解释条件变化原因。  
- **Message**：提供人类可读的详细说明，便于排查。  
- **LastTransitionTime**：记录条件状态最后一次变化的时间，便于监控与调试  [superorbital.io](https://superorbital.io/blog/status-and-conditions/)。  
### 3. 更新策略
- **幂等性**：条件更新应幂等，避免重复写入相同状态。  
- **最小化更新**：仅在状态变化时更新 `LastTransitionTime`，避免无意义的频繁写入。  
- **一致性**：不同控制器更新条件时，应遵循统一的命名和语义。  
### 4. 可观测性与监控
- **kubectl wait**：条件设计应支持 `kubectl wait --for=condition=Ready=true` 等命令。  
- **监控集成**：条件应能与 Prometheus、Alertmanager 等监控工具结合，触发告警。  
- **最佳实践**：为关键条件（如 `Ready`）建立监控指标，避免仅依赖日志  [Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/best-practices-monitoring-proactive)。  
## 📊 条件设计示例（Pod）
```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-04-20T09:00:00Z"
    reason: ContainersReady
    message: All containers are ready
  - type: Scheduled
    status: "True"
    lastTransitionTime: "2026-04-20T08:59:00Z"
    reason: SchedulerSuccess
    message: Pod successfully scheduled on node worker-1
```
## ✅ 总结
- 使用 **conditions** 来表达对象的多维状态，而不是依赖单一 `phase`。  
- 保持条件语义清晰、布尔化、带时间戳，便于监控与自动化工具使用。  
- 遵循 **幂等更新** 与 **一致性命名**，避免条件混乱。  

# 条件设计清单
涵盖常见的五类条件（Ready、Scheduled、Initialized、Progressing、Degraded），并给出推荐的字段组合，方便你在自定义 CRD 中直接套用。  
## 📑 条件设计清单
### 1. **Ready**
- **Type**: `Ready`  
- **Status**: `True` / `False` / `Unknown`  
- **Reason**: `ComponentsReady` / `ComponentsNotReady`  
- **Message**: “所有组件已就绪” / “部分组件未就绪”  
- **LastTransitionTime**: 更新时间戳  
### 2. **Scheduled**
- **Type**: `Scheduled`  
- **Status**: `True` / `False`  
- **Reason**: `SchedulerSuccess` / `SchedulerFailure`  
- **Message**: “已成功调度到节点 X” / “调度失败，资源不足”  
- **LastTransitionTime**: 更新时间戳  
### 3. **Initialized**
- **Type**: `Initialized`  
- **Status**: `True` / `False`  
- **Reason**: `InitCompleted` / `InitPending`  
- **Message**: “初始化完成” / “等待初始化任务”  
- **LastTransitionTime**: 更新时间戳  
### 4. **Progressing**
- **Type**: `Progressing`  
- **Status**: `True` / `False`  
- **Reason**: `UpdateInProgress` / `UpdateCompleted`  
- **Message**: “正在更新版本 v2.0” / “更新完成”  
- **LastTransitionTime**: 更新时间戳  
### 5. **Degraded**
- **Type**: `Degraded`  
- **Status**: `True` / `False`  
- **Reason**: `ComponentFailure` / `ResourceExhausted`  
- **Message**: “组件 A 出现错误” / “资源不足导致性能下降”  
- **LastTransitionTime**: 更新时间戳  
## 📊 推荐字段组合模板（CRD 可直接套用）

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-04-20T09:00:00Z"
    reason: ComponentsReady
    message: All components are ready
  - type: Scheduled
    status: "True"
    lastTransitionTime: "2026-04-20T08:59:00Z"
    reason: SchedulerSuccess
    message: Resource successfully scheduled
  - type: Initialized
    status: "True"
    lastTransitionTime: "2026-04-20T08:58:00Z"
    reason: InitCompleted
    message: Initialization completed
  - type: Progressing
    status: "False"
    lastTransitionTime: "2026-04-20T08:57:00Z"
    reason: UpdateCompleted
    message: No update in progress
  - type: Degraded
    status: "False"
    lastTransitionTime: "2026-04-20T08:56:00Z"
    reason: Healthy
    message: System is healthy
```
## ✅ 总结
- **统一字段组合**：`Type` + `Status` + `Reason` + `Message` + `LastTransitionTime`。  
- **布尔化条件**：保持 `True/False/Unknown`，避免复杂枚举。  
- **监控友好**：条件设计应支持 `kubectl wait` 和监控告警。  
