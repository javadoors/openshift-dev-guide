# OLM
# OpenShift Operator Lifecycle Manager (OLM) 概要设计
## 1. 系统概述
**Operator Lifecycle Manager (OLM)** 是 OpenShift Container Platform 的核心组件,用于安装、更新和管理集群中运行的所有 Operator 及其关联服务的生命周期。OLM 是 Operator Framework 的一部分,提供声明式的方式来管理 Kubernetes 原生应用程序。
### 1.1 核心价值
- **自动化管理**: 自动安装、升级 Operator 及其依赖
- **声明式配置**: 使用 Kubernetes 原生 API 管理 Operator 生命周期
- **多租户支持**: 通过命名空间隔离实现多租户管理
- **依赖解析**: 自动解析和安装 Operator 所需的依赖
- **版本控制**: 通过频道(Channel)管理 Operator 版本更新
### 1.2 默认运行环境
OLM 默认在 OpenShift Container Platform 中运行,提供:
- Web 控制台管理界面供集群管理员安装 Operator
- 项目授权机制,允许用户使用集群上的可用 Operator 目录
- 自助服务体验,开发人员无需成为专家即可置备数据库、监控等服务
## 2. 核心概念
### 2.1 ClusterServiceVersion (CSV)
**CSV 是描述 Operator 的主要元数据资源**,每个 CSV 表示 Operator 的一个版本。

**核心内容**:
- **元数据**: 名称、描述、版本(semver)、链接、标签、图标
- **安装策略**: 
  - 部署类型
  - 服务账户和所需权限集
  - 部署配置
- **CRD 定义**:
  - 自有 CRD(由该服务管理)
  - 必需 CRD(集群中必须存在才能运行)
  - 资源列表(Operator 与之交互的资源)
  - 描述符(注解 CRD 规格和状态字段以提供语义信息)

**类比理解**: CSV 类似于 Linux 软件包管理器中的 RPM 文件,包含安装 Operator 及其依赖项的信息。OLM 则类似于 yum 或 DNF 包管理工具。
### 2.2 CatalogSource
**CatalogSource 包含访问 Operator 存储库的信息**,存储 CSV、CRD 和 Package 的元数据。

**核心功能**:
- 使用 Operator Registry API 提供 Operator 目录服务
- OLM 通过 CatalogSource 查询可用的 Operator 和已安装 Operator 的升级版本
- Operator 被组织为更新软件包和更新流

**目录结构**:
```
CatalogSource
├── Package (软件包)
│   ├── Channel 1 (频道,如 alpha)
│   │   ├── CSV v0.1.1
│   │   ├── CSV v0.1.2
│   │   └── CSV v0.1.3 (频道头)
│   └── Channel 2 (频道,如 beta)
│       └── CSV v0.2.0
```
### 2.3 Subscription
**Subscription 是用户创建的用于安装和更新 Operator 的资源**,订阅特定 CatalogSource 中的特定软件包和频道。

**核心字段**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: etcd
  namespace: default
spec:
  channel: singlenamespace-alpha      # 订阅的频道
  installPlanApproval: Automatic      # 批准模式: Automatic/Manual
  name: etcd                          # 软件包名称
  source: operatorhubio-catalog       # CatalogSource 名称
  sourceNamespace: olm                # CatalogSource 所在命名空间
  startingCSV: etcdoperator.v0.9.2    # 起始 CSV
```
**批准模式**:
- **Automatic**: 自动安装和升级
- **Manual**: 需要手动审查和批准
### 2.4 InstallPlan
**InstallPlan 描述了 OLM 为满足 CSV 资源需求将创建的完整资源列表**。

**核心功能**:
- 计算安装或升级 CSV 所需的资源
- 列出将要创建的所有 Kubernetes 资源
- 对于 Manual 批准模式,用户在此资源上设置批准
### 2.5 OperatorGroup
**OperatorGroup 控制 Operator 的多租户性**,指定单个 Operator 可以访问的命名空间。

**核心功能**:
- 定义 Operator 的作用域(命名空间列表或集群范围)
- 防止 Operator 访问未授权的命名空间
- 实现多租户隔离
## 3. 架构设计
### 3.1 整体架构
```
┌─────────────────────────────────────────────────────────────┐
│                    OpenShift Cluster                         │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              OLM Namespace (olm)                      │   │
│  │                                                        │   │
│  │  ┌──────────────────┐      ┌──────────────────┐      │   │
│  │  │   OLM Operator   │      │ Catalog Operator │      │   │
│  │  │                  │      │                  │      │   │
│  │  │ - 监听 CSV       │      │ - 解析依赖       │      │   │
│  │  │ - 维护状态       │      │ - 安装 CSV       │      │   │
│  │  │ - 创建部署       │      │ - 监听目录更新   │      │   │
│  │  └──────────────────┘      └──────────────────┘      │   │
│  │           │                          │                │   │
│  │           └──────────┬───────────────┘                │   │
│  │                      │                                 │   │
│  │           ┌──────────▼──────────┐                     │   │
│  │           │   CatalogSource     │                     │   │
│  │           │   (Operator 目录)   │                     │   │
│  │           └─────────────────────┘                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         Target Namespace (用户命名空间)               │   │
│  │                                                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │   │
│  │  │Subscription  │  │ InstallPlan  │  │OperatorGroup│ │   │
│  │  └──────────────┘  └──────────────┘  └─────────────┘ │   │
│  │         │                  │                          │   │
│  │         └──────────┬───────┘                          │   │
│  │                    │                                  │   │
│  │         ┌──────────▼──────────┐                      │   │
│  │         │        CSV          │                      │   │
│  │         │  (Operator 元数据)  │                      │   │
│  │         └─────────────────────┘                      │   │
│  │                    │                                  │   │
│  │         ┌──────────▼──────────┐                      │   │
│  │         │   Operator Pod      │                      │   │
│  │         │   (实际运行的服务)  │                      │   │
│  │         └─────────────────────┘                      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```
### 3.2 核心控制器
#### 3.2.1 OLM Operator
**职责**:
- 监听 CSV 资源的变化
- 创建 CSV 中定义的应用部署
- 维护 Operator 的期望状态
- 如果 Operator 部署被删除,重新创建以恢复状态
#### 3.2.2 Catalog Operator
**职责**:
- 解析并安装 CSV 及其依赖资源
- 监听 CatalogSource 中频道的变化
- 将 Operator 更新到最新可用版本
- 创建和管理 InstallPlan
## 4. 关键工作流程
### 4.1 Operator 安装流程
```
用户创建 Subscription
        │
        ▼
Catalog Operator 解析订阅
        │
        ▼
查询 CatalogSource 获取最新 CSV
        │
        ▼
创建 InstallPlan (列出所需资源)
        │
        ▼
    批准模式?
   ╱          ╲
Automatic    Manual
   │           │
   │      用户审查批准
   │           │
   ▼           ▼
创建所需资源 (CSV, Deployment, RBAC 等)
        │
        ▼
OLM Operator 监听到 CSV
        │
        ▼
创建 Operator 部署
        │
        ▼
Operator 开始运行
```
### 4.2 Operator 升级流程
#### 4.2.1 基本升级路径
```
当前安装: CSV v0.1.1
        │
        ▼
OLM 查询 CatalogSource
        │
        ▼
发现升级路径: v0.1.3 → v0.1.2 → v0.1.1
        │
        ▼
安装 v0.1.2 (替换 v0.1.1)
        │
        ▼
安装 v0.1.3 (替换 v0.1.2)
        │
        ▼
升级完成,匹配频道头
```
**关键机制**:
- 每个 CSV 有 `replaces` 字段,指明替换的 Operator
- 构建可通过 OLM 查询的 CSV 图
- OLM 一次仅升级一个版本,直至到达频道头
#### 4.2.2 跳过升级
**场景**: 某些版本存在严重漏洞,需要跳过

**实现方式**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: etcdoperator.v0.9.2
spec:
  replaces: etcdoperator.v0.9.0
  skips:
  - etcdoperator.v0.9.1    # 跳过有问题的版本
```
**保证**:
- 任何 Operator 在新 CatalogSource 中均有单一替换项
- 如果未曾安装不良更新,将来也绝不会安装
#### 4.2.3 批量替换
**实现方式**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: elasticsearch-operator.v4.1.2
  annotations:
    olm.skipRange: '>=4.1.0 <4.1.2'    # 跳过版本范围
```
**优先级顺序**:
1. Subscription 指定的源中的频道头
2. 在指定源中替换当前 Operator 的下一 Operator
3. 其他可见源中的频道头
4. 其他可见源中替换当前 Operator 的下一 Operator
### 4.3 Z-stream 支持
**定义**: 对于相同主版本,Z-stream 或补丁版本必须取代所有先前 Z-stream 版本

**示例**:
```
v1.0.3 → v1.0.2 → v1.0.1
v1.0.3 替换 v1.0.2 和 v1.0.1
```
## 5. 与 Cluster Version Operator (CVO) 的关系
### 5.1 协作关系
```
┌─────────────────────────────────────────────┐
│         OpenShift Cluster                   │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │   Cluster Version Operator (CVO)     │  │
│  │                                      │  │
│  │  - 管理 OpenShift 核心组件           │  │
│  │  - 处理集群版本升级                  │  │
│  │  - 管理 Payload 中的 Operators       │  │
│  └──────────────────────────────────────┘  │
│                    │                        │
│                    ▼                        │
│  ┌──────────────────────────────────────┐  │
│  │   Operator Lifecycle Manager (OLM)   │  │
│  │                                      │  │
│  │  - 管理用户安装的 Operators          │  │
│  │  - 处理 Operator 生命周期            │  │
│  │  - 提供自助服务体验                  │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```
### 5.2 职责划分
| 组件 | CVO | OLM |
|------|-----|-----|
| **管理范围** | OpenShift 核心组件 | 用户安装的 Operators |
| **更新来源** | OpenShift Payload | CatalogSource |
| **更新控制** | 集群管理员 | 命名空间管理员 |
| **版本管理** | 集群版本 | Operator 版本 |
| **依赖管理** | Runlevel 顺序 | CSV 依赖解析 |
### 5.3 集成机制
1. **CVO 管理 OLM**: OLM 本身作为 OpenShift 核心组件,由 CVO 管理
2. **OLM 管理 Operators**: 用户通过 OLM 安装和管理额外的 Operators
3. **版本协调**: CVO 升级集群版本时,可能触发 OLM 升级相关 Operators
## 6. 核心特性
### 6.1 依赖管理
**依赖类型**:
- **必需 CRD**: 集群中必须存在才能运行
- **自有 CRD**: 由该 Operator 管理
- **资源依赖**: Operator 与之交互的资源列表

**自动解析**: Catalog Operator 自动解析依赖关系,创建 InstallPlan
### 6.2 多租户隔离
**实现机制**:
- OperatorGroup 定义 Operator 作用域
- 命名空间级别的访问控制
- RBAC 权限管理
### 6.3 安全性
**安全措施**:
- RBAC 权限控制
- 服务账户隔离
- 资源配额限制
- 安全上下文约束(SCC)
### 6.4 可观测性
**监控指标**:
- CSV 安装状态
- Subscription 更新状态
- InstallPlan 执行进度
- Operator 健康状态
## 7. 使用示例
### 7.1 安装 Operator
```bash
# 创建 Subscription
kubectl apply -f subscription.yaml

# subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: etcd
  namespace: default
spec:
  channel: singlenamespace-alpha
  installPlanApproval: Automatic
  name: etcd
  source: operatorhubio-catalog
  sourceNamespace: olm
```
### 7.2 查询可用 Operators
```bash
# 查询可用 Operators
kubectl -n olm get packagemanifests

# 输出示例
NAME                               CATALOG               AGE
akka-cluster-operator              Community Operators   19s
appsody-operator                   Community Operators   19s
etcd                               Community Operators   30s
```
### 7.3 手动批准升级
```bash
# 查看待批准的 InstallPlan
kubectl get installplan -n default

# 批准 InstallPlan
kubectl patch installplan <install-plan-name> \
  -n default \
  --type merge \
  -p '{"spec":{"approved":true}}'
```
## 8. 总结
**Operator Lifecycle Manager (OLM)** 是 OpenShift 生态系统的核心组件,提供了完整的 Operator 生命周期管理能力:

1. **声明式管理**: 使用 Kubernetes 原生 API 管理 Operator
2. **自动化流程**: 自动安装、升级和依赖解析
3. **版本控制**: 通过频道和 CSV 图管理版本更新
4. **多租户支持**: 通过 OperatorGroup 实现命名空间隔离
5. **安全可控**: RBAC 权限管理和手动批准机制
6. **用户友好**: Web 控制台和 CLI 工具支持

OLM 与 CVO 协同工作,共同构成了 OpenShift 的组件管理体系:CVO 负责核心组件,OLM 负责用户安装的 Operators,实现了分层管理和职责分离。

# Bundle 格式
在 Operator Lifecycle Manager (OLM) 中，**Bundle 格式**是目前推荐的 Operator 打包方式，逐步取代了早期的 `PackageManifest` 格式。它更标准化、可移植，并且与 Kubernetes 原生资源管理方式保持一致。  
## 📦 什么是 Bundle 格式
- **组成部分**：
  - **ClusterServiceVersion (CSV)**：描述 Operator 的版本、安装方式、权限、依赖等。
  - **CRDs (CustomResourceDefinitions)**：定义 Operator 管理的自定义资源。
  - **metadata/annotations.yaml**：包含包名、渠道、默认渠道等元信息。
- **存储形式**：每个 Bundle 是一个目录或镜像，里面包含上述文件。
- **分发方式**：Bundle 可以打包成 **OCI 镜像**，通过 CatalogSource 提供给 OLM。
## 📂 Bundle 目录结构示例
```
etcd-operator/
 ├── manifests/
 │    ├── etcdclusters.crd.yaml
 │    ├── etcdbackups.crd.yaml
 │    ├── etcdrestores.crd.yaml
 │    └── etcdoperator.v0.9.4.clusterserviceversion.yaml
 └── metadata/
      └── annotations.yaml
```

### `annotations.yaml` 示例
```yaml
annotations:
  operators.operatorframework.io.bundle.package.v1: etcd
  operators.operatorframework.io.bundle.channels.v1: stable,beta
  operators.operatorframework.io.bundle.channel.default.v1: stable
```
## 🔄 与 PackageManifest 的区别
| 特性 | PackageManifest | Bundle |
|------|----------------|--------|
| **格式来源** | OLM 内部生成的 CR | 独立目录或镜像 |
| **内容** | 包名、渠道、CSV 引用 | 完整的 CSV、CRDs、metadata |
| **分发方式** | CatalogSource 内置 | OCI 镜像，支持镜像仓库 |
| **状态** | 已逐步弃用 | 推荐使用，未来标准 |
## 🚀 优势
- **标准化**：与 Kubernetes 资源管理方式一致。
- **可移植**：通过镜像分发，易于跨环境使用。
- **可维护**：每个版本独立打包，清晰管理 Operator 生命周期。
- **兼容性**：支持 OperatorHub、OpenShift Catalog 等生态。

✅ **总结**：Bundle 格式是 OLM 的现代 Operator 打包方式，包含完整的 CSV、CRDs 和元信息，推荐在 OpenShift 及 Kubernetes 环境中使用。它比 `PackageManifest` 更标准化和可移植，是未来的主流。  

      
# OpenShift Operator Lifecycle Manager (OLM) 详细设计
## 1. 系统架构详细设计
### 1.1 整体架构图
```
┌─────────────────────────────────────────────────────────────────────────┐
│                          OpenShift Cluster                               │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    OLM Namespace (openshift-operators)              │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────┐    ┌──────────────────────────┐     │ │
│  │  │      OLM Operator        │    │    Catalog Operator      │     │ │
│  │  │                          │    │                          │     │ │
│  │  │  ┌────────────────────┐  │    │  ┌────────────────────┐  │     │ │
│  │  │  │ CSV Controller     │  │    │  │ Subscription Ctrl  │  │     │ │
│  │  │  │ - 监听 CSV 变化    │  │    │  │ - 监听 Subscription│  │     │ │
│  │  │  │ - 状态协调         │  │    │  │ - 创建 InstallPlan │  │     │ │
│  │  │  │ - 创建 Deployment  │  │    │  │ - 版本更新检测     │  │     │ │
│  │  │  └────────────────────┘  │    │  └────────────────────┘  │     │ │
│  │  │                          │    │                          │     │ │
│  │  │  ┌────────────────────┐  │    │  ┌────────────────────┐  │     │ │
│  │  │  │ OperatorGroup Ctrl │  │    │  │ InstallPlan Ctrl   │  │     │ │
│  │  │  │ - 多租户管理       │  │    │  │ - 依赖解析         │  │     │ │
│  │  │  │ - 命名空间选择     │  │    │  │ - 资源创建         │  │     │ │
│  │  │  └────────────────────┘  │    │  └────────────────────┘  │     │ │
│  │  │                          │    │                          │     │ │
│  │  │  ┌────────────────────┐  │    │  ┌────────────────────┐  │     │ │
│  │  │  │ OperatorCondition  │  │    │  │ CatalogSource Ctrl │  │     │ │
│  │  │  │ - 状态通信         │  │    │  │ - 目录同步         │  │     │ │
│  │  │  │ - 健康检查         │  │    │  │ - gRPC 客户端      │  │     │ │
│  │  │  └────────────────────┘  │    │  └────────────────────┘  │     │ │
│  │  └──────────────────────────┘    └──────────────────────────┘     │ │
│  │              │                              │                      │ │
│  │              └──────────────┬───────────────┘                      │ │
│  │                             │                                       │ │
│  │                  ┌──────────▼──────────┐                           │ │
│  │                  │  Catalog Registry   │                           │ │
│  │                  │  (gRPC Server)      │                           │ │
│  │                  │  - Package Manifest │                           │ │
│  │                  │  - CSV Storage      │                           │ │
│  │                  │  - Channel Index    │                           │ │
│  │                  └─────────────────────┘                           │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │              Marketplace Namespace (openshift-marketplace)          │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                    CatalogSource Pods                         │  │ │
│  │  │                                                                │  │ │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐ │  │ │
│  │  │  │ Red Hat Catalog │  │ Community Cat.  │  │ Certified Cat. │ │  │ │
│  │  │  │ (index image)   │  │ (index image)   │  │ (index image)  │ │  │ │
│  │  │  └─────────────────┘  └─────────────────┘  └────────────────┘ │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                Target Namespace (用户命名空间)                      │ │
│  │                                                                     │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │ │
│  │  │Subscription  │  │InstallPlan   │  │   OperatorGroup         │  │ │
│  │  │              │  │              │  │                         │  │ │
│  │  │- package     │  │- csv         │  │ - targetNamespaces      │  │ │
│  │  │- channel     │  │- approval    │  │ - selector              │  │ │
│  │  │- source      │  │- status      │  │ - serviceAccount        │  │ │
│  │  └──────────────┘  └──────────────┘  └─────────────────────────┘  │ │
│  │         │                  │                    │                  │ │
│  │         └──────────┬───────┴────────────────────┘                  │ │
│  │                    │                                               │ │
│  │         ┌──────────▼──────────┐                                   │ │
│  │         │        CSV          │                                   │ │
│  │         │                     │                                   │ │
│  │         │ - metadata          │                                   │ │
│  │         │ - install strategy  │                                   │ │
│  │         │ - crds              │                                   │ │
│  │         │ - status            │                                   │ │
│  │         └─────────────────────┘                                   │ │
│  │                    │                                               │ │
│  │         ┌──────────▼──────────┐                                   │ │
│  │         │   Operator Pod      │                                   │ │
│  │         │   (Deployment)      │                                   │ │
│  │         │                     │                                   │ │
│  │         │ - ServiceAccount    │                                   │ │
│  │         │ - RBAC Rules        │                                   │ │
│  │         │ - CRD Controllers   │                                   │ │
│  │         └─────────────────────┘                                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```
### 1.2 核心组件职责
#### 1.2.1 OLM Operator
**主要职责**:
- 管理 ClusterServiceVersion (CSV) 资源
- 确保 Operator 应用的正确部署和运行
- 维护 Operator 的期望状态
- 处理 OperatorGroup 的多租户隔离

**核心控制器**:
1. **CSV Controller**
   - 监听 CSV 资源的创建、更新、删除
   - 验证 CSV 中声明的依赖资源
   - 创建和管理 Operator 部署
   - 维护 CSV 状态机
2. **OperatorGroup Controller**
   - 管理命名空间选择器
   - 为 OperatorGroup 创建服务账户和 RBAC
   - 注入目标命名空间配置
   - 处理多租户隔离
3. **OperatorCondition Controller**
   - 监听 OperatorCondition 资源
   - 在 OLM 和 Operator 之间建立通信通道
   - 处理 Operator 报告的状态条件
#### 1.2.2 Catalog Operator
**主要职责**:
- 管理 Operator 目录、订阅和安装计划
- 解析 CSV 中声明的依赖资源
- 监听目录中频道的变化并完成版本更新
- 创建和管理 InstallPlan

**核心控制器**:
1. **Subscription Controller**
   - 监听 Subscription 资源
   - 查询 CatalogSource 获取最新 CSV
   - 创建 InstallPlan 进行安装或升级
   - 维护 Subscription 状态机
2. **InstallPlan Controller**
   - 解析依赖关系
   - 生成需要创建的资源列表
   - 处理手动/自动批准
   - 创建 CRD、CSV 等资源
3. **CatalogSource Controller**
   - 管理 CatalogSource Pod 的生命周期
   - 建立 gRPC 连接到目录服务
   - 定期轮询目录更新
   - 维护目录健康状态
## 2. 核心资源详细设计
### 2.1 ClusterServiceVersion (CSV)
#### 2.1.1 数据结构
```go
type ClusterServiceVersion struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ClusterServiceVersionSpec   `json:"spec,omitempty"`
    Status ClusterServiceVersionStatus `json:"status,omitempty"`
}

type ClusterServiceVersionSpec struct {
    // 应用元数据
    DisplayName string `json:"displayName"`
    Version     string `json:"version"` // semver 格式
    Description string `json:"description,omitempty"`
    Keywords    []string `json:"keywords,omitempty"`
    Maintainers []Maintainer `json:"maintainers,omitempty"`
    Provider    AppLink `json:"provider,omitempty"`
    Links       []AppLink `json:"links,omitempty"`
    Icon        []Icon `json:"icon,omitempty"`
    
    // 安装策略
    InstallStrategy InstallStrategy `json:"install"`
    InstallModes    []InstallMode `json:"installModes,omitempty"`
    
    // 自定义资源定义
    CustomResourceDefinitions CustomResourceDefinitions `json:"customresourcedefinitions,omitempty"`
    
    // RBAC 权限
    Permissions []StrategyDeploymentPermissions `json:"permissions,omitempty"`
    
    // 依赖关系
    RelatedImages []RelatedImage `json:"relatedImages,omitempty"`
    
    // 升级配置
    Replaces string `json:"replaces,omitempty"`
    Skips    []string `json:"skips,omitempty"`
    
    // 最小 Kubernetes 版本
    MinKubeVersion string `json:"minKubeVersion,omitempty"`
    
    // 标签和注解
    Labels      map[string]string `json:"labels,omitempty"`
    Annotations map[string]string `json:"annotations,omitempty"`
}

type InstallStrategy struct {
    StrategyName string `json:"strategy"`
    StrategySpec StrategySpec `json:"spec"`
}

type StrategyDeploymentSpec struct {
    DeploymentSpecs []StrategyDeploymentSpec `json:"deployments,omitempty"`
    Permissions     []StrategyDeploymentPermissions `json:"permissions,omitempty"`
}

type StrategyDeploymentSpec struct {
    Name string `json:"name"`
    Spec appsv1.DeploymentSpec `json:"spec"`
}

type CustomResourceDefinitions struct {
    Owned    []CRDDescription `json:"owned,omitempty"`
    Required []CRDDescription `json:"required,omitempty"`
}

type CRDDescription struct {
    Name        string `json:"name"`
    Version     string `json:"version"`
    Kind        string `json:"kind"`
    DisplayName string `json:"displayName,omitempty"`
    Description string `json:"description,omitempty"`
    Resources   []APIResourceReference `json:"resources,omitempty"`
    SpecDescriptors   []SpecDescriptor `json:"specDescriptors,omitempty"`
    StatusDescriptors []StatusDescriptor `json:"statusDescriptors,omitempty"`
}
```
#### 2.1.2 CSV 状态机
```
┌─────────────────────────────────────────────────────────────────┐
│                      CSV 状态机                                  │
│                                                                  │
│                                                                  │
│          ┌────────┐                                             │
│          │  None  │ (初始状态)                                  │
│          └────┬───┘                                             │
│               │ Operator 发现 CSV                               │
│               ▼                                                 │
│          ┌────────┐                                             │
│          │Pending │ 等待依赖满足                                │
│          └────┬───┘                                             │
│               │ 所有依赖已满足                                  │
│               ▼                                                 │
│      ┌──────────────┐                                           │
│      │ InstallReady │ 准备安装                                  │
│      └──────┬───────┘                                           │
│             │ 开始执行安装策略                                  │
│             ▼                                                   │
│      ┌──────────────┐                                           │
│      │ Installing   │ 正在安装                                  │
│      └──────┬───────┘                                           │
│             │                                                   │
│        ┌────┴────┬──────────────┐                               │
│        ▼         ▼              ▼                               │
│   ┌─────────┐ ┌────────┐  ┌──────────┐                         │
│   │Succeeded│ │ Failed │  │Replacing │                         │
│   └─────────┘ └────────┘  └────┬─────┘                         │
│        ▲         ▲              │ 发现新 CSV 替换              │
│        │         │              ▼                               │
│        │         │        ┌──────────┐                         │
│        │         │        │ Deleting │                         │
│        │         │        └──────────┘                         │
│        │         │              │                              │
│        └─────────┴──────────────┘                              │
│               垃圾回收完成                                      │
└─────────────────────────────────────────────────────────────────┘
```
**状态说明**:

| 状态 | 描述 | 触发条件 |
|------|------|---------|
| **None** | 初始状态 | CSV 刚创建 |
| **Pending** | 等待依赖 | CSV 中声明的 CRD 或其他依赖未满足 |
| **InstallReady** | 准备安装 | 所有依赖已满足,可以开始安装 |
| **Installing** | 正在安装 | 正在执行安装策略,创建部署 |
| **Succeeded** | 安装成功 | 所有组件已就绪并正常运行 |
| **Failed** | 安装失败 | 安装策略执行失败或组件丢失 |
| **Replacing** | 待替换 | 发现新 CSV 替换当前版本 |
| **Deleting** | 删除中 | 垃圾回收机制准备删除 CSV |
#### 2.1.3 CSV 控制器实现
```go
func (r *CSVReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    csv := &operatorsv1alpha1.ClusterServiceVersion{}
    if err := r.client.Get(ctx, req.NamespacedName, csv); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 状态机处理
    switch csv.Status.Phase {
    case operatorsv1alpha1.CSVPhaseNone:
        return r.handleNonePhase(ctx, csv)
    case operatorsv1alpha1.CSVPhasePending:
        return r.handlePendingPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseInstallReady:
        return r.handleInstallReadyPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseInstalling:
        return r.handleInstallingPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseSucceeded:
        return r.handleSucceededPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseFailed:
        return r.handleFailedPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseReplacing:
        return r.handleReplacingPhase(ctx, csv)
    case operatorsv1alpha1.CSVPhaseDeleting:
        return r.handleDeletingPhase(ctx, csv)
    default:
        return ctrl.Result{}, nil
    }
}

func (r *CSVReconciler) handlePendingPhase(ctx context.Context, csv *operatorsv1alpha1.ClusterServiceVersion) (ctrl.Result, error) {
    // 检查所有依赖是否满足
    requirementsMet, missingRequirements := r.checkRequirements(ctx, csv)
    
    if !requirementsMet {
        // 更新状态,记录缺失的依赖
        csv.Status.RequirementStatus = missingRequirements
        csv.Status.Message = "Requirements not met"
        return ctrl.Result{RequeueAfter: 5 * time.Second}, r.client.Status().Update(ctx, csv)
    }

    // 所有依赖已满足,转换到 InstallReady 状态
    csv.Status.Phase = operatorsv1alpha1.CSVPhaseInstallReady
    csv.Status.Message = "InstallReady"
    csv.Status.Reason = operatorsv1alpha1.CSVReasonInstallReady
    return ctrl.Result{}, r.client.Status().Update(ctx, csv)
}

func (r *CSVReconciler) checkRequirements(ctx context.Context, csv *operatorsv1alpha1.ClusterServiceVersion) (bool, []RequirementStatus) {
    var missingRequirements []RequirementStatus
    allMet := true

    // 检查必需的 CRD
    for _, requiredCRD := range csv.Spec.CustomResourceDefinitions.Required {
        crd := &apiextensionsv1.CustomResourceDefinition{}
        err := r.client.Get(ctx, client.ObjectKey{Name: requiredCRD.Name}, crd)
        if err != nil {
            allMet = false
            missingRequirements = append(missingRequirements, RequirementStatus{
                Name:    requiredCRD.Name,
                Status:  "NotPresent",
                Message: "Required CRD not found",
            })
        }
    }

    // 检查自有 CRD
    for _, ownedCRD := range csv.Spec.CustomResourceDefinitions.Owned {
        crd := &apiextensionsv1.CustomResourceDefinition{}
        err := r.client.Get(ctx, client.ObjectKey{Name: ownedCRD.Name}, crd)
        if err != nil {
            // 自有 CRD 不存在是正常的,将由 InstallPlan 创建
            continue
        }
        
        // 检查 CRD 是否被其他 CSV 管理
        if crdOwner := r.getCRDOwner(crd); crdOwner != "" && crdOwner != csv.Name {
            allMet = false
            missingRequirements = append(missingRequirements, RequirementStatus{
                Name:    ownedCRD.Name,
                Status:  "Conflict",
                Message: fmt.Sprintf("CRD owned by another CSV: %s", crdOwner),
            })
        }
    }

    return allMet, missingRequirements
}
```
### 2.2 CatalogSource
#### 2.2.1 数据结构
```go
type CatalogSource struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CatalogSourceSpec   `json:"spec,omitempty"`
    Status CatalogSourceStatus `json:"status,omitempty"`
}

type CatalogSourceSpec struct {
    // 目录源类型
    SourceType CatalogSourceType `json:"sourceType"`
    
    // gRPC 目录源配置
    Image string `json:"image,omitempty"` // Index Image
    Address string `json:"address,omitempty"` // gRPC 服务地址
    
    // 内联目录源配置
    ConfigMap string `json:"configmap,omitempty"`
    
    // 显示信息
    DisplayName string `json:"displayName,omitempty"`
    Publisher   string `json:"publisher,omitempty"`
    
    // 优先级
    Priority int `json:"priority,omitempty"`
    
    // 更新策略
    UpdateStrategy UpdateStrategy `json:"updateStrategy,omitempty"`
    
    // gRPC Pod 配置
    GrpcPodConfig GrpcPodConfig `json:"grpcPodConfig,omitempty"`
}

type UpdateStrategy struct {
    RegistryPoll *RegistryPoll `json:"registryPoll,omitempty"`
}

type RegistryPoll struct {
    Interval metav1.Duration `json:"interval,omitempty"`
}

type GrpcPodConfig struct {
    NodeSelector       map[string]string `json:"nodeSelector,omitempty"`
    Tolerations        []corev1.Toleration `json:"tolerations,omitempty"`
    PriorityClassName  string `json:"priorityClassName,omitempty"`
    SecurityContextConfig string `json:"securityContextConfig,omitempty"`
}

type CatalogSourceStatus struct {
    // 连接状态
    ConnectionState ConnectionState `json:"connectionState,omitempty"`
    
    // 最新镜像轮询时间
    LatestImageRegistryPoll metav1.Time `json:"latestImageRegistryPoll,omitempty"`
    
    // 注册服务信息
    RegistryService RegistryServiceStatus `json:"registryService,omitempty"`
}

type ConnectionState struct {
    Address           string      `json:"address,omitempty"`
    LastConnect       metav1.Time `json:"lastConnect,omitempty"`
    LastObservedState string      `json:"lastObservedState,omitempty"`
}
```
#### 2.2.2 CatalogSource 工作流程
```
┌─────────────────────────────────────────────────────────────────┐
│              CatalogSource 生命周期                              │
│                                                                  │
│  1. 创建 CatalogSource CR                                       │
│     └─> Catalog Operator 监听到创建事件                         │
│                                                                  │
│  2. 创建 gRPC Pod (基于 Index Image)                            │
│     ├─> 创建 Service (ClusterIP)                                │
│     ├─> 创建 Deployment                                         │
│     └─> 等待 Pod Ready                                          │
│                                                                  │
│  3. 建立 gRPC 连接                                              │
│     ├─> 连接到 Service 地址                                     │
│     ├─> 验证连接状态                                            │
│     └─> 更新 ConnectionState                                    │
│                                                                  │
│  4. 定期轮询更新                                                │
│     ├─> 检查 Image Digest 变化                                  │
│     ├─> 如有变化,重启 Pod                                       │
│     └─> 更新 LatestImageRegistryPoll                            │
│                                                                  │
│  5. 提供 Operator Registry API                                  │
│     ├─> ListPackages                                            │
│     ├─> GetPackageManifest                                      │
│     ├─> GetClusterServiceVersion                                │
│     └─> GetBundleForChannel                                     │
└─────────────────────────────────────────────────────────────────┘
```
### 2.3 Subscription
#### 2.3.1 数据结构
```go
type Subscription struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   SubscriptionSpec   `json:"spec,omitempty"`
    Status SubscriptionStatus `json:"status,omitempty"`
}

type SubscriptionSpec struct {
    // 目录源信息
    Source       string `json:"source"`        // CatalogSource 名称
    SourceNamespace string `json:"sourceNamespace"` // CatalogSource 命名空间
    
    // 软件包和频道
    Package string `json:"name"`    // 软件包名称
    Channel string `json:"channel"` // 频道名称
    
    // 起始 CSV
    StartingCSV string `json:"startingCSV,omitempty"`
    
    // 批准模式
    InstallPlanApproval Approval `json:"installPlanApproval,omitempty"`
    
    // 配置
    Config SubscriptionConfig `json:"config,omitempty"`
}

type SubscriptionStatus struct {
    // 当前状态
    CurrentCSV string `json:"currentCSV,omitempty"`
    InstalledCSV string `json:"installedCSV,omitempty"`
    
    // InstallPlan 引用
    InstallPlanRef *corev1.ObjectReference `json:"installPlanRef,omitempty"`
    
    // 状态
    State SubscriptionState `json:"state,omitempty"`
    
    // 条件
    Conditions []SubscriptionCondition `json:"conditions,omitempty"`
    
    // 目录健康
    CatalogHealth []SubscriptionCatalogHealth `json:"catalogHealth,omitempty"`
}

type SubscriptionState string

const (
    SubscriptionStateNone            SubscriptionState = "None"
    SubscriptionStateUpgradePending  SubscriptionState = "UpgradePending"
    SubscriptionStateAtLatest        SubscriptionState = "AtLatestKnown"
)
```
#### 2.3.2 Subscription 状态机
```
┌─────────────────────────────────────────────────────────────────┐
│                  Subscription 状态机                             │
│                                                                  │
│                                                                  │
│          ┌────────┐                                             │
│          │  None  │ 初始状态                                    │
│          └────┬───┘                                             │
│               │ 创建 Subscription                               │
│               ▼                                                 │
│      ┌──────────────────┐                                       │
│      │ UpgradeAvailable │ 发现可用更新                          │
│      └────────┬─────────┘                                       │
│               │ 创建 InstallPlan                                │
│               ▼                                                 │
│      ┌──────────────────┐                                       │
│      │  UpgradePending  │ 等待 InstallPlan 完成                 │
│      └────────┬─────────┘                                       │
│               │ InstallPlan 完成                                │
│               ▼                                                 │
│      ┌──────────────────┐                                       │
│      │  AtLatestKnown   │ 已安装最新版本                        │
│      └────────┬─────────┘                                       │
│               │ 发现新版本                                      │
│               │                                                 │
│               └──────────┐                                      │
│                          ▼                                      │
│                  ┌──────────────────┐                           │
│                  │ UpgradeAvailable │ (循环)                    │
│                  └──────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```
#### 2.3.3 Subscription Controller 实现

```go
func (r *SubscriptionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    sub := &operatorsv1alpha1.Subscription{}
    if err := r.client.Get(ctx, req.NamespacedName, sub); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 获取 CatalogSource
    catalogSource := &operatorsv1alpha1.CatalogSource{}
    err := r.client.Get(ctx, client.ObjectKey{
        Name:      sub.Spec.Source,
        Namespace: sub.Spec.SourceNamespace,
    }, catalogSource)
    if err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to get catalog source: %v", err)
    }

    // 查询目录获取最新 CSV
    latestCSV, err := r.getLatestCSVFromCatalog(ctx, catalogSource, sub.Spec.Package, sub.Spec.Channel)
    if err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to get latest CSV: %v", err)
    }

    // 检查是否需要更新
    if sub.Status.InstalledCSV == "" {
        // 首次安装
        return r.createInstallPlan(ctx, sub, latestCSV)
    }

    if latestCSV != sub.Status.InstalledCSV {
        // 发现新版本
        sub.Status.State = operatorsv1alpha1.SubscriptionStateUpgradeAvailable
        sub.Status.CurrentCSV = latestCSV
        return ctrl.Result{}, r.createInstallPlan(ctx, sub, latestCSV)
    }

    // 已是最新版本
    sub.Status.State = operatorsv1alpha1.SubscriptionStateAtLatestKnown
    return ctrl.Result{RequeueAfter: r.pollInterval}, r.client.Status().Update(ctx, sub)
}

func (r *SubscriptionReconciler) getLatestCSVFromCatalog(
    ctx context.Context,
    catalogSource *operatorsv1alpha1.CatalogSource,
    packageName string,
    channelName string,
) (string, error) {
    // 创建 gRPC 客户端
    conn, err := grpc.Dial(
        catalogSource.Status.ConnectionState.Address,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return "", err
    }
    defer conn.Close()

    client := api.NewRegistryClient(conn)

    // 获取 Package Manifest
    req := &api.GetPackageManifestRequest{
        Name: packageName,
    }
    resp, err := client.GetPackageManifest(ctx, req)
    if err != nil {
        return "", err
    }

    // 查找指定频道
    for _, channel := range resp.GetChannels() {
        if channel.GetName() == channelName {
            return channel.GetCurrentCSVName(), nil
        }
    }

    return "", fmt.Errorf("channel %s not found in package %s", channelName, packageName)
}

func (r *SubscriptionReconciler) createInstallPlan(
    ctx context.Context,
    sub *operatorsv1alpha1.Subscription,
    csvName string,
) (ctrl.Result, error) {
    installPlan := &operatorsv1alpha1.InstallPlan{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-%s", sub.Name, uuid.New().String()[:8]),
            Namespace: sub.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: operatorsv1alpha1.GroupVersion.String(),
                    Kind:       "Subscription",
                    Name:       sub.Name,
                    UID:        sub.UID,
                },
            },
        },
        Spec: operatorsv1alpha1.InstallPlanSpec{
            ClusterServiceVersionNames: []string{csvName},
            Approval:                   sub.Spec.InstallPlanApproval,
            Approved:                   sub.Spec.InstallPlanApproval == operatorsv1alpha1.ApprovalAutomatic,
        },
    }

    if err := r.client.Create(ctx, installPlan); err != nil {
        return ctrl.Result{}, err
    }

    sub.Status.InstallPlanRef = &corev1.ObjectReference{
        APIVersion: operatorsv1alpha1.GroupVersion.String(),
        Kind:       "InstallPlan",
        Name:       installPlan.Name,
        Namespace:  installPlan.Namespace,
    }
    sub.Status.State = operatorsv1alpha1.SubscriptionStateUpgradePending

    return ctrl.Result{}, r.client.Status().Update(ctx, sub)
}
```
### 2.4 InstallPlan
#### 2.4.1 数据结构
```go
type InstallPlan struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   InstallPlanSpec   `json:"spec,omitempty"`
    Status InstallPlanStatus `json:"status,omitempty"`
}

type InstallPlanSpec struct {
    // 要安装的 CSV 列表
    ClusterServiceVersionNames []string `json:"clusterServiceVersionNames"`
    
    // 批准模式
    Approval Approval `json:"approval"`
    
    // 是否已批准
    Approved bool `json:"approved"`
}

type InstallPlanStatus struct {
    // 计划状态
    Phase InstallPlanPhase `json:"phase,omitempty"`
    
    // 计划详情
    Plan []Step `json:"plan,omitempty"`
    
    // 目录来源
    CatalogSources []CatalogSourceRef `json:"catalogSources,omitempty"`
    
    // 条件
    Conditions []InstallPlanCondition `json:"conditions,omitempty"`
}

type InstallPlanPhase string

const (
    InstallPlanPhaseNone             InstallPlanPhase = "None"
    InstallPlanPhasePlanning         InstallPlanPhase = "Planning"
    InstallPlanPhaseRequiresApproval InstallPlanPhase = "RequiresApproval"
    InstallPlanPhaseInstalling       InstallPlanPhase = "Installing"
    InstallPlanPhaseComplete         InstallPlanPhase = "Complete"
    InstallPlanPhaseFailed           InstallPlanPhase = "Failed"
)

type Step struct {
    Resolving string `json:"resolving,omitempty"`
    Resource  StepResource `json:"resource"`
    Status    StepStatus `json:"status"`
}

type StepResource struct {
    Name     string `json:"name"`
    Kind     string `json:"kind"`
    Group    string `json:"group"`
    Version  string `json:"version"`
    Manifest string `json:"manifest"`
}

type StepStatus string

const (
    StepStatusUnknown    StepStatus = "Unknown"
    StepStatusNotPresent StepStatus = "NotPresent"
    StepStatusPresent    StepStatus = "Present"
    StepStatusCreated    StepStatus = "Created"
)
```
#### 2.4.2 InstallPlan 状态机
```
┌─────────────────────────────────────────────────────────────────┐
│                  InstallPlan 状态机                              │
│                                                                  │
│          ┌────────┐                                             │
│          │  None  │ 初始状态                                    │
│          └────┬───┘                                             │
│               │ 开始规划                                        │
│               ▼                                                 │
│      ┌──────────────┐                                           │
│      │   Planning   │ 解析依赖,生成计划                         │
│      └──────┬───────┘                                           │
│             │                                                   │
│        ┌────┴────┐                                              │
│        ▼         ▼                                              │
│   ┌─────────┐ ┌──────────────────┐                              │
│   │ Failed  │ │ RequiresApproval │ 需要手动批准                 │
│   └─────────┘ └────────┬─────────┘                              │
│                        │ 用户批准                                │
│                        ▼                                        │
│                ┌──────────────┐                                  │
│                │  Installing  │ 创建资源                        │
│                └──────┬───────┘                                  │
│                       │                                         │
│                  ┌────┴────┐                                    │
│                  ▼         ▼                                    │
│           ┌──────────┐ ┌────────┐                               │
│           │ Complete │ │ Failed │                               │
│           └──────────┘ └────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```
#### 2.4.3 依赖解析实现
```go
func (r *InstallPlanReconciler) resolveDependencies(
    ctx context.Context,
    installPlan *operatorsv1alpha1.InstallPlan,
) ([]Step, error) {
    var steps []Step
    visited := make(map[string]bool)

    // 递归解析依赖
    for _, csvName := range installPlan.Spec.ClusterServiceVersionNames {
        csvSteps, err := r.resolveCSVDependencies(ctx, csvName, visited)
        if err != nil {
            return nil, err
        }
        steps = append(steps, csvSteps...)
    }

    return steps, nil
}

func (r *InstallPlanReconciler) resolveCSVDependencies(
    ctx context.Context,
    csvName string,
    visited map[string]bool,
) ([]Step, error) {
    if visited[csvName] {
        return nil, nil // 避免循环依赖
    }
    visited[csvName] = true

    var steps []Step

    // 从目录获取 CSV
    csv, err := r.getCSVFromCatalog(ctx, csvName)
    if err != nil {
        return nil, fmt.Errorf("failed to get CSV %s: %v", csvName, err)
    }

    // 解析必需的 CRD
    for _, requiredCRD := range csv.Spec.CustomResourceDefinitions.Required {
        // 检查 CRD 是否已存在
        crd := &apiextensionsv1.CustomResourceDefinition{}
        err := r.client.Get(ctx, client.ObjectKey{Name: requiredCRD.Name}, crd)
        if err != nil && errors.IsNotFound(err) {
            // CRD 不存在,需要找到提供该 CRD 的 CSV
            providingCSV, err := r.findCSVProvidingCRD(ctx, requiredCRD.Name)
            if err != nil {
                return nil, fmt.Errorf("no CSV provides required CRD %s: %v", requiredCRD.Name, err)
            }

            // 递归解析依赖
            depSteps, err := r.resolveCSVDependencies(ctx, providingCSV, visited)
            if err != nil {
                return nil, err
            }
            steps = append(steps, depSteps...)
        }
    }

    // 添加 CRD 创建步骤
    for _, ownedCRD := range csv.Spec.CustomResourceDefinitions.Owned {
        crdManifest, err := r.generateCRDManifest(ownedCRD)
        if err != nil {
            return nil, err
        }

        steps = append(steps, Step{
            Resolving: csvName,
            Resource: StepResource{
                Name:     ownedCRD.Name,
                Kind:     "CustomResourceDefinition",
                Group:    "apiextensions.k8s.io",
                Version:  "v1",
                Manifest: crdManifest,
            },
            Status: StepStatusNotPresent,
        })
    }

    // 添加 CSV 创建步骤
    csvManifest, err := json.Marshal(csv)
    if err != nil {
        return nil, err
    }

    steps = append(steps, Step{
        Resolving: csvName,
        Resource: StepResource{
            Name:     csv.Name,
            Kind:     "ClusterServiceVersion",
            Group:    operatorsv1alpha1.GroupVersion.Group,
            Version:  operatorsv1alpha1.GroupVersion.Version,
            Manifest: string(csvManifest),
        },
        Status: StepStatusNotPresent,
    })

    return steps, nil
}

func (r *InstallPlanReconciler) executeInstallPlan(
    ctx context.Context,
    installPlan *operatorsv1alpha1.InstallPlan,
) error {
    for i, step := range installPlan.Status.Plan {
        if step.Status == StepStatusPresent || step.Status == StepStatusCreated {
            continue
        }

        // 解析资源清单
        obj := &unstructured.Unstructured{}
        if err := yaml.Unmarshal([]byte(step.Resource.Manifest), obj); err != nil {
            return fmt.Errorf("failed to unmarshal manifest: %v", err)
        }

        // 创建资源
        if err := r.client.Create(ctx, obj); err != nil {
            if errors.IsAlreadyExists(err) {
                installPlan.Status.Plan[i].Status = StepStatusPresent
            } else {
                return fmt.Errorf("failed to create resource %s: %v", step.Resource.Name, err)
            }
        } else {
            installPlan.Status.Plan[i].Status = StepStatusCreated
        }
    }

    // 检查是否所有步骤都完成
    allCompleted := true
    for _, step := range installPlan.Status.Plan {
        if step.Status != StepStatusPresent && step.Status != StepStatusCreated {
            allCompleted = false
            break
        }
    }

    if allCompleted {
        installPlan.Status.Phase = InstallPlanPhaseComplete
    }

    return r.client.Status().Update(ctx, installPlan)
}
```
### 2.5 OperatorGroup
#### 2.5.1 数据结构
```go
type OperatorGroup struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   OperatorGroupSpec   `json:"spec,omitempty"`
    Status OperatorGroupStatus `json:"status,omitempty"`
}

type OperatorGroupSpec struct {
    // 目标命名空间
    TargetNamespaces []string `json:"targetNamespaces,omitempty"`
    
    // 命名空间选择器
    Selector metav1.LabelSelector `json:"selector,omitempty"`
    
    // 服务账户名称
    ServiceAccountName string `json:"serviceAccountName,omitempty"`
    
    // 静态提供者
    StaticProvidedAPIs []string `json:"staticProvidedAPIs,omitempty"`
    StaticRequiredAPIs []string `json:"staticRequiredAPIs,omitempty"`
}

type OperatorGroupStatus struct {
    // 命名空间列表
    Namespaces []string `json:"namespaces,omitempty"`
    
    // 服务账户名称
    ServiceAccountRef *corev1.ObjectReference `json:"serviceAccountRef,omitempty"`
    
    // 提供的 API
    ProvidedAPIs []GroupVersionKind `json:"providedAPIs,omitempty"`
}
```
#### 2.5.2 多租户隔离实现
```go
func (r *OperatorGroupReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    og := &operatorsv1.OperatorGroup{}
    if err := r.client.Get(ctx, req.NamespacedName, og); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 解析目标命名空间
    targetNamespaces, err := r.resolveTargetNamespaces(ctx, og)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 创建或更新服务账户
    if err := r.ensureServiceAccount(ctx, og); err != nil {
        return ctrl.Result{}, err
    }

    // 创建或更新 RBAC 规则
    if err := r.ensureRBAC(ctx, og, targetNamespaces); err != nil {
        return ctrl.Result{}, err
    }

    // 注入目标命名空间到 CSV
    if err := r.injectTargetNamespaces(ctx, og, targetNamespaces); err != nil {
        return ctrl.Result{}, err
    }

    // 更新状态
    og.Status.Namespaces = targetNamespaces
    return ctrl.Result{}, r.client.Status().Update(ctx, og)
}

func (r *OperatorGroupReconciler) resolveTargetNamespaces(
    ctx context.Context,
    og *operatorsv1.OperatorGroup,
) ([]string, error) {
    // 如果指定了目标命名空间,直接使用
    if len(og.Spec.TargetNamespaces) > 0 {
        return og.Spec.TargetNamespaces, nil
    }

    // 使用标签选择器查找命名空间
    namespaceList := &corev1.NamespaceList{}
    selector, err := metav1.LabelSelectorAsSelector(&og.Spec.Selector)
    if err != nil {
        return nil, err
    }

    if err := r.client.List(ctx, namespaceList, client.MatchingLabelsSelector{Selector: selector}); err != nil {
        return nil, err
    }

    var namespaces []string
    for _, ns := range namespaceList.Items {
        namespaces = append(namespaces, ns.Name)
    }

    return namespaces, nil
}

func (r *OperatorGroupReconciler) ensureRBAC(
    ctx context.Context,
    og *operatorsv1.OperatorGroup,
    targetNamespaces []string,
) error {
    // 获取 OperatorGroup 中的所有 CSV
    csvList := &operatorsv1alpha1.ClusterServiceVersionList{}
    if err := r.client.List(ctx, csvList, client.InNamespace(og.Namespace)); err != nil {
        return err
    }

    for _, csv := range csvList.Items {
        // 为每个 CSV 创建 RBAC 规则
        for _, permission := range csv.Spec.Permissions {
            // 创建 Role 或 ClusterRole
            if len(targetNamespaces) == 1 && targetNamespaces[0] == og.Namespace {
                // 单命名空间模式: 创建 Role
                role := &rbacv1.Role{
                    ObjectMeta: metav1.ObjectMeta{
                        Name:      fmt.Sprintf("%s-%s", csv.Name, permission.ServiceAccountName),
                        Namespace: og.Namespace,
                    },
                    Rules: permission.Rules,
                }
                if err := r.client.Create(ctx, role); err != nil && !errors.IsAlreadyExists(err) {
                    return err
                }

                // 创建 RoleBinding
                roleBinding := &rbacv1.RoleBinding{
                    ObjectMeta: metav1.ObjectMeta{
                        Name:      fmt.Sprintf("%s-%s", csv.Name, permission.ServiceAccountName),
                        Namespace: og.Namespace,
                    },
                    Subjects: []rbacv1.Subject{
                        {
                            Kind:      "ServiceAccount",
                            Name:      permission.ServiceAccountName,
                            Namespace: og.Namespace,
                        },
                    },
                    RoleRef: rbacv1.RoleRef{
                        Kind: "Role",
                        Name: role.Name,
                    },
                }
                if err := r.client.Create(ctx, roleBinding); err != nil && !errors.IsAlreadyExists(err) {
                    return err
                }
            } else {
                // 多命名空间模式: 创建 ClusterRole
                clusterRole := &rbacv1.ClusterRole{
                    ObjectMeta: metav1.ObjectMeta{
                        Name: fmt.Sprintf("%s-%s", csv.Name, permission.ServiceAccountName),
                    },
                    Rules: permission.Rules,
                }
                if err := r.client.Create(ctx, clusterRole); err != nil && !errors.IsAlreadyExists(err) {
                    return err
                }

                // 为每个目标命名空间创建 RoleBinding
                for _, ns := range targetNamespaces {
                    roleBinding := &rbacv1.RoleBinding{
                        ObjectMeta: metav1.ObjectMeta{
                            Name:      fmt.Sprintf("%s-%s", csv.Name, permission.ServiceAccountName),
                            Namespace: ns,
                        },
                        Subjects: []rbacv1.Subject{
                            {
                                Kind:      "ServiceAccount",
                                Name:      permission.ServiceAccountName,
                                Namespace: og.Namespace,
                            },
                        },
                        RoleRef: rbacv1.RoleRef{
                            Kind: "ClusterRole",
                            Name: clusterRole.Name,
                        },
                    }
                    if err := r.client.Create(ctx, roleBinding); err != nil && !errors.IsAlreadyExists(err) {
                        return err
                    }
                }
            }
        }
    }

    return nil
}
```
## 3. 升级路径实现
### 3.1 版本更新图
```
┌─────────────────────────────────────────────────────────────────┐
│                    CatalogSource 中的版本图                      │
│                                                                  │
│  Package: etcd                                                   │
│                                                                  │
│  Channel: alpha                                                  │
│    ┌──────────────┐                                             │
│    │ v0.9.4 (Head)│ <── 频道头                                  │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v0.9.3     │                                             │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v0.9.2     │                                             │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v0.9.1     │ (skipRange: >=0.9.0 <0.9.2)                │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v0.9.0     │                                             │
│    └──────────────┘                                             │
│                                                                  │
│  Channel: stable                                                 │
│    ┌──────────────┐                                             │
│    │ v1.0.2 (Head)│                                             │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v1.0.1     │                                             │
│    └──────┬───────┘                                             │
│           │ replaces                                            │
│    ┌──────▼───────┐                                             │
│    │   v1.0.0     │                                             │
│    └──────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
```
### 3.2 升级路径解析
```go
func (r *SubscriptionReconciler) findUpdatePath(
    ctx context.Context,
    catalogSource *operatorsv1alpha1.CatalogSource,
    packageName string,
    channelName string,
    currentCSV string,
) ([]string, error) {
    // 连接到目录服务
    conn, err := grpc.Dial(
        catalogSource.Status.ConnectionState.Address,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }
    defer conn.Close()

    client := api.NewRegistryClient(conn)

    // 获取频道头
    req := &api.GetPackageManifestRequest{Name: packageName}
    resp, err := client.GetPackageManifest(ctx, req)
    if err != nil {
        return nil, err
    }

    var channelHead string
    for _, channel := range resp.GetChannels() {
        if channel.GetName() == channelName {
            channelHead = channel.GetCurrentCSVName()
            break
        }
    }

    if channelHead == "" {
        return nil, fmt.Errorf("channel %s not found", channelName)
    }

    // 从频道头回溯到当前版本
    var upgradePath []string
    current := channelHead

    for current != "" && current != currentCSV {
        upgradePath = append([]string{current}, upgradePath...)

        // 获取当前 CSV
        csvReq := &api.GetBundleRequest{PkgName: packageName, ChannelName: channelName, CsvName: current}
        csvResp, err := client.GetBundle(ctx, csvReq)
        if err != nil {
            return nil, err
        }

        // 解析 CSV 获取 replaces 字段
        csv := &operatorsv1alpha1.ClusterServiceVersion{}
        if err := yaml.Unmarshal([]byte(csvResp.GetCsvJson()), csv); err != nil {
            return nil, err
        }

        // 检查 skipRange
        if skipRange, ok := csv.Annotations["olm.skipRange"]; ok {
            // 检查当前版本是否在跳过范围内
            if r.isVersionInRange(currentCSV, skipRange) {
                // 可以直接跳到当前版本
                return []string{channelHead}, nil
            }
        }

        // 检查 skips
        if len(csv.Spec.Skips) > 0 {
            for _, skip := range csv.Spec.Skips {
                if skip == currentCSV {
                    // 跳过中间版本
                    return []string{current}, nil
                }
            }
        }

        current = csv.Spec.Replaces
    }

    if current == "" {
        return nil, fmt.Errorf("no upgrade path found from %s to %s", currentCSV, channelHead)
    }

    return upgradePath, nil
}

func (r *SubscriptionReconciler) isVersionInRange(version, skipRange string) bool {
    // 解析 semver 范围
    rangeRegex := regexp.MustCompile(`>=([^,]+)\s*<([^,]+)`)
    matches := rangeRegex.FindStringSubmatch(skipRange)
    if len(matches) != 3 {
        return false
    }

    minVersion := strings.TrimSpace(matches[1])
    maxVersion := strings.TrimSpace(matches[2])

    // 比较版本
    v := semver.MustParse(version)
    minV := semver.MustParse(minVersion)
    maxV := semver.MustParse(maxVersion)

    return v.GreaterThanOrEqual(minV) && v.LessThan(maxV)
}
```
## 4. 安全机制
### 4.1 RBAC 权限控制
```go
func (r *CSVReconciler) validatePermissions(
    ctx context.Context,
    csv *operatorsv1alpha1.ClusterServiceVersion,
) error {
    // 检查 OperatorGroup
    ogList := &operatorsv1.OperatorGroupList{}
    if err := r.client.List(ctx, ogList, client.InNamespace(csv.Namespace)); err != nil {
        return err
    }

    if len(ogList.Items) == 0 {
        return fmt.Errorf("no OperatorGroup found in namespace %s", csv.Namespace)
    }

    og := ogList.Items[0]

    // 验证权限范围
    for _, permission := range csv.Spec.Permissions {
        // 检查服务账户是否存在
        sa := &corev1.ServiceAccount{}
        err := r.client.Get(ctx, client.ObjectKey{
            Name:      permission.ServiceAccountName,
            Namespace: csv.Namespace,
        }, sa)
        if err != nil {
            return fmt.Errorf("service account %s not found", permission.ServiceAccountName)
        }

        // 验证权限是否在 OperatorGroup 允许的范围内
        if err := r.validatePermissionScope(permission, og); err != nil {
            return err
        }
    }

    return nil
}

func (r *CSVReconciler) validatePermissionScope(
    permission operatorsv1alpha1.StrategyDeploymentPermissions,
    og *operatorsv1.OperatorGroup,
) error {
    for _, rule := range permission.Rules {
        // 检查是否有权限访问目标命名空间之外的资源
        for _, resource := range rule.Resources {
            if !r.isResourceAllowedInNamespaces(resource, og.Status.Namespaces) {
                return fmt.Errorf("permission for resource %s is not allowed in operator group scope", resource)
            }
        }
    }
    return nil
}
```
### 4.2 CRD 所有权验证
```go
func (r *CSVReconciler) validateCRDOwnership(
    ctx context.Context,
    csv *operatorsv1alpha1.ClusterServiceVersion,
) error {
    for _, ownedCRD := range csv.Spec.CustomResourceDefinitions.Owned {
        crd := &apiextensionsv1.CustomResourceDefinition{}
        err := r.client.Get(ctx, client.ObjectKey{Name: ownedCRD.Name}, crd)
        if err != nil {
            if errors.IsNotFound(err) {
                // CRD 不存在,可以创建
                continue
            }
            return err
        }

        // 检查 CRD 是否被其他 CSV 管理
        ownerAnnotation := crd.Annotations["olm.owner"]
        if ownerAnnotation != "" && ownerAnnotation != csv.Name {
            return fmt.Errorf("CRD %s is already owned by CSV %s", ownedCRD.Name, ownerAnnotation)
        }
    }

    return nil
}
```
## 5. 性能优化
### 5.1 缓存机制
```go
type CatalogCache struct {
    client    api.RegistryClient
    cache     map[string]*cachedPackage
    mutex     sync.RWMutex
    ttl       time.Duration
}

type cachedPackage struct {
    manifest  *api.PackageManifest
    timestamp time.Time
}

func (c *CatalogCache) GetPackageManifest(ctx context.Context, packageName string) (*api.PackageManifest, error) {
    c.mutex.RLock()
    if cached, ok := c.cache[packageName]; ok {
        if time.Since(cached.timestamp) < c.ttl {
            c.mutex.RUnlock()
            return cached.manifest, nil
        }
    }
    c.mutex.RUnlock()

    // 从目录获取
    resp, err := c.client.GetPackageManifest(ctx, &api.GetPackageManifestRequest{Name: packageName})
    if err != nil {
        return nil, err
    }

    // 更新缓存
    c.mutex.Lock()
    c.cache[packageName] = &cachedPackage{
        manifest:  resp,
        timestamp: time.Now(),
    }
    c.mutex.Unlock()

    return resp, nil
}
```
### 5.2 批量处理
```go
func (r *SubscriptionReconciler) batchReconcile(ctx context.Context, subscriptions []operatorsv1alpha1.Subscription) error {
    // 批量查询目录
    catalogQueries := make(map[string][]string)
    for _, sub := range subscriptions {
        key := fmt.Sprintf("%s/%s", sub.Spec.SourceNamespace, sub.Spec.Source)
        catalogQueries[key] = append(catalogQueries[key], sub.Spec.Package)
    }

    // 批量获取包清单
    packageManifests := make(map[string]*api.PackageManifest)
    for catalogKey, packages := range catalogQueries {
        parts := strings.Split(catalogKey, "/")
        catalogSource, err := r.getCatalogSource(ctx, parts[0], parts[1])
        if err != nil {
            continue
        }

        for _, pkg := range packages {
            manifest, err := r.getPackageManifest(ctx, catalogSource, pkg)
            if err == nil {
                packageManifests[fmt.Sprintf("%s/%s", catalogKey, pkg)] = manifest
            }
        }
    }

    // 批量处理订阅
    for _, sub := range subscriptions {
        key := fmt.Sprintf("%s/%s/%s", sub.Spec.SourceNamespace, sub.Spec.Source, sub.Spec.Package)
        if manifest, ok := packageManifests[key]; ok {
            r.processSubscription(ctx, &sub, manifest)
        }
    }

    return nil
}
```
## 6. 监控与可观测性
### 6.1 指标收集
```go
var (
    csvInstallCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "olm_csv_install_total",
            Help: "Total number of CSV installations",
        },
        []string{"namespace", "name", "phase"},
    )

    subscriptionUpdateCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "olm_subscription_update_total",
            Help: "Total number of subscription updates",
        },
        []string{"namespace", "name", "package", "channel"},
    )

    installPlanDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "olm_install_plan_duration_seconds",
            Help:    "Duration of install plan execution",
            Buckets: prometheus.DefBuckets,
        },
        []string{"namespace", "approval"},
    )
)

func (r *CSVReconciler) recordMetrics(csv *operatorsv1alpha1.ClusterServiceVersion) {
    csvInstallCount.WithLabelValues(
        csv.Namespace,
        csv.Name,
        string(csv.Status.Phase),
    ).Inc()
}

func (r *InstallPlanReconciler) recordDuration(start time.Time, installPlan *operatorsv1alpha1.InstallPlan) {
    duration := time.Since(start).Seconds()
    installPlanDuration.WithLabelValues(
        installPlan.Namespace,
        string(installPlan.Spec.Approval),
    ).Observe(duration)
}
```
### 6.2 事件记录
```go
func (r *CSVReconciler) recordEvent(csv *operatorsv1alpha1.ClusterServiceVersion, eventType, reason, message string) {
    r.recorder.Eventf(csv, eventType, reason, message)
}

// 使用示例
r.recordEvent(csv, corev1.EventTypeNormal, "InstallSucceeded", 
    fmt.Sprintf("CSV %s installed successfully", csv.Name))

r.recordEvent(csv, corev1.EventTypeWarning, "InstallFailed", 
    fmt.Sprintf("Failed to install CSV %s: %v", csv.Name, err))
```
## 7. 总结
OpenShift Operator Lifecycle Manager (OLM) 通过以下核心设计实现了 Operator 的全生命周期管理:

1. **声明式管理**: 使用 Kubernetes 原生 CRD 管理 Operator 生命周期
2. **状态机驱动**: 通过状态机确保操作的可预测性和可恢复性
3. **依赖解析**: 自动解析和安装 Operator 所需的依赖
4. **版本控制**: 通过频道和更新图管理 Operator 版本升级
5. **多租户隔离**: 通过 OperatorGroup 实现命名空间级别的隔离
6. **安全机制**: RBAC 权限控制和 CRD 所有权验证
7. **性能优化**: 缓存机制和批量处理提高效率
8. **可观测性**: 指标收集和事件记录支持监控和故障排查

这些设计使 OLM 成为 Kubernetes 生态系统中管理 Operator 的核心组件,为用户提供了可靠、安全、高效的 Operator 管理体验。

      
# 开发被 OLM 管理的 Operator 全面教程
## 目录
1. [环境准备](#1-环境准备)
2. [创建 Operator 项目](#2-创建-operator-项目)
3. [定义 CRD](#3-定义-crd)
4. [实现 Controller 逻辑](#4-实现-controller-逻辑)
5. [生成 OLM 元数据](#5-生成-olm-元数据)
6. [本地测试](#6-本地测试)
7. [打包和发布](#7-打包和发布)
8. [通过 OLM 安装](#8-通过-olm-安装)
9. [高级功能](#9-高级功能)
10. [最佳实践](#10-最佳实践)
## 1. 环境准备
### 1.1 安装必要工具
```bash
# 安装 Go 语言 (1.21+)
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# 安装 Operator SDK
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.33.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH} && sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk

# 安装 OPM (Operator Package Manager)
curl -LO https://github.com/operator-framework/operator-registry/releases/download/v1.33.0/${OS}-${ARCH}-opm
chmod +x ${OS}-${ARCH}-opm && sudo mv ${OS}-${ARCH}-opm /usr/local/bin/opm

# 安装 Docker 或 Podman
# Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 验证安装
go version
operator-sdk version
opm version
docker version
```
### 1.2 准备 Kubernetes 集群
```bash
# 方案1: 使用 Kind (Kubernetes in Docker)
go install sigs.k8s.io/kind@v0.20.0
kind create cluster --name olm-test

# 方案2: 使用 Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start

# 安装 OLM
operator-sdk olm install
operator-sdk olm status
```
## 2. 创建 Operator 项目
### 2.1 创建项目结构
```bash
# 创建项目目录
mkdir -p $HOME/projects/memcached-operator
cd $HOME/projects/memcached-operator

# 初始化项目
operator-sdk init \
  --domain example.com \
  --repo github.com/example/memcached-operator

# 查看项目结构
tree -L 2
```

**项目结构**:
```
memcached-operator/
├── Dockerfile                 # 容器镜像构建文件
├── Makefile                   # 构建和部署脚本
├── PROJECT                    # 项目元数据
├── README.md
├── config/
│   ├── default/              # 默认配置
│   ├── manager/              # Manager 配置
│   ├── manifests/            # CRD 清单
│   ├── prometheus/           # 监控配置
│   ├── rbac/                 # RBAC 配置
│   └── samples/              # 示例 CR
├── api/
│   └── v1alpha1/             # API 定义
├── controllers/              # Controller 实现
├── hack/                     # 脚本工具
└── go.mod                    # Go 模块定义
```
### 2.2 创建 API 和 Controller
```bash
# 创建 Memcached API
operator-sdk create api \
  --group cache \
  --version v1alpha1 \
  --kind Memcached \
  --resource \
  --controller
```
## 3. 定义 CRD
### 3.1 定义 API 结构
编辑 `api/v1alpha1/memcached_types.go`:
```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MemcachedSpec 定义 Memcached 的期望状态
type MemcachedSpec struct {
	// Size 定义 Memcached 实例数量
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=10
	// +kubebuilder:default=3
	Size int32 `json:"size"`

	// Port 定义 Memcached 服务端口
	// +kubebuilder:validation:Minimum=1024
	// +kubebuilder:validation:Maximum=65535
	// +kubebuilder:default=11211
	Port int32 `json:"port,omitempty"`

	// Image 定义 Memcached 容器镜像
	// +kubebuilder:default="memcached:1.6.17-alpine"
	Image string `json:"image,omitempty"`

	// Resources 定义资源限制
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`

	// Config 定义 Memcached 配置参数
	Config MemcachedConfig `json:"config,omitempty"`
}

// MemcachedConfig 定义 Memcached 配置
type MemcachedConfig struct {
	// MaxMemory 定义最大内存使用量
	MaxMemory string `json:"maxMemory,omitempty"`

	// MaxConnections 定义最大连接数
	MaxConnections int32 `json:"maxConnections,omitempty"`

	// Verbose 启用详细日志
	Verbose bool `json:"verbose,omitempty"`
}

// MemcachedStatus 定义 Memcached 的观测状态
type Memv1alpha1.MemcachedStatus struct {
	// Conditions 表示当前状态条件
	// +operator-sdk:csv:customresourcedefinitions:type=spec
	Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`

	// Nodes 表示当前运行的 Pod 名称列表
	// +operator-sdk:csv:customresourcedefinitions:type=status
	Nodes []string `json:"nodes,omitempty"`

	// CurrentSize 表示当前实际运行的实例数量
	// +operator-sdk:csv:customresourcedefinitions:type=status
	CurrentSize int32 `json:"currentSize,omitempty"`

	// ServiceName 表示创建的 Service 名称
	// +operator-sdk:csv:customresourcedefinitions:type=status
	ServiceName string `json:"serviceName,omitempty"`

	// Ready 表示是否所有实例都已就绪
	// +operator-sdk:csv:customresourcedefinitions:type=status
	Ready bool `json:"ready,omitempty"`
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
//+kubebuilder:printcolumn:name="Size",type="integer",JSONPath=".spec.size"
//+kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
//+kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
//+kubebuilder:printcolumn:name="Nodes",type="string",JSONPath=".status.nodes"

// Memcached is the Schema for the memcacheds API
type Memcached struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MemcachedSpec   `json:"spec,omitempty"`
	Status MemcachedStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// MemcachedList contains a list of Memcached
type MemcachedList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Memcached `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Memcached{}, &MemcachedList{})
}
```
### 3.2 生成 CRD 清单
```bash
# 生成 CRD 和 RBAC 清单
make generate
make manifests

# 查看生成的 CRD
cat config/crd/bases/cache.example.com_memcacheds.yaml
```
## 4. 实现 Controller 逻辑
### 4.1 实现 Controller
编辑 `controllers/memcached_controller.go`:
```go
package controllers

import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/intstr"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	cachev1alpha1 "github.com/example/memcached-operator/api/v1alpha1"
)

const memcachedFinalizer = "cache.example.com/finalizer"

// MemcachedReconciler reconciles a Memcached object
type MemcachedReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
//+kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

// Reconcile is part of the main kubernetes reconciliation loop
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 获取 Memcached 实例
	memcached := &cachev1alpha1.Memcached{}
	if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
		if errors.IsNotFound(err) {
			logger.Info("Memcached resource not found. Ignoring since object must be deleted")
			return ctrl.Result{}, nil
		}
		logger.Error(err, "Failed to get Memcached")
		return ctrl.Result{}, err
	}

	// 检查是否被标记为删除
	if memcached.GetDeletionTimestamp() != nil {
		if controllerutil.ContainsFinalizer(memcached, memcachedFinalizer) {
			// 执行清理逻辑
			if err := r.cleanupResources(ctx, memcached); err != nil {
				return ctrl.Result{}, err
			}

			// 移除 finalizer
			controllerutil.RemoveFinalizer(memcached, memcachedFinalizer)
			if err := r.Update(ctx, memcached); err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	}

	// 添加 finalizer
	if !controllerutil.ContainsFinalizer(memcached, memcachedFinalizer) {
		controllerutil.AddFinalizer(memcached, memcachedFinalizer)
		if err := r.Update(ctx, memcached); err != nil {
			return ctrl.Result{}, err
		}
	}

	// 确保 Deployment 存在
	deployment, err := r.reconcileDeployment(ctx, memcached)
	if err != nil {
		return ctrl.Result{}, err
	}

	// 确保 Service 存在
	service, err := r.reconcileService(ctx, memcached)
	if err != nil {
		return ctrl.Result{}, err
	}

	// 更新状态
	if err := r.updateStatus(ctx, memcached, deployment, service); err != nil {
		return ctrl.Result{}, err
	}

	// 设置条件
	r.setCondition(memcached, "Available", metav1.ConditionTrue, "DeploymentAvailable", 
		fmt.Sprintf("Deployment %s is available", deployment.Name))

	return ctrl.Result{RequeueAfter: time.Minute}, nil
}

// reconcileDeployment 创建或更新 Deployment
func (r *MemcachedReconciler) reconcileDeployment(ctx context.Context, memcached *cachev1alpha1.Memcached) (*appsv1.Deployment, error) {
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      memcached.Name,
			Namespace: memcached.Namespace,
		},
	}

	op, err := controllerutil.CreateOrUpdate(ctx, r.Client, deployment, func() error {
		// 设置 OwnerReference
		if err := controllerutil.SetControllerReference(memcached, deployment, r.Scheme); err != nil {
			return err
		}

		// 设置 Deployment 规格
		deployment.Spec.Replicas = &memcached.Spec.Size
		deployment.Spec.Selector = &metav1.LabelSelector{
			MatchLabels: map[string]string{"app": memcached.Name},
		}
		deployment.Spec.Template.ObjectMeta.Labels = map[string]string{"app": memcached.Name}
		deployment.Spec.Template.Spec.Containers = []corev1.Container{
			{
				Name:  "memcached",
				Image: memcached.Spec.Image,
				Ports: []corev1.ContainerPort{
					{
						ContainerPort: memcached.Spec.Port,
						Name:          "memcached",
					},
				},
				Resources: memcached.Spec.Resources,
				Command:   r.buildMemcachedCommand(memcached),
				ReadinessProbe: &corev1.Probe{
					ProbeHandler: corev1.ProbeHandler{
						TCPSocket: &corev1.TCPSocketAction{
							Port: intstr.FromInt(int(memcached.Spec.Port)),
						},
					},
					InitialDelaySeconds: 5,
					TimeoutSeconds:      5,
					PeriodSeconds:       10,
				},
				LivenessProbe: &corev1.Probe{
					ProbeHandler: corev1.ProbeHandler{
						TCPSocket: &corev1.TCPSocketAction{
							Port: intstr.FromInt(int(memcached.Spec.Port)),
						},
					},
					InitialDelaySeconds: 15,
					TimeoutSeconds:      5,
					PeriodSeconds:       20,
				},
			},
		}

		return nil
	})

	if err != nil {
		return nil, err
	}

	log.FromContext(ctx).Info("Deployment reconciled", "operation", op)
	return deployment, nil
}

// reconcileService 创建或更新 Service
func (r *MemcachedReconciler) reconcileService(ctx context.Context, memcached *cachev1alpha1.Memcached) (*corev1.Service, error) {
	service := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%s-service", memcached.Name),
			Namespace: memcached.Namespace,
		},
	}

	op, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
		// 设置 OwnerReference
		if err := controllerutil.SetControllerReference(memcached, service, r.Scheme); err != nil {
			return err
		}

		// 设置 Service 规格
		service.Spec.Type = corev1.ServiceTypeClusterIP
		service.Spec.Selector = map[string]string{"app": memcached.Name}
		service.Spec.Ports = []corev1.ServicePort{
			{
				Port:       memcached.Spec.Port,
				TargetPort: intstr.FromInt(int(memcached.Spec.Port)),
				Protocol:   corev1.ProtocolTCP,
				Name:       "memcached",
			},
		}

		return nil
	})

	if err != nil {
		return nil, err
	}

	log.FromContext(ctx).Info("Service reconciled", "operation", op)
	return service, nil
}

// buildMemcachedCommand 构建 Memcached 启动命令
func (r *MemcachedReconciler) buildMemcachedCommand(memcached *cachev1alpha1.Memcached) []string {
	cmd := []string{"memcached"}

	// 端口
	cmd = append(cmd, "-p", fmt.Sprintf("%d", memcached.Spec.Port))

	// 最大内存
	if memcached.Spec.Config.MaxMemory != "" {
		cmd = append(cmd, "-m", memcached.Spec.Config.MaxMemory)
	}

	// 最大连接数
	if memcached.Spec.Config.MaxConnections > 0 {
		cmd = append(cmd, "-c", fmt.Sprintf("%d", memcached.Spec.Config.MaxConnections))
	}

	// 详细模式
	if memcached.Spec.Config.Verbose {
		cmd = append(cmd, "-vv")
	}

	return cmd
}

// updateStatus 更新 Memcached 状态
func (r *MemcachedReconciler) updateStatus(ctx context.Context, memcached *cachev1alpha1.Memcached, 
	deployment *appsv1.Deployment, service *corev1.Service) error {
	
	// 获取 Pod 列表
	podList := &corev1.PodList{}
	if err := r.List(ctx, podList, 
		client.InNamespace(memcached.Namespace),
		client.MatchingLabels{"app": memcached.Name}); err != nil {
		return err
	}

	// 提取 Pod 名称
	var nodes []string
	var readyCount int32
	for _, pod := range podList.Items {
		nodes = append(nodes, pod.Name)
		if pod.Status.Phase == corev1.PodRunning {
			for _, condition := range pod.Status.Conditions {
				if condition.Type == corev1.PodReady && condition.Status == corev1.ConditionTrue {
					readyCount++
					break
				}
			}
		}
	}

	// 更新状态
	memcached.Status.Nodes = nodes
	memcached.Status.CurrentSize = int32(len(nodes))
	memcached.Status.ServiceName = service.Name
	memcached.Status.Ready = readyCount == memcached.Spec.Size

	return r.Status().Update(ctx, memcached)
}

// setCondition 设置状态条件
func (r *MemcachedReconciler) setCondition(memcached *cachev1alpha1.Memcached, 
	conditionType string, status metav1.ConditionStatus, reason, message string) {
	
	condition := metav1.Condition{
		Type:               conditionType,
		Status:             status,
		Reason:             reason,
		Message:            message,
		LastTransitionTime: metav1.Now(),
	}

	// 查找并更新或添加条件
	for i, c := range memcached.Status.Conditions {
		if c.Type == conditionType {
			memcached.Status.Conditions[i] = condition
			return
		}
	}
	memcached.Status.Conditions = append(memcached.Status.Conditions, condition)
}

// cleanupResources 清理资源
func (r *MemcachedReconciler) cleanupResources(ctx context.Context, memcached *cachev1alpha1.Memcached) error {
	// 清理逻辑,例如删除外部资源
	return nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&cachev1alpha1.Memcached{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Complete(r)
}
```
### 4.2 生成 RBAC 清单
```bash
# 生成 RBAC 清单
make generate
make manifests
```
## 5. 生成 OLM 元数据
### 5.1 创建 CSV 注解
编辑 `config/manifests/bases/cache.example.com_memcacheds.clusterserviceversion.yaml`:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  annotations:
    alm-examples: |-
      [
        {
          "apiVersion": "cache.example.com/v1alpha1",
          "kind": "Memcached",
          "metadata": {
            "name": "memcached-sample"
          },
          "spec": {
            "size": 3,
            "port": 11211,
            "image": "memcached:1.6.17-alpine",
            "config": {
              "maxMemory": "64",
              "maxConnections": 1024,
              "verbose": false
            },
            "resources": {
              "requests": {
                "cpu": "100m",
                "memory": "64Mi"
              },
              "limits": {
                "cpu": "500m",
                "memory": "256Mi"
              }
            }
          }
        }
      ]
    capabilities: Basic Install
    categories: Database
    containerImage: quay.io/example/memcached-operator:v1.0.0
    createdAt: "2024-01-01T00:00:00Z"
    description: Memcached Operator manages Memcached clusters on Kubernetes
    operatorframework.io/suggested-namespace: memcached-operator
    operators.operatorframework.io/builder: operator-sdk-v1.33.0
    operators.operatorframework.io/project_layout: go.kubebuilder.io/v4
    repository: https://github.com/example/memcached-operator
    support: Example Inc.
  name: memcached-operator.v1.0.0
  namespace: placeholder
spec:
  apiservicedefinitions: {}
  customresourcedefinitions:
    owned:
    - description: Memcached is the Schema for the memcacheds API
      displayName: Memcached
      kind: Memcached
      name: memcacheds.cache.example.com
      version: v1alpha1
      specDescriptors:
      - description: Size defines the number of Memcached instances
        displayName: Size
        path: size
        x-descriptors:
        - urn:alm:descriptor:com.tectonic.ui:podCount
      - description: Port defines the Memcached service port
        displayName: Port
        path: port
        x-descriptors:
        - urn:alm:descriptor:com.tectonic.ui:number
      - description: Image defines the Memcached container image
        displayName: Image
        path: image
        x-descriptors:
        - urn:alm:descriptor:com.tectonic.ui:imagePullPolicy
      statusDescriptors:
      - description: Nodes represents the list of current running pod names
        displayName: Nodes
        path: nodes
        x-descriptors:
        - urn:alm:descriptor:com.tectonic.ui:text
      - description: Ready represents whether all instances are ready
        displayName: Ready
        path: ready
        x-descriptors:
        - urn:alm:descriptor:com.tectonic.ui:booleanSwitch
  description: |
    ## Memcached Operator
    
    The Memcached Operator manages Memcached clusters on Kubernetes. It provides:
    
    ### Features
    - **Automated Deployment**: Automatically deploys and manages Memcached instances
    - **Scaling**: Scale Memcached clusters up or down
    - **Self-Healing**: Automatic recovery from failures
    - **Configuration Management**: Manage Memcached configuration through CRDs
    
    ### Quick Start
    1. Install the operator
    2. Create a Memcached CR:
       ```yaml
       apiVersion: cache.example.com/v1alpha1
       kind: Memcached
       metadata:
         name: memcached-sample
       spec:
         size: 3
       ```
    3. The operator will create a 3-node Memcached cluster
  displayName: Memcached Operator
  icon:
  - base64data: PHN2ZyBpZD0iTGF5ZXJfMSIgZGF0YS1uYW1lPSJMYXllciAxIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAxMDI0IDEwMjQiPjxkZWZzPjxzdHlsZT4uY2xzLTF7ZmlsbDojZmZmO30uY2xzLTJ7ZmlsbDojMzMzO308L3N0eWxlPjwvZGVmcz48dGl0bGU+bWVtY2FjaGVkPC90aXRsZT48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik04ODkuMDYsNjQwYy0zLjUzLTMuNS04LjI2LTUuNDQtMTMuMDYtNS40NHMtOS41MywxLjk0LTEzLjA2LDUuNDRMODI1LDY3Ny45VjUxMmMwLTI4LjcyLTIzLjI4LTUyLTUyLTUySDYwMFYyNTJjMC0yOC43Mi0yMy4yOC01Mi01Mi01MkgxNzVjLTI4LjcyLDAtNTIsMjMuMjgtNTIsNTJWNDUyYzAsMjguNzIsMjMuMjgsNTIsNTIsNTJIMjUwVjc2OGMwLDI4LjcyLDIzLjI4LDUyLDUyLDUySDU0OFY4ODljMCwyOC43MiwyMy4yOCw1Miw1Miw1Mkg3NzNjMjguNzIsMCw1Mi0yMy4yOCw1Mi01MlY3NjhoMThjMjguNzIsMCw1Mi0yMy4yOCw1Mi01MlY2NTNjMC00LjgtMS45NC05LjUzLTUuNDQtMTMuMDZaTTE3NSw0NTJWMjUySDU0OFY0NTJaTTU0OCw3NjhIMzAyVjUwNEg1NDhabTIyNSwxMjFINjAwVjc2OGgxNzNaIi8+PC9zdmc+
    mediatype: image/svg+xml
  install:
    spec:
      clusterPermissions:
      - rules:
        - apiGroups:
          - cache.example.com
          resources:
          - memcacheds
          verbs:
          - create
          - delete
          - get
          - list
          - patch
          - update
          - watch
        - apiGroups:
          - cache.example.com
          resources:
          - memcacheds/finalizers
          verbs:
          - update
        - apiGroups:
          - cache.example.com
          resources:
          - memcacheds/status
          verbs:
          - get
          - patch
          - update
        - apiGroups:
          - apps
          resources:
          - deployments
          verbs:
          - create
          - delete
          - get
          - list
          - patch
          - update
          - watch
        - apiGroups:
          - ""
          resources:
          - pods
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - ""
          resources:
          - services
          verbs:
          - create
          - delete
          - get
          - list
          - patch
          - update
          - watch
        serviceAccountName: memcached-operator-controller-manager
      deployments:
      - label:
          app.kubernetes.io/component: manager
          app.kubernetes.io/created-by: memcached-operator
          app.kubernetes.io/instance: controller-manager
          app.kubernetes.io/managed-by: kustomize
          app.kubernetes.io/name: deployment
          app.kubernetes.io/part-of: memcached-operator
          control-plane: controller-manager
        name: memcached-operator-controller-manager
        spec:
          replicas: 1
          selector:
            matchLabels:
              control-plane: controller-manager
          strategy: {}
          template:
            metadata:
              annotations:
                kubectl.kubernetes.io/default-container: manager
              labels:
                control-plane: controller-manager
            spec:
              containers:
              - args:
                - --secure-listen-address=0.0.0.0:8443
                - --upstream=http://127.0.0.1:8080/
                - --logtostderr=true
                - --v=0
                image: gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
                name: kube-rbac-proxy
                ports:
                - containerPort: 8443
                  name: https
                  protocol: TCP
                resources:
                  limits:
                    cpu: 500m
                    memory: 128Mi
                  requests:
                    cpu: 5m
                    memory: 64Mi
                securityContext:
                  allowPrivilegeEscalation: false
                  capabilities:
                    drop:
                    - ALL
              - args:
                - --health-probe-bind-address=:8081
                - --metrics-bind-address=127.0.0.1:8080
                - --leader-elect
                command:
                - /manager
                image: quay.io/example/memcached-operator:v1.0.0
                livenessProbe:
                  httpGet:
                    path: /healthz
                    port: 8081
                  initialDelaySeconds: 15
                  periodSeconds: 20
                name: manager
                readinessProbe:
                  httpGet:
                    path: /readyz
                    port: 8081
                  initialDelaySeconds: 5
                  periodSeconds: 10
                resources:
                  limits:
                    cpu: 500m
                    memory: 128Mi
                  requests:
                    cpu: 10m
                    memory: 64Mi
                securityContext:
                  allowPrivilegeEscalation: false
                  capabilities:
                    drop:
                    - ALL
              securityContext:
                runAsNonRoot: true
              serviceAccountName: memcached-operator-controller-manager
              terminationGracePeriodSeconds: 10
      permissions:
      - rules:
        - apiGroups:
          - ""
          resources:
          - configmaps
          verbs:
          - get
          - list
          - watch
          - create
          - update
          - patch
          - delete
        - apiGroups:
          - coordination.k8s.io
          resources:
          - leases
          verbs:
          - get
          - list
          - watch
          - create
          - update
          - patch
          - delete
        - apiGroups:
          - ""
          resources:
          - events
          verbs:
          - create
          - patch
        serviceAccountName: memcached-operator-controller-manager
    strategy: deployment
  installModes:
  - supported: false
    type: OwnNamespace
  - supported: false
    type: SingleNamespace
  - supported: false
    type: MultiNamespace
  - supported: true
    type: AllNamespaces
  keywords:
  - memcached
  - cache
  - database
  labels:
    operatorframework.io/arch.amd64: supported
    operatorframework.io/os.linux: supported
  links:
  - name: Memcached Operator
    url: https://github.com/example/memcached-operator
  maintainers:
  - email: support@example.com
    name: Example Support
  maturity: stable
  minKubeVersion: 1.21.0
  provider:
    name: Example Inc.
    url: https://example.com
  replaces: memcached-operator.v0.9.0
  skips:
  - memcached-operator.v0.9.1
  version: 1.0.0
```
### 5.2 创建 Package Manifest
创建 `config/manifests/bases/memcached-operator-package.yaml`:
```yaml
packageName: memcached-operator
channels:
- name: stable
  currentCSV: memcached-operator.v1.0.0
- name: alpha
  currentCSV: memcached-operator.v1.0.0
defaultChannel: stable
```
### 5.3 生成 Bundle
```bash
# 生成 Bundle
make bundle

# 验证 Bundle
operator-sdk bundle validate ./bundle --select-optional name=community
```
## 6. 本地测试
### 6.1 使用 Operator SDK 运行
```bash
# 安装 CRD
make install

# 本地运行 Operator
make run

# 在另一个终端创建 CR
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml

# 查看 CR 状态
kubectl get memcached -o yaml

# 查看 Operator 日志
# 在运行 make run 的终端查看

# 清理
make uninstall
```
### 6.2 使用 OLM 测试
```bash
# 构建 Bundle 镜像
make bundle-build BUNDLE_IMG=quay.io/example/memcached-operator-bundle:v1.0.0

# 推送 Bundle 镜像
docker push quay.io/example/memcached-operator-bundle:v1.0.0

# 运行 Operator SDK scorecard
operator-sdk scorecard ./bundle

# 使用 OLM 测试
kubectl create ns memcached-test
operator-sdk run bundle quay.io/example/memcached-operator-bundle:v1.0.0 -n memcached-test

# 创建 CR
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml -n memcached-test

# 清理
operator-sdk cleanup memcached-operator -n memcached-test
kubectl delete ns memcached-test
```
## 7. 打包和发布
### 7.1 构建和推送镜像
```bash
# 设置镜像仓库
export IMAGE_TAG_BASE=quay.io/example/memcached-operator

# 构建所有镜像
make docker-build docker-push

# 构建 Bundle 镜像
make bundle-build bundle-push

# 构建目录镜像
make catalog-build catalog-push
```
### 7.2 创建 Index Image
```bash
# 创建初始 Index Image
opm index add \
  --bundles quay.io/example/memcached-operator-bundle:v1.0.0 \
  --tag quay.io/example/memcached-operator-index:v1.0.0 \
  --build-tool docker

# 推送 Index Image
docker push quay.io/example/memcached-operator-index:v1.0.0

# 添加新版本到现有 Index Image
opm index add \
  --bundles quay.io/example/memcached-operator-bundle:v1.1.0 \
  --from-index quay.io/example/memcached-operator-index:v1.0.0 \
  --tag quay.io/example/memcached-operator-index:v1.1.0 \
  --build-tool docker

docker push quay.io/example/memcached-operator-index:v1.1.0
```
### 7.3 创建 CatalogSource
创建 `catalog-source.yaml`:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: memcached-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Memcached Operator Catalog
  image: quay.io/example/memcached-operator-index:v1.0.0
  publisher: Example Inc.
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 30m
```
```bash
# 应用 CatalogSource
kubectl apply -f catalog-source.yaml

# 验证 CatalogSource
kubectl get catalogsource -n openshift-marketplace
kubectl get pods -n openshift-marketplace -l olm.catalogSource=memcached-operator-catalog
```
## 8. 通过 OLM 安装
### 8.1 创建 OperatorGroup
创建 `operator-group.yaml`:
```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: memcached-operator-group
  namespace: memcached-operator
spec:
  targetNamespaces:
  - memcached-operator
```

```bash
# 创建命名空间
kubectl create ns memcached-operator

# 创建 OperatorGroup
kubectl apply -f operator-group.yaml
```
### 8.2 创建 Subscription
创建 `subscription.yaml`:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: memcached-operator-subscription
  namespace: memcached-operator
spec:
  channel: stable
  name: memcached-operator
  source: memcached-operator-catalog
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

```bash
# 创建 Subscription
kubectl apply -f subscription.yaml

# 查看安装状态
kubectl get subscription -n memcached-operator
kubectl get installplan -n memcached-operator
kubectl get csv -n memcached-operator
kubectl get pods -n memcached-operator
```
### 8.3 验证安装
```bash
# 查看 CSV 状态
kubectl get csv -n memcached-operator -o yaml

# 查看 Operator Pod
kubectl get pods -n memcached-operator -l control-plane=controller-manager

# 查看 Operator 日志
kubectl logs -n memcached-operator -l control-plane=controller-manager -c manager
```
### 8.4 创建 Memcached 实例
```bash
# 创建 Memcached 实例
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml -n memcached-operator

# 查看实例状态
kubectl get memcached -n memcached-operator
kubectl describe memcached memcached-sample -n memcached-operator

# 查看创建的资源
kubectl get deployment -n memcached-operator
kubectl get service -n memcached-operator
kubectl get pods -n memcached-operator
```
## 9. 高级功能
### 9.1 添加 Webhook
#### 9.1.1 创建 Webhook
```bash
# 创建 Webhook
operator-sdk create webhook \
  --group cache \
  --version v1alpha1 \
  --kind Memcached \
  --defaulting \
  --programmatic-validation
```
#### 9.1.2 实现验证逻辑
编辑 `api/v1alpha1/memcached_webhook.go`:
```go
package v1alpha1

import (
	"fmt"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

// log is for logging in this package.
var memcachedlog = logf.Log.WithName("memcached-resource")

func (r *Memcached) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

//+kubebuilder:webhook:path=/mutate-cache-example-com-v1alpha1-memcached,mutating=true,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1alpha1,name=mmemcached.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Memcached{}

// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *Memcached) Default() {
	memcachedlog.Info("default", "name", r.Name)

	// 设置默认值
	if r.Spec.Size == 0 {
		r.Spec.Size = 3
	}
	if r.Spec.Port == 0 {
		r.Spec.Port = 11211
	}
	if r.Spec.Image == "" {
		r.Spec.Image = "memcached:1.6.17-alpine"
	}
}

//+kubebuilder:webhook:path=/validate-cache-example-com-v1alpha1-memcached,mutating=false,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1alpha1,name=vmemcached.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &Memcached{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateCreate() (admission.Warnings, error) {
	memcachedlog.Info("validate create", "name", r.Name)

	return nil, r.validateMemcached()
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
	memcachedlog.Info("validate update", "name", r.Name)

	return nil, r.validateMemcached()
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *Memcached) ValidateDelete() (admission.Warnings, error) {
	memcachedlog.Info("validate delete", "name", r.Name)

	return nil, nil
}

func (r *Memcached) validateMemcached() error {
	// 验证 Size
	if r.Spec.Size < 1 || r.Spec.Size > 10 {
		return fmt.Errorf("size must be between 1 and 10, got %d", r.Spec.Size)
	}

	// 验证 Port
	if r.Spec.Port < 1024 || r.Spec.Port > 65535 {
		return fmt.Errorf("port must be between 1024 and 65535, got %d", r.Spec.Port)
	}

	// 验证 Image
	if r.Spec.Image == "" {
		return fmt.Errorf("image must be specified")
	}

	return nil
}
```
### 9.2 添加监控指标
#### 9.2.1 定义指标
编辑 `controllers/memcached_controller.go`:
```go
import (
	"github.com/prometheus/client_golang/prometheus"
	"sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
	memcachedInstancesTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "memcached_instances_total",
			Help: "Total number of Memcached instances",
		},
		[]string{"namespace"},
	)

	memcachedSizeGauge = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "memcached_size",
			Help: "Current size of Memcached instances",
		},
		[]string{"namespace", "name"},
	)

	reconcileDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "memcached_reconcile_duration_seconds",
			Help:    "Duration of reconciliation",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"namespace", "name"},
	)
)

func init() {
	metrics.Registry.MustRegister(memcachedInstancesTotal)
	metrics.Registry.MustRegister(memcachedSizeGauge)
	metrics.Registry.MustRegister(reconcileDuration)
}

func (r *MemcachedReconciler) recordMetrics(memcached *cachev1alpha1.Memcached, duration time.Duration) {
	memcachedInstancesTotal.WithLabelValues(memcached.Namespace).Inc()
	memcachedSizeGauge.WithLabelValues(memcached.Namespace, memcached.Name).Set(float64(memcached.Spec.Size))
	reconcileDuration.WithLabelValues(memcached.Namespace, memcached.Name).Observe(duration.Seconds())
}
```
#### 9.2.2 配置 ServiceMonitor
创建 `config/prometheus/monitor.yaml`:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    control-plane: controller-manager
  name: memcached-operator-metrics-monitor
  namespace: system
spec:
  endpoints:
  - path: /metrics
    port: https
    scheme: https
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    tlsConfig:
      insecureSkipVerify: true
  selector:
    matchLabels:
      control-plane: controller-manager
```
### 9.3 添加依赖管理
#### 9.3.1 定义依赖
编辑 CSV 文件:
```yaml
spec:
  customresourcedefinitions:
    required:
    - name: prometheuses.monitoring.coreos.com
      version: v1
      description: Requires Prometheus Operator
```
#### 9.3.2 实现依赖检查
```go
func (r *MemcachedReconciler) checkDependencies(ctx context.Context) error {
	// 检查 Prometheus CRD 是否存在
	prometheusCRD := &apiextensionsv1.CustomResourceDefinition{}
	err := r.Get(ctx, client.ObjectKey{Name: "prometheuses.monitoring.coreos.com"}, prometheusCRD)
	if err != nil {
		if errors.IsNotFound(err) {
			return fmt.Errorf("Prometheus CRD not found, please install Prometheus Operator first")
		}
		return err
	}

	return nil
}
```
## 10. 最佳实践
### 10.1 版本管理
```bash
# 语义化版本控制
# v1.0.0 -> v1.0.1 (补丁版本,bug 修复)
# v1.0.0 -> v1.1.0 (次版本,新功能,向后兼容)
# v1.0.0 -> v2.0.0 (主版本,破坏性变更)

# 在 CSV 中定义升级路径
spec:
  replaces: memcached-operator.v0.9.0
  skips:
  - memcached-operator.v0.9.1  # 跳过有问题的版本
```
### 10.2 错误处理
```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// 区分可恢复和不可恢复错误
	if err := r.doSomething(ctx); err != nil {
		if errors.IsNotFound(err) {
			// 资源不存在,无需重新排队
			return ctrl.Result{}, nil
		}
		if errors.IsConflict(err) {
			// 冲突错误,稍后重试
			return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
		}
		// 其他错误,记录并返回
		return ctrl.Result{}, err
	}

	// 正常重新排队
	return ctrl.Result{RequeueAfter: time.Minute}, nil
}
```
### 10.3 资源限制
```yaml
# 在 CSV 中设置资源限制
spec:
  install:
    spec:
      deployments:
      - name: memcached-operator-controller-manager
        spec:
          template:
            spec:
              containers:
              - name: manager
                resources:
                  limits:
                    cpu: 500m
                    memory: 128Mi
                  requests:
                    cpu: 10m
                    memory: 64Mi
```
### 10.4 安全最佳实践
```yaml
# 在 CSV 中设置安全上下文
spec:
  install:
    spec:
      deployments:
      - name: memcached-operator-controller-manager
        spec:
          template:
            spec:
              containers:
              - name: manager
                securityContext:
                  allowPrivilegeEscalation: false
                  capabilities:
                    drop:
                    - ALL
                  readOnlyRootFilesystem: true
                  runAsNonRoot: true
                  runAsUser: 1000
```
### 10.5 测试策略
```bash
# 单元测试
make test

# 端到端测试
make test-e2e

# Scorecard 测试
operator-sdk scorecard ./bundle

# Bundle 验证
operator-sdk bundle validate ./bundle --select-optional name=community
```
### 10.6 文档最佳实践
```yaml
# 在 CSV 中提供详细描述
spec:
  description: |
    ## Memcached Operator
    
    ### Overview
    The Memcached Operator manages Memcached clusters on Kubernetes.
    
    ### Features
    - Automated deployment and scaling
    - Self-healing capabilities
    - Configuration management
    
    ### Prerequisites
    - Kubernetes 1.21+
    - OLM installed
    
    ### Quick Start
    ```yaml
    apiVersion: cache.example.com/v1alpha1
    kind: Memcached
    metadata:
      name: memcached-sample
    spec:
      size: 3
    ```
```
## 总结
本教程提供了开发被 OLM 管理的 Operator 的完整流程:
1. **环境准备**: 安装必要的工具和准备 Kubernetes 集群
2. **项目创建**: 使用 Operator SDK 初始化项目
3. **CRD 定义**: 定义自定义资源 API
4. **Controller 实现**: 实现业务逻辑
5. **OLM 元数据**: 生成 CSV 和 Package Manifest
6. **本地测试**: 在本地环境中测试 Operator
7. **打包发布**: 构建镜像并发布到目录
8. **OLM 安装**: 通过 OLM 安装和管理 Operator
9. **高级功能**: Webhook、监控、依赖管理
10. **最佳实践**: 版本管理、错误处理、安全、测试

通过遵循这个教程,您可以开发出生产级别的、被 OLM 管理的 Operator,为用户提供可靠的 Kubernetes 应用管理体验。


