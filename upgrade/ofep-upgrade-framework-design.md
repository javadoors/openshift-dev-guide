---
title: 升级框架声明式改造
ofep-number: 0001
authors:
  - "@cluster-api-provider-bke-maintainers"
owning-sig: sig-bke
participating-sigs:
  - sig-cluster-lifecycle
status: provisional
creation-date: 2026-05-06
reviewers:
  - TBD
approvers:
  - TBD
see-also:
  - "/ofeps/sig-bke/0000-template"
replaces: []
replaced-by: []
stage: alpha
latest-milestone: "v1.0"
milestone:
  alpha: "v1.0"
  beta: "v1.1"
  stable: "v1.3"
feature-gates:
  - name: DeclarativeUpgradeFramework
    components:
      - bke-installer-controller
      - bke-upgrade-controller
disable-supported: true
metrics:
  - bke_upgrade_framework_phase_duration_seconds
  - bke_upgrade_framework_component_reconcile_total
  - bke_upgrade_framework_component_failed_total
---

# oFEP-0001：升级框架声明式改造

<!-- toc -->
- [发布签核清单](#发布签核清单)
- [摘要](#摘要)
- [动机](#动机)
  - [目标](#目标)
  - [非目标](#非目标)
- [提案](#提案)
  - [用户故事（可选）](#用户故事可选)
  - [注释/限制/注意事项（可选）](#注释限制注意事项可选)
  - [风险与缓解措施](#风险与缓解措施)
- [设计细节](#设计细节)
  - [版本部署包](#版本部署包)
  - [版本包使用方式（参考 OpenShift）](#版本包使用方式参考-openshift)
  - [集群升级处理（参考 OpenShift）](#集群升级处理参考-openshift)
  - [声明式组件模型](#声明式组件模型)
  - [旧 phase 到声明式组件映射](#旧-phase-到声明式组件映射)
  - [旧架构升级到新架构策略](#旧架构升级到新架构策略)
  - [测试计划](#测试计划)
  - [毕业标准](#毕业标准)
  - [升级/降级策略](#升级降级策略)
  - [版本倾斜策略](#版本倾斜策略)
- [生产可用性审查](#生产可用性审查)
  - [功能启用和回滚](#功能启用和回滚)
  - [推出、升级和回滚规划](#推出升级和回滚规划)
  - [监控要求](#监控要求)
  - [依赖项](#依赖项)
  - [可扩展性](#可扩展性)
  - [故障排除](#故障排除)
- [工作量评估](#工作量评估)
  - [评估口径](#评估口径)
  - [升级式组件框架工作量](#升级式组件框架工作量)
  - [各 phase 改造工作量](#各-phase-改造工作量)
  - [老架构到新架构升级迁移工作量](#老架构到新架构升级迁移工作量)
  - [汇总与排期建议](#汇总与排期建议)
- [实施历史](#实施历史)
- [缺点](#缺点)
- [替代方案](#替代方案)
- [所需基础设施（可选）](#所需基础设施可选)
<!-- /toc -->

## 发布签核清单

标记有（R）的项目在达到里程碑前必需完成。

- [ ]（R）发布里程碑中的增强问题已关联本 oFEP
- [ ]（R）oFEP 审批者已批准状态为 implementable
- [x]（R）设计细节已记录
- [ ]（R）测试计划已覆盖单元、集成、e2e
- [ ]（R）毕业标准已设定并达成
- [ ]（R）生产可用性审查完成并批准
- [ ]（R）实施历史按里程碑更新
- [ ]（R）面向用户文档已在文档仓库发布

## 摘要

本提案引入一个声明式升级框架，用于统一组件安装、升级和状态检测流程。框架通过版本部署包（Release Bundle）和声明式组件定义（ReleaseImage）驱动执行，将现有基于 phase 的流程逐步改造为组件级编排流程，实现更稳定的升级行为、更清晰的依赖关系管理，以及可观测的升级执行过程。

## 动机

现有安装与升级逻辑以 phase 为中心，组件边界不清晰，升级路径和安装路径复用程度低，导致以下问题：

- 组件依赖难以显式表达，升级失败时定位成本高。
- 新增组件或版本时，流程改动点分散，维护成本高。
- 缺少统一的版本包语义，难以进行可回放、可审计的升级操作。

### 目标

1. 定义标准化版本部署包，显式声明组件、版本与依赖关系。
2. 建立统一的声明式组件执行引擎，支持安装与升级两类动作。
3. 将旧 phase 流程映射到组件模型，支持分阶段迁移。
4. 保证升级过程具备幂等性、可观测性和可回滚能力。

### 非目标

1. 本提案不重构现有证书签发与管理机制（仍沿用老证书流程）。
2. 本提案不在首个版本中引入全新的负载均衡升级逻辑（先沿用旧规格）。
3. 本提案不一次性替换所有旧控制器行为，控制器内置公共阶段保持不变。

## 提案

本提案采用“版本包 + 声明式组件 + 兼容映射”三层结构：

1. **版本包层：** 管理不同版本的模板、安装脚本与 release 元数据。
2. **组件声明层：** 使用 ReleaseImage 定义组件类型、镜像、清单和依赖。
3. **执行与兼容层：** 将旧 phase 拆解为安装动作与升级动作，并映射到组件执行图。

升级控制器读取目标版本的 ReleaseImage，基于依赖拓扑进行组件编排，对每个组件执行 pre-check、apply/upgrade、health-check、结果上报，最终形成完整升级状态。

### 用户故事（可选）

#### 故事 1

作为集群运维人员，我希望在升级前明确看到目标版本包含哪些组件、各组件依赖关系是什么，这样我可以评估变更影响并制定窗口计划。

#### 故事 2

作为平台开发人员，我希望新增一个组件时只需要在版本包中声明组件定义和依赖关系，而不需要修改多个 phase 分支逻辑。

### 注释/限制/注意事项（可选）

- `inline` 类型组件由控制器内置逻辑执行，适用于核心流程组件。
- `script` 类型组件由 manifests 中脚本或 YAML 驱动，适用于扩展组件。
- 第一阶段允许旧 phase 和新组件编排并存，以降低切换风险。

### 风险与缓解措施

1. **风险：** 组件依赖图配置错误导致升级阻塞。  
   **缓解：** 在版本包校验阶段引入 DAG 检查和必需依赖检查。
2. **风险：** 旧 phase 与新组件并行期间出现重复执行。  
   **缓解：** 引入组件级幂等标记与阶段互斥保护。
3. **风险：** 升级失败后状态不一致。  
   **缓解：** 所有组件执行都落盘状态并支持重入恢复。

## 设计细节

### 版本部署包

版本部署包至少包含以下内容：

- 不同组件版本对应的配置模板文件。
- 安装/升级使用的脚本或 YAML 模板文件。
- `release` 元数据文件，声明组件名称、版本、类型和依赖关系。

使用逻辑：通过openFuyao版本号->openfuyao_release.yaml->找到组件列表->构建DAG->并行执行（安装、卸载、升级、健康检查）

推荐目录结构：

```text
release-bundles/
  v1.0.0/
    openfuyao_base.yaml               # 模板层，定义可复用的组件集合
    openfuyao_management_cluster.yaml # 管理集群，定义独有的组件集合
    openfuyao_workload_cluster.yaml   # 业务集群，定义独有的组件集合
    components/
      app-operator/
        manifests/
        scripts/
```

简易说明（各文件含义与作用）：

| 路径 | 含义 | 作用 |
| --- | --- | --- |
| `release-bundles/` | 版本包根目录 | 统一存放所有版本部署资产，保证版本可追溯、可回放。 |
| `release-bundles/v1.0.0/` | 指定版本目录 | `v1.0.0` 的完整版本快照，描述该版本安装/升级所需内容。 |
| `release-bundles/v1.0.0/openfuyao_base.yaml` | 模板层文件 | 定义可复用组件集合（通常是共享控制面组件）及依赖关系。 |
| `release-bundles/v1.0.0/openfuyao_management_cluster.yaml` | 管理集群实例文件 | 在模板基础上补充管理集群独有组件，形成管理集群最终组件清单。 |
| `release-bundles/v1.0.0/openfuyao_workload_cluster.yaml` | 业务集群实例文件 | 在模板基础上补充业务集群独有组件，形成业务集群最终组件清单。 |
| `release-bundles/v1.0.0/components/` | 组件资产目录 | 存放组件安装、升级、健康检查所需的 manifests 与脚本。 |
| `release-bundles/v1.0.0/components/app-operator/` | 单组件目录 | `app-operator` 组件独立资产目录，便于单组件维护和演进。 |
| `release-bundles/v1.0.0/components/app-operator/manifests/` | 资源清单目录 | 存放组件 YAML 资源清单，用于声明式下发资源。 |
| `release-bundles/v1.0.0/components/app-operator/scripts/` | 脚本目录 | 存放安装/升级辅助脚本，用于迁移、检查、回滚等流程控制。 |

### 版本包使用方式（参考 OpenShift）

参考 OpenShift 的 Release Image 思路，版本包在本方案中的使用流程如下：

1. **选择目标版本：** 控制器根据集群期望版本定位 `release-bundles/<version>/` 目录。
2. **加载模板层：** 读取 `openfuyao_base.yaml`，得到共享组件集合（如控制面组件）及依赖关系。
3. **加载实例层：** 根据集群类型读取 `openfuyao_management_cluster.yaml` 或 `openfuyao_workload_cluster.yaml`。
4. **模板展开合并：** 将模板组件与实例组件合并，校验组件重名、字段完整性、版本约束。
5. **构建依赖 DAG：** 基于 `dependencies` 生成有向无环图，得到可执行顺序和并行批次。
6. **执行组件动作：** 按组件类型执行 `inline/script/manifests/helm` 动作，统一做 pre-check、apply/upgrade、health-check。
7. **回写升级状态：** 将执行结果回写到 `status.availableUpdates` 和 `status.upgradePlan`，并记录组件级状态。
8. **失败恢复与重入：** 失败时依据组件状态重试或回滚，支持从检查点继续推进而非全量重跑。

推荐实践：

- 模板层只放可复用公共组件，实例层只放集群特有组件，避免职责混杂。
- 每次版本发布都应冻结 `release-bundles/<version>/` 内容，禁止原地修改历史版本包。
- 在 CI 中增加“模板展开 + DAG 校验 + 干跑（dry-run）”三类校验，提前拦截错误版本包。

### 集群升级处理（参考 OpenShift）

参考 OpenShift Cluster Version Operator（CVO）的处理思路，集群升级按“声明目标版本 -> 生成升级计划 -> 分层执行与观测 -> 失败可恢复”推进。

#### 1）声明升级目标

- 用户或系统在集群对象中声明目标版本（例如 `spec.desiredVersion`）。
- 控制器基于 `status.availableUpdates` 判断目标版本是否可达。
- 若目标版本不可达（缺少版本包、版本不兼容），直接阻断并上报条件状态。

#### 2）生成 UpgradePlan

- 读取目标版本对应版本包（模板层 + 实例层），展开得到全量组件清单。
- 对比当前运行组件与目标组件，计算变更集（新增、变更、保持、下线）。
- 基于 `dependencies` 构建 DAG，形成执行批次，并写入 `status.upgradePlan`（`Pending`）。

#### 3）执行前门禁检查

- 集群健康门禁：API Server、etcd、关键控制器健康。
- 版本门禁：组件镜像可拉取、配置兼容、CRD 可用。
- 容量门禁：关键节点资源满足升级窗口要求。
- 门禁失败时不进入执行阶段，保留 `Pending` 并给出失败原因。

#### 4）分批执行升级（CVO 风格分层）

- 按 DAG 拓扑执行，批次内并行、批次间串行。
- 每个组件统一执行链路：`PreCheck -> ApplyOrUpgrade -> HealthCheck -> ReportStatus`。
- 对 `inline/script/manifests/helm` 统一封装执行器接口，保证行为一致。
- 每个组件完成后立即回写状态：`Pending/Running/Succeeded/Failed`。

#### 5）失败处理与回滚策略

- 组件失败先进行限次重试（带退避）。
- 超过阈值后升级任务进入 `Paused` 或 `Failed`，并停止后续批次。
- 支持两类恢复策略：
  - 自动回滚：对支持回滚的组件恢复到上一个稳定版本。
  - 人工介入：保留现场状态，允许运维修复后从检查点续跑。

#### 6）重入与幂等

- 控制器重启或任务中断后，从 `status.upgradePlan` 与组件状态恢复进度。
- 已成功组件默认跳过，不重复执行破坏性动作。
- 保障“多次触发同一目标版本升级，结果一致”。

#### 7）升级完成与验收

- 全部组件达到 `Succeeded` 后，更新集群当前版本为目标版本。
- 升级计划置为 `Succeeded`，并记录升级耗时、失败重试次数和关键事件。
- 输出升级摘要，供审计与后续容量规划使用。

#### 8）建议状态字段（示例）

```yaml
status:
  currentVersion: "v1.0.0"
  availableUpdates:
    - version: "v1.1.0"
      image: "registry.example.com/release:1.1.0"
  upgradePlan:
    targetVersion: "v1.1.0"
    state: "Running" # Pending/Running/Paused/Failed/Succeeded
    startedAt: "2026-05-06T17:00:00+08:00"
    finishedAt: ""
    componentStatuses:
      - name: kube-apiserver
        phase: Succeeded
      - name: app-operator
        phase: Running
```

### 声明式组件模型

参考 OpenShift 风格的设计，本提案采用 `ReleaseImageTemplate + ReleaseImage` 的双层模型，支持管理集群与工作集群复用共享控制面组件，支持的类型有inline/script/manifests/helm。

#### 1）ReleaseImageTemplate（模板层）

用于定义可复用的组件集合（如 etcd、kube-apiserver、kube-controller-manager）：

```yaml
apiVersion: release.example.com/v1alpha1
kind: ReleaseImageTemplate
metadata:
  name: control-plane-template
spec:
  version: "1.0.0"
  description: "Shared control plane components"
  components:
    - name: etcd
      type: core
      version: "3.5.10"
      image: "registry.example.com/etcd:3.5.10"
      manifests:
        - config/etcd.yaml
      dependencies: []
    - name: kube-apiserver
      type: core
      version: "1.29.3"
      image: "registry.example.com/kube-apiserver:1.29.3"
      manifests:
        - config/kube-apiserver.yaml
      dependencies:
        - etcd
```

#### 2）ReleaseImage（实例层）

用于描述具体集群版本，可引用多个模板并补充集群特有组件：

```yaml
apiVersion: release.example.com/v1alpha1
kind: ReleaseImage
metadata:
  name: workload-cluster-release-1.0.0
spec:
  clusterType: workload
  version: "1.0.0"
  releaseTemplateRefs:
    - name: control-plane-template
  components:
    - name: app-operator
      type: addon
      version: "2.0.0"
      image: "registry.example.com/app-operator:2.0.0"
      manifests:
        - addons/app-operator.yaml
      dependencies:
        - kube-apiserver
status:
  availableUpdates:
    - version: "1.1.0"
      image: "registry.example.com/release:1.1.0"
  upgradePlan:
    targetVersion: "1.1.0"
    state: "Pending"
```

关键约束：

- `ReleaseImageTemplate.spec.components[].name` 在模板内必须唯一。
- `ReleaseImage.spec.components[].name` 在实例内必须唯一；与模板展开后的全量组件名必须唯一。
- 依赖关系必须构成有向无环图（DAG），并在模板展开后再次校验。
- `releaseTemplateRefs` 解析失败或版本不匹配时，升级任务必须阻断。
- 组件执行状态统一记录为：`Pending`、`Running`、`Succeeded`、`Failed`。
- `status.availableUpdates` 与 `status.upgradePlan` 作为升级决策和进度观测的标准入口。

### 旧 phase 到声明式组件映射

组件接口如下：

```go
type ComponentVersionExecutor struct {
    install()
    uninstall()
    upgrade()
    roolback()
    healthzCheck()
    nodeSelector()
}
```

按“安装 + 升级”双路径映射：

| 安装流程 phase | 升级流程 phase | 说明 |
| --- | --- | --- |
| `EnsureFinalizer` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsurePaused` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsureClusterManage` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsureDeleteOrReset` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsureDryRun` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsureBKEAgent` | `EnsureAgentUpgrade` | BKEAgent 安装与升级处理。 |
| `EnsureNodesEnv`（里面包含 containerd） | 不涉及 | 节点环境准备仅在安装流程。 |
| `EnsureContainerd`（新增） | `EnsureContainerdUpgrade` | containerd 从 `EnsureNodesEnv` 拆分为独立 phase。 |
| `EnsureClusterAPIObj` | 不涉及 | 生成集群安装部署的Cluster API资源                            |
| `EnsureCerts` | `NA`（还用老证书） | 证书流程暂不改造。 |
| `EnsureLoadBalance`（生成 HA 配置） | `NA`（现在升级没有做，先沿用旧规格） | 升级阶段暂不引入新负载均衡升级流程。 |
| `EnsureEtcd`（新增，仅生成静态 Pod 文件，etcd 安装在 `EnsureMasterInit` 完成） | `EnsureEtcdUpgrade` | etcd 安装与升级职责拆分。 |
| `EnsureMasterInit`（安装 kubelet/kubectl、生成静态 Pod 等核心组件安装） | `EnsureMasterUpgrade`（也包含 kube-proxy 升级，kube-proxy 通过 Addon 安装） | 控制面主节点初始化与升级映射。 |
| `EnsureMasterJoin` | `EnsureMasterUpgrade` | 控制面扩容节点复用升级 phase。 |
| `EnsureKubeProxy`（新增） | `EnsureKubeProxyUpgrade`（新增） | 新增独立组件 phase，将kube-proxy的升级从`EnsureMasterUpgrade`阶段分离 |
| `EnsureCoreDNS`（新增） | `EnsureCoreDNSUpgrade`（新增） | 新增独立组件 phase。 |
| `EnsureCalico`（新增） | `EnsureCalicoUpgrade`（新增） | 新增独立组件 phase。 |
| `EnsureWorkerJoin` | `EnsureWorkerUpgrade` | 工作节点加入与升级映射。 |
| `EnsureAddonDeploy` | `EnsureComponentUpgrade`（后续组件拆解） | Addon 升级后续下沉到组件级。 |
| `EnsureNodesPostProcess` | 不涉及 | 直接内置在控制器代码逻辑中。 |
| `EnsureClusterAPIManagerManifest` | 不涉及 | 直接内置在控制器代码逻辑中。 |

### 旧架构升级到新架构策略

迁移采用三阶段：

1. **阶段 A（并存期）：** 新增声明式组件执行器，旧 phase 继续可用。
2. **阶段 B（切换期）：** 默认走新执行器，旧 phase 作为回退路径。
3. **阶段 C（收敛期）：** 下线可替代旧 phase，仅保留控制器公共内置阶段。

每个阶段必须满足：

- 支持“升级失败重试不重复破坏”。
- 提供等价功能验证用例。
- 提供一键回退到上一稳定路径的操作说明。

### 测试计划

[x] 我们理解组件所有者可能要求补充测试后再合入实现。

##### 先决条件测试更新

- 增加版本包校验测试（字段完整性、依赖 DAG、组件重复名）。

##### 单元测试

- 组件依赖排序与循环检测。
- 组件状态机转换逻辑。
- 安装与升级动作路由（inline/script）分发。
- 失败重试与幂等控制。

- `internal/upgrade/framework`: `2026-05-06` - `TBD`

##### 集成测试

- 目标：验证同一版本包下安装与升级路径均可收敛到一致状态。
- 测试名称：`TestUpgradeFramework_InstallThenUpgrade`
- 测试名称：`TestUpgradeFramework_ComponentFailureRetry`

##### e2e 测试

- 测试名称：`[e2e] declarative-upgrade-framework-happy-path`
- 测试名称：`[e2e] declarative-upgrade-framework-rollback-path`

### 毕业标准

#### Alpha

- Feature Gate 默认关闭，可按集群开启。
- 核心组件（etcd、master、worker、kube-proxy、coredns）完成声明式路径。
- 关键单元测试与集成测试齐备。

#### Beta

- 默认启用新升级路径，旧路径可配置回退。
- e2e 测试稳定运行至少 2 个迭代周期。
- 监控指标与告警规则达到生产可观测要求。

#### Stable

- 生产环境运行反馈稳定，无阻塞级缺陷。
- 回滚演练通过，故障恢复手册完善。
- 旧可替代 phase 完成下线。

### 升级/降级策略

- 升级到新框架时，不强制用户修改集群 API；仅需切换 Feature Gate 或控制器配置。
- 降级时支持关闭 `DeclarativeUpgradeFramework` 并恢复旧路径执行。
- 对于已部分执行的新组件，按组件状态恢复点重入或回退，不做全量重装。

### 版本倾斜策略

- 控制器按 `n-1` 版本兼容版本包 schema。
- 组件执行端与控制器通过明确的版本字段协商，未知字段忽略并告警。
- 在混部升级阶段，优先保证 control-plane 组件升级兼容，再推进 worker 组件。

## 生产可用性审查

### 功能启用和回滚

###### 如何在实时集群中启用/禁用此功能？

- [x] **功能开关（Feature gate）**
  - 功能开关名称：`DeclarativeUpgradeFramework`
  - 依赖该功能开关的组件：`bke-installer-controller`、`bke-upgrade-controller`
- [ ] **其他机制**

###### 启用该功能会改变任何默认行为吗？

会。升级流程从 phase 主导转为组件编排主导，但功能目标保持等价。

###### 该功能一旦启用，是否可以禁用？

可以。关闭 Feature Gate 并重启控制器后恢复旧路径；已执行组件状态保留用于审计。

###### 如果该功能之前已回滚，现在我们重新启用它会发生什么情况？

控制器根据组件状态恢复点继续执行，已成功组件不会重复执行破坏性动作。

###### 是否有任何针对功能启用/禁用的测试？

有。包含 Feature Gate 开关前后对象行为一致性测试与重入测试。

### 推出、升级和回滚规划

###### 部署或回滚为何会失败？这会影响正在运行的工作负载吗？

可能因版本包错误、依赖错误或组件健康检查失败而回滚。已运行工作负载原则上不受影响，但控制面升级失败可能导致控制平面短时抖动。

###### 哪些具体指标应该通知回滚？

- `bke_upgrade_framework_component_failed_total` 在短窗口内持续上升。
- 组件 `Running` 状态超时比例超过阈值。
- 控制面关键组件健康检查失败率超阈值。

###### 升级和回滚测试了吗？升级->降级->升级的路径测试了吗？

已纳入集成和 e2e 计划，要求覆盖升级->降级->升级闭环验证。

###### 推出时是否伴随任何功能、API、字段、标志的弃用或删除？

首阶段不删除旧 API，仅新增 Feature Gate 与组件状态字段。

### 监控要求

###### 操作员如何确定该功能是否正在被工作负载使用？

通过控制器指标和组件状态对象判断，不依赖日志检索。

###### 使用此功能的人如何知道它适用于他们的实例？

- [ ] 事件
- [x] API .状态
  - 条件名称：`UpgradeFrameworkReady`
  - 其他字段：`componentStatuses`
- [ ] 其他

###### 增强的合理 SLO 是什么？

- 升级任务成功率 >= 99%
- 核心组件单次升级 99 分位耗时 <= 15 分钟

###### 运维人员可以使用哪些 SLI 判断健康？

- [x] 指标
  - 指标名称：`bke_upgrade_framework_phase_duration_seconds`
  - 聚合方法：按组件名和动作类型聚合 P95/P99
  - 暴露组件：`bke-upgrade-controller`
- [ ] 其他

###### 是否存在尚未覆盖的指标？

存在。组件依赖等待时长分布暂未提供，后续在 Beta 阶段补齐。

### 依赖项

- Kubernetes API Server
  - 使用说明：
    - 若中断：组件状态无法推进，升级任务暂停。
    - 若性能下降：状态写入延迟导致升级时长增加。
- Etcd
  - 使用说明：
    - 若中断：状态存储不可用，升级中断并触发保护退出。
    - 若性能下降：控制循环重试增多。

### 可扩展性

###### 启用/使用此功能会导致任何新的 API 调用吗？

会。新增组件状态对象的 `GET/LIST/WATCH/PATCH`，由升级控制器发起，频率随组件数量线性增长。

###### 启用/使用此功能是否会导致引入新的 API 类型？

会。新增 `ReleaseImage` 与组件执行状态相关类型（具体以实现阶段 CRD 命名为准）。

###### 启用/使用此功能是否会导致对云提供商的任何新的 API 调用？

不会直接新增云厂商 API 调用。

###### 启用/使用此功能是否会导致现有 API 对象大小或数量增加？

会。每个升级任务会增加组件状态记录对象；对象大小与组件数成正比。

###### 启用/使用此功能是否会导致现有 SLI/SLO 所涵盖操作耗时增加？

可能略有增加（主要来自状态持久化和健康检查），但通过并发调度和批处理控制在可接受范围。

###### 启用/使用此功能是否会导致资源使用率不可忽略增加？

会有控制器 CPU 和内存的有限增加，预计在中大型集群仍可控；需在 Beta 阶段补充压测数据。

###### 启用/使用此功能是否会导致节点资源耗尽？

正常情况下不会。异常重试风暴场景通过速率限制和退避机制缓解。

### 故障排除

###### 如果 API 服务器和/或 etcd 不可用，此功能如何反应？

控制器进入重试等待并上报不可达状态，不继续推进升级步骤。

###### 其他已知故障模式有哪些？

- **版本包依赖环**
  - **检测方式：** 版本包校验指标与事件告警
  - **缓解手段：** 阻断升级并回滚到上一个可用版本包
  - **诊断信息：** 控制器 `Error` 日志包含依赖链路
  - **测试情况：** 有单元测试
- **组件长时间未就绪**
  - **检测方式：** 组件运行超时指标
  - **缓解手段：** 触发组件重试或人工介入回滚
  - **诊断信息：** 健康检查失败原因记录在状态字段
  - **测试情况：** 有集成测试

###### 如果未满足 SLO，应采取哪些步骤来确定问题？

先定位失败组件与依赖路径，再结合阶段耗时指标与事件，判断是依赖等待、执行失败还是健康检查超时，并按组件粒度重试或回滚。

## 工作量评估

### 评估口径

- 工作量单位为人天（1 人天 = 1 名熟悉本项目代码的开发 1 天有效工作）。
- 评估范围包含设计落地、编码实现、单元测试、基础联调与自测。
- 评估范围不包含跨团队依赖等待、发布窗口排队、长周期灰度观察。

### 升级式组件框架工作量

| 模块 | 预估人天 | 说明 |
| --- | ---: | --- |
| Release Bundle 规范与加载器 | 3 | 版本包结构定义、release 元数据解析与校验入口。 |
| 模板与实例模型（ReleaseImageTemplate/ReleaseImage） | 6 | 模板定义、实例引用、模板展开与冲突校验。 |
| 升级状态模型（availableUpdates/upgradePlan） | 3 | 升级候选版本发现、升级计划状态流转。 |
| 组件编排执行器（DAG + 状态机） | 7 | 模板展开后的全量依赖拓扑、执行调度、失败中断与重试。 |
| 组件执行插件层（inline/script） | 4 | 安装与升级动作统一接口和执行路由。 |
| 幂等与重入恢复机制 | 3 | 断点续跑、重复执行保护、失败恢复。 |
| 可观测性（指标/事件/日志） | 2 | 阶段耗时、失败计数、关键事件与日志标准化。 |
| Feature Gate 与新旧路径切换 | 3 | 新旧流程切换、回退控制、兼容入口。 |
| 框架联调与基线回归 | 3 | 最小可用闭环验证、基础回归。 |

小计：34 人天。

### 各 phase 改造工作量

| phase 改造项 | 预估人天 | 说明 |
| --- | ---: | --- |
| `EnsureFinalizer`（内置） | 0.5 | 与新流程入口和状态更新对齐。 |
| `EnsurePaused`（内置） | 0.5 | 暂停/恢复语义与调度流程兼容。 |
| `EnsureClusterManage`（内置） | 1 | 管理阶段与组件编排边界梳理。 |
| `EnsureDeleteOrReset`（内置） | 1 | 重置、删除、回退保护策略适配。 |
| `EnsureDryRun`（内置） | 1 | dry-run 路径覆盖新组件调度。 |
| `EnsureBKEAgent -> EnsureAgentUpgrade` | 2 | Agent 安装/升级动作拆分与状态映射。 |
| `EnsureNodesEnv -> NA` | 0.5 | 保留策略确认与文档固化。 |
| `EnsureContainerd -> EnsureContainerdUpgrade`（新增） | 3 | 从 NodesEnv 拆分独立 phase 并接入升级流程。 |
| `EnsureClusterAPIObj -> NA(apply yaml)` | 1 | 兼容现有 apply 行为与适配层补充。 |
| `EnsureCerts -> NA` | 0.5 | 老证书流程兼容验证与风险说明。 |
| `EnsureLoadBalance -> NA` | 1 | HA 旧规格兼容校验，升级阶段暂不重构。 |
| `EnsureEtcd -> EnsureEtcdUpgrade`（新增） | 4 | etcd 安装职责拆分与升级解耦。 |
| `EnsureMasterInit -> EnsureMasterUpgrade` | 5 | 控制面核心安装升级路径改造。 |
| `EnsureMasterJoin -> EnsureMasterUpgrade` | 2 | master join 逻辑与 upgrade 统一。 |
| `KubeProxy`（新增） | 2 | 新组件安装/升级与状态上报。 |
| `CoreDNS`（新增） | 2 | 新组件安装/升级与状态上报。 |
| `EnsureWorkerJoin -> EnsureWorkerUpgrade` | 3 | worker 升级编排与幂等控制。 |
| `EnsureAddonDeploy -> EnsureComponentUpgrade` | 4 | Addon 逐步拆解为组件级升级。 |
| `EnsureNodesPostProcess`（内置） | 1 | 后处理阶段收敛和兼容。 |
| `EnsureAgentSwitch` 去除并替换新阶段 | 2 | 旧逻辑下线与新阶段迁移。 |

小计：37 人天。

### 老架构到新架构升级迁移工作量

| 迁移专项 | 预估人天 | 说明 |
| --- | ---: | --- |
| 存量集群识别与分层策略 | 1 | 识别旧架构集群范围，按版本和拓扑分层。 |
| 旧状态到新模型兼容读取 | 2 | 控制器兼容读取旧对象与旧 phase 状态。 |
| 迁移转换器（旧对象 -> 模板实例+组件状态） | 4 | 一次性转换与增量补偿，保证幂等。 |
| 升级迁移编排（A/B/C 三阶段） | 4 | 并存、切换、收敛阶段的执行门禁与开关。 |
| 迁移失败回滚机制 | 2 | 回滚到旧路径、恢复检查点与保护策略。 |
| 模板引用一致性修复工具 | 2 | 模板缺失、版本漂移、组件重名的修复工具。 |
| 迁移专项测试（升级->回滚->再升级） | 4 | 集成测试和关键场景验证。 |
| 灰度与观测策略落地 | 1 | 分批放量与告警阈值策略。 |

小计：20 人天。

### 汇总与排期建议

- 升级式组件框架：34 人天。
- 各 phase 改造：37 人天。
- 老架构到新架构升级迁移：20 人天。
- 开发与基础联调合计：91 人天。
- 建议风险缓冲（20%）：18 人天。
- 建议总体排期：109 人天。

建议人力投入：

- 1 人并行：约 21 到 22 周。
- 2 人并行：约 11 周。
- 3 人并行：约 7 到 8 周。

## 实施历史

- 2026-05-06：完成 oFEP 初稿，明确声明式升级框架目标与迁移路径。

## 缺点

- 架构复杂度上升，需要维护版本包和组件模型。
- 迁移期存在双路径维护成本。
- 对运维团队提出更高的版本包治理要求。

## 替代方案

1. **继续沿用纯 phase 架构：** 改造成本低，但长期可维护性差。
2. **一次性全量替换旧流程：** 交付速度快，但迁移风险过高。
3. **仅改造升级不改造安装：** 短期可行，但安装升级模型分裂，后续负担更大。

## 所需基础设施（可选）

- 新增版本包存储目录及校验流水线。
- 新增升级框架指标采集与告警规则模板。
