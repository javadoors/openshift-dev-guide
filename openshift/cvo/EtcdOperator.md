# OpenShift Etcd Operator 概要设计
## 一、定位与职责
Etcd Operator 是 OpenShift Cluster Version Operator (CVO) 管理的核心组件之一，负责**声明式地管理集群 etcd 集群的生命周期**。它是 OpenShift 控制平面自托管架构的关键一环——etcd 运行在集群自身的 Pod 中，由 Operator 自动化管理。

**核心职责**：
- etcd 集群的自动化部署与初始化
- etcd 成员的滚动升级（版本 + 配置）
- etcd 集群的运行时健康保障（自愈）
- etcd 备份与灾难恢复
- etcd 运维操作（Defrag、空间回收等）
## 二、架构总览
```
┌──────────────────────────────────────────────────────────────────┐
│                     OpenShift Control Plane                      │
│                                                                  │
│  ┌──────────┐     ┌──────────────┐    ┌───────────────────────┐  │
│  │   CVO    │───▶│  Etcd        │───▶│  etcd Pod (Static)    │  │
│  │          │    │  Operator    │     │  ┌─────────────────┐  │  │
│  │ 管理清单  │    │              │     │  │ etcd member     │  │  │
│  │ 版本定义  │    │ Reconcile    │     │  │ (etcd binary)   │  │  │
│  └──────────┘    │ Loop         │     │  └─────────────────┘  │  │
│                  │              │     │  ┌─────────────────┐  │  │
│                  │ Status       │     │  │ etcdctl sidecar │  │  │
│                  │ Reporting    │     │  │ (健康检查/备份)  │  │  │
│                  └──────┬───────┘     │  └─────────────────┘  │  │
│                         │             └───────────────────────┘  │
│                         │                                       │
│                  ┌──────▼───────┐                               │
│                  │   Config     │                               │
│                  │   Maps/      │                               │
│                  │   Secrets    │                               │
│                  └──────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```
## 三、核心 CRD 设计
### 3.1 Etcd（自定义资源）
```yaml
apiVersion: operator.openshift.io/v1
kind: Etcd
metadata:
  name: cluster
spec:
  managementState: Managed          # Managed | Unmanaged | Removed
  observedConfig:                   # 上一次已应用的配置快照
    storage:
      pvc:
        size: 25Gi
  unsupportedConfigOverrides:       # 不支持的配置覆盖（紧急用）
    apiServerArguments: {}
status:
  conditions:
    - type: Available
      status: "True"
    - type: Progressing
      status: "False"
    - type: Degraded
      status: "False"
    - type: Upgradeable
      status: "True"
      message: "etcd members are healthy, safe to upgrade"
  members:
    - id: 1
      name: etcd-member-master-0
      peerURLs: ["https://10.0.0.1:2380"]
      clientURLs: ["https://10.0.0.1:2379"]
      health: healthy
      version: "3.5.9"
  currentVersion: "3.5.9"
  desiredVersion: "3.5.9"
  recentBackup:
    startTime: "2026-04-20T10:00:00Z"
    completionTime: "2026-04-20T10:01:30Z"
    snapshotSize: 128MB
```
**关键 Status Condition**：

| Condition | 含义 |
|-----------|------|
| `Available` | etcd 集群是否有法定人数（quorum）可用 |
| `Progressing` | 是否正在执行变更操作（升级/扩缩容） |
| `Degraded` | 是否存在降级问题（成员不健康/磁盘不足） |
| `Upgradeable` | 当前是否可以安全升级（阻塞 CVO 升级的关键信号） |
## 四、核心控制器设计
### 4.1 Reconcile 主循环
```
┌──────────────────────────────────────────────────────────────┐
│                    Reconcile Loop                            │
│                                                              │
│  1. 读取 Etcd CR 和集群状态                                   │
│  2. 检查 managementState                                     │
│     ├── Unmanaged → 仅上报 Status，不做变更                   │
│     └── Managed → 继续                                       │
│  3. preFlightChecks()  前置检查                              │
│     ├── etcd 集群是否有 quorum                               │
│     ├── 磁盘空间是否充足                                      │
│     └── 当前是否可安全执行变更                                 │
│  4. syncEtcdCluster()  主同步逻辑                             │
│     ├── syncStaticPods()     同步 Static Pod 清单             │
│     ├── syncCertificates()   同步 TLS 证书                    │
│     ├── syncConfigMaps()     同步配置                         │
│     └── syncRBAC()           同步权限                         │
│  5. runHealthChecks()  健康检查                               │
│  6. updateStatus()     更新 CR Status                        │
│  7. reportUpgradeReadiness()  上报升级就绪状态                │
└──────────────────────────────────────────────────────────────┘
```
### 4.2 Static Pod 管理
OpenShift 的 etcd 以 **Static Pod** 方式运行（非 Deployment/DaemonSet），Operator 通过写入 Manifest 文件到 `/etc/kubernetes/manifests/` 来管理：
```
┌──────────────────────────────────────────────────────────────┐
│               Static Pod 同步流程                             │
│                                                              │
│  1. 计算期望的 Pod Manifest                                   │
│     ├── 从 CVO 获取目标 etcd 版本                             │
│     ├── 从 Etcd CR 获取配置参数                               │
│     ├── 从 Secret 挂载 TLS 证书                               │
│     └── 渲染 Pod Manifest 模板                                │
│                                                              │
│  2. 对比当前 Manifest 与期望 Manifest                         │
│     ├── 相同 → 跳过                                          │
│     └── 不同 → 写入新 Manifest                               │
│         → kubelet 自动检测变更，重建 Pod                      │
│                                                              │
│  3. 等待 Pod 就绪                                            │
│     ├── Pod Running + Ready                                 │
│     ├── etcd member 健康检查通过                              │
│     └── 超时 → Degraded                                      │
└──────────────────────────────────────────────────────────────┘
```
**Pod 结构**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-member-master-0
  namespace: openshift-etcd
spec:
  containers:
    - name: etcd
      image: quay.io/openshift/etcd:v3.5.9
      command:
        - /bin/sh
        - -c
        - |
          exec etcd \
            --name=etcd-member-master-0 \
            --data-dir=/var/lib/etcd \
            --listen-client-urls=https://0.0.0.0:2379 \
            --advertise-client-urls=https://10.0.0.1:2379 \
            --listen-peer-urls=https://0.0.0.0:2380 \
            --initial-advertise-peer-urls=https://10.0.0.1:2380 \
            --initial-cluster=etcd-member-master-0=https://10.0.0.1:2380,... \
            --initial-cluster-state=existing \
            --client-cert-auth \
            --trusted-ca-file=/etc/etcd/tls/ca.crt \
            --cert-file=/etc/etcd/tls/server.crt \
            --key-file=/etc/etcd/tls/server.key \
            --peer-client-cert-auth \
            --peer-trusted-ca-file=/etc/etcd/tls/ca.crt \
            --peer-cert-file=/etc/etcd/tls/peer.crt \
            --peer-key-file=/etc/etcd/tls/peer.key
      volumeMounts:
        - name: etcd-data
          mountPath: /var/lib/etcd
        - name: etcd-certs
          mountPath: /etc/etcd/tls
    - name: etcd-metrics
      image: quay.io/openshift/etcd:v3.5.9
      command: ["/usr/bin/etcdctl", "endpoint", "health", "--cluster"]

    - name: etcd-health-monitor  # Sidecar: 持续健康检查
      image: quay.io/openshift/etcd:v3.5.9
      command: ["/bin/bash", "-c", "while true; do etcdctl endpoint health; sleep 5; done"]
  volumes:
    - name: etcd-data
      hostPath:
        path: /var/lib/etcd
    - name: etcd-certs
      secret:
        secretName: etcd-serving
```
## 五、升级流程设计
### 5.1 滚动升级策略
```
┌──────────────────────────────────────────────────────────────────┐
│                  Etcd Rolling Upgrade Flow                        │
│                                                                   │
│  前置条件:                                                        │
│    ✓ etcd 集群健康（有 quorum）                                   │
│    ✓ 所有 member 均可通信                                         │
│    ✓ 磁盘空间充足                                                 │
│    ✓ Upgradeable condition = True                                 │
│                                                                   │
│  Step 1: 设置 Status.Progressing = True                          │
│                                                                   │
│  Step 2: 逐成员滚动升级（一次一个）                               │
│                                                                   │
│    ┌──────────────────────────────────────────────────────┐      │
│    │  对每个 etcd member（按特定顺序）：                    │      │
│    │                                                       │      │
│    │  2a. preUpgradeHealthCheck()                          │      │
│    │      → 确认集群当前健康                               │      │
│    │                                                       │      │
│    │  2b. backupMember()  可选：升级前备份                  │      │
│    │      → etcdctl snapshot save                          │      │
│    │                                                       │      │
│    │  2c. 更新该 member 的 Static Pod Manifest             │      │
│    │      → 新版本镜像                                     │      │
│    │      → kubelet 重建 Pod                               │      │
│    │                                                       │      │
│    │  2d. waitForMemberReady()                             │      │
│    │      ├── Pod Running + Ready                          │      │
│    │      ├── etcdctl endpoint health 通过                 │      │
│    │      ├── etcdctl member list 包含该成员               │      │
│    │      └── 超时 → Degraded + 回滚                      │      │
│    │                                                       │      │
│    │  2e. postUpgradeHealthCheck()                         │      │
│    │      → 确认整个集群仍然健康                            │      │
│    │                                                       │      │
│    │  2f. 继续下一个 member                                │      │
│    └──────────────────────────────────────────────────────┘      │
│                                                                   │
│  Step 3: 更新 Status.currentVersion = desiredVersion              │
│                                                                   │
│  Step 4: 设置 Status.Progressing = False                          │
│          设置 Status.Upgradeable = True                            │
└──────────────────────────────────────────────────────────────────┘
```
### 5.2 升级安全保障
| 保障机制 | 说明 |
|----------|------|
| **Quorum 保护** | 一次只升级一个 member，确保始终有 (N/2+1) 个 member 可用 |
| **前置健康检查** | 每次升级 member 前确认集群健康 |
| **后置健康检查** | 每个 member 升级后确认集群恢复健康 |
| **Upgradeable 信号** | 集群不健康时阻塞 CVO 发起升级 |
| **自动回滚** | 升级超时或健康检查失败时，回退到旧版本 Manifest |
| **版本兼容性** | etcd 支持跨小版本升级（3.4 → 3.5），但需要按序升级 |
## 六、备份与灾难恢复设计
### 6.1 自动备份
```
┌──────────────────────────────────────────────────────────────┐
│                   Automated Backup                            │
│                                                               │
│  触发方式:                                                    │
│    ├── 定时触发（默认每 2 小时）                               │
│    ├── 升级前自动触发                                         │
│    └── 手动触发（通过 annotation）                             │
│                                                               │
│  备份流程:                                                    │
│    1. 选择一个健康的 etcd member                              │
│    2. etcdctl snapshot save /var/lib/etcd-backup/snapshot.db  │
│    3. 校验快照完整性 (etcdctl snapshot status)                │
│    4. 上传到备份存储（PVC / S3 / 本地）                       │
│    5. 更新 Etcd CR Status.recentBackup                       │
│    6. 清理过期备份（保留最近 N 份）                            │
└──────────────────────────────────────────────────────────────┘
```
### 6.2 灾难恢复
```
┌──────────────────────────────────────────────────────────────┐
│                Disaster Recovery                              │
│                                                               │
│  场景1: 少数 member 丢失（仍有 quorum）                       │
│    → 自动移除失败 member                                      │
│    → 重新添加新 member（从现有 member 同步数据）              │
│                                                               │
│  场景2: 多数 member 丢失（失去 quorum）                       │
│    → 需要人工介入                                             │
│    → 从最近备份恢复                                           │
│    → etcdctl snapshot restore → 重建数据目录                  │
│    → 逐个启动 member，使用 --initial-cluster-state=existing   │
│                                                               │
│  场景3: 数据损坏                                              │
│    → 从备份恢复到新数据目录                                   │
│    → 滚动重启所有 member                                      │
└──────────────────────────────────────────────────────────────┘
```
## 七、证书轮换设计
```
┌──────────────────────────────────────────────────────────────┐
│              Certificate Rotation                             │
│                                                               │
│  证书类型:                                                    │
│    ├── Server 证书 (etcd serving)                             │
│    ├── Peer 证书 (etcd peer communication)                    │
│    ├── Client 证书 (API Server → etcd)                        │
│    └── CA 证书 (etcd CA)                                      │
│                                                               │
│  轮换流程:                                                    │
│    1. 检测证书即将过期（< 80% 生命周期）                      │
│    2. 生成新证书，写入新 Secret                               │
│    3. 滚动更新 etcd Pod（逐个）                               │
│       → 加载新证书                                            │
│    4. 更新 API Server 的 etcd 客户端证书                      │
│    5. 确认所有连接使用新证书                                   │
│    6. 清理旧 Secret                                           │
│                                                               │
│  CA 轮换（特殊处理）:                                         │
│    → 两阶段: 先信任新旧 CA → 再仅信任新 CA                    │
│    → 确保滚动过程中不中断连接                                 │
└──────────────────────────────────────────────────────────────┘
```
## 八、健康检查与自愈设计
### 8.1 多层健康检查
```
┌──────────────────────────────────────────────────────────────┐
│              Health Check Hierarchy                           │
│                                                               │
│  Layer 1: Pod 级别                                            │
│    ├── Pod Phase == Running                                   │
│    ├── Container Ready == True                                │
│    └── Restart Count < threshold                              │
│                                                               │
│  Layer 2: etcd Member 级别                                    │
│    ├── etcdctl endpoint health                                │
│    ├── etcdctl alarm list (无空间告警)                        │
│    ├── etcdctl endpoint status (延迟/DB大小正常)              │
│    └── Member 是否在集群成员列表中                            │
│                                                               │
│  Layer 3: etcd Cluster 级别                                   │
│    ├── 是否有 quorum (N/2+1 members healthy)                  │
│    ├── Leader 是否存在                                        │
│    ├── 所有 member 版本是否一致                                │
│    └── 数据一致性检查 (revision diff)                         │
│                                                               │
│  Layer 4: 基础设施级别                                        │
│    ├── 磁盘空间 > 10% 可用                                    │
│    ├── 磁盘 IO 延迟 < 阈值                                    │
│    └── 网络连通性 (peer-to-peer)                              │
└──────────────────────────────────────────────────────────────┘
```
### 8.2 自愈策略
| 故障场景 | 自愈动作 |
|----------|----------|
| Pod 崩溃 | kubelet 自动重启 Static Pod |
| Member 启动失败 | 重写 Manifest + 重启 Pod |
| 磁盘空间不足 | 触发 Defrag + 告警 |
| Member 网络分区 | 等待恢复 + 超时后标记 Degraded |
| 证书过期 | 自动轮换证书 + 滚动重启 |
| Member 数据损坏 | 从其他 member 同步数据 |
## 九、Defrag 与空间管理
```
┌──────────────────────────────────────────────────────────────┐
│                  Defragmentation                              │
│                                                               │
│  触发条件:                                                    │
│    ├── DB 大小超过阈值（默认 2GB）                             │
│    ├── 碎片率超过阈值                                         │
│    └── 手动触发（通过 annotation）                             │
│                                                               │
│  执行流程:                                                    │
│    1. 检查集群健康                                            │
│    2. 逐 member 执行 defrag（一次一个）                       │
│       → etcdctl defrag --cluster                              │
│    3. 每个 defrag 后等待 member 恢复                          │
│    4. 确认集群健康                                            │
│                                                               │
│  ⚠️ 注意: defrag 期间 member 短暂不可用                      │
│     必须逐个执行，保证 quorum                                 │
└──────────────────────────────────────────────────────────────┘
```
## 十、与 CVO 的集成
```
┌──────────────────────────────────────────────────────────────┐
│               CVO Integration                                 │
│                                                               │
│  CVO 管理清单:                                                │
│    - etcd Operator Deployment                                 │
│    - Etcd CR (cluster)                                        │
│    - RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)   │
│    - ConfigMap (etcd 配置模板)                                │
│                                                               │
│  升级协作:                                                    │
│    1. CVO 更新 etcd Operator 镜像版本                         │
│    2. etcd Operator 滚动重启                                  │
│    3. 新 Operator 读取新目标版本                              │
│    4. 执行 etcd member 滚动升级                               │
│    5. 升级完成后上报 Status.Upgradeable = True                │
│    6. CVO 继续升级下一个组件                                  │
│                                                               │
│  阻塞机制:                                                    │
│    Status.Upgradeable = False → CVO 暂停升级                 │
│    Status.Degraded = True → CVO 报告集群降级                  │
└──────────────────────────────────────────────────────────────┘
```
## 十一、关键设计原则总结
| 原则 | 实现方式 |
|------|----------|
| **声明式** | Etcd CR 定义期望状态，Operator 持续调和 |
| **自托管** | etcd 运行在集群 Pod 中，不依赖外部基础设施 |
| **Quorum 优先** | 所有变更操作一次只影响一个 member |
| **渐进式升级** | 滚动升级 + 前后健康检查 + 自动回滚 |
| **可观测性** | 多层 Condition + Metrics + Events |
| **安全默认** | mTLS 双向认证 + 自动证书轮换 |
| **灾备就绪** | 自动定时备份 + 灾难恢复流程 |
| **CVO 协作** | Upgradeable 信号阻塞/放行集群升级 |
     
# OpenShift Etcd Operator 详细设计
## 一、整体架构分解
### 1.1 组件拓扑
```
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenShift Control Plane                       │
│                                                                      │
│   ┌─────────┐                                                        │
│   │   CVO   │  下发 Etcd CR + Operator Deployment                    │
│   └────┬────┘                                                        │
│        │                                                             │
│        ▼                                                             │
│   ┌────────────────────────────────────────────────────────────┐     │
│   │                    Etcd Operator Pod                        │     │
│   │                                                             │     │
│   │  ┌──────────────────┐  ┌───────────────────────────────┐  │     │
│   │  │  Main Controller │  │  Target Reconciler            │  │     │
│   │  │  (Etcd CR Watch) │  │  (per-member 状态机)          │  │     │
│   │  └────────┬─────────┘  └───────────────────────────────┘  │     │
│   │           │                                                  │     │
│   │  ┌────────▼─────────────────────────────────────────────┐  │     │
│   │  │              Sub-Controllers                         │  │     │
│   │  │                                                      │  │     │
│   │  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │  │     │
│   │  │  │  Cert        │  │  StaticPod   │  │  Backup   │ │  │     │
│   │  │  │  Controller  │  │  Controller  │  │  Controller│ │  │     │
│   │  │  └──────────────┘  └──────────────┘  └───────────┘ │  │     │
│   │  │                                                      │  │     │
│   │  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │  │     │
│   │  │  │  Config      │  │  Defrag      │  │  Member   │ │  │     │
│   │  │  │  Controller  │  │  Controller  │  │  Repair   │ │  │     │
│   │  │  └──────────────┘  └──────────────┘  └───────────┘ │  │     │
│   │  └─────────────────────────────────────────────────────┘  │     │
│   │                                                            │     │
│   │  ┌──────────────────────────────────────────────────────┐  │     │
│   │  │              Shared Infrastructure                   │  │     │
│   │  │  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │  │     │
│   │  │  │ Etcd     │  │ Event    │  │ Metrics           │ │  │     │
│   │  │  │ Client   │  │ Recorder │  │ Collector         │ │  │     │
│   │  │  └──────────┘  └──────────┘  └───────────────────┘ │  │     │
│   │  └──────────────────────────────────────────────────────┘  │     │
│   └────────────────────────────────────────────────────────────┘     │
│                                                                      │
│   ┌────────────────────────────────────────────────────────────┐     │
│   │              Target Cluster (Self-Hosted)                  │     │
│   │                                                            │     │
│   │  Master-0               Master-1               Master-2   │     │
│   │  ┌──────────────┐      ┌──────────────┐      ┌──────────┐│     │
│   │  │ etcd Pod     │      │ etcd Pod     │      │ etcd Pod ││     │
│   │  │ ┌──────────┐ │      │ ┌──────────┐ │      │┌────────┐││     │
│   │  │ │  etcd    │ │      │ │  etcd    │ │      ││  etcd  │││     │
│   │  │ └──────────┘ │      │ └──────────┘ │      │└────────┘││     │
│   │  │ ┌──────────┐ │      │ ┌──────────┐ │      │┌────────┐││     │
│   │  │ │ sidecar  │ │      │ │ sidecar  │ │      ││ sidecar│││     │
│   │  │ └──────────┘ │      │ └──────────┘ │      │└────────┘││     │
│   │  └──────────────┘      └──────────────┘      └──────────┘│     │
│   └────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```
### 1.2 分层设计思路
```
┌──────────────────────────────────────────┐
│  Layer 4: Orchestration Layer            │  升级编排、扩缩容编排
│  (Main Controller)                       │  跨 member 协调
├──────────────────────────────────────────┤
│  Layer 3: Lifecycle Layer                │  单 member 生命周期
│  (Target Reconciler + State Machine)     │  状态机驱动
├──────────────────────────────────────────┤
│  Layer 2: Resource Layer                 │  Pod/ConfigMap/Secret
│  (Sub-Controllers)                       │  资源 CRUD
├──────────────────────────────────────────┤
│  Layer 1: Infrastructure Layer           │  etcdctl、API 调用
│  (Etcd Client + Host Operations)         │  底层操作
└──────────────────────────────────────────┘
```
**设计意图**：每层只关心自己的职责，上层依赖下层提供的抽象接口。例如升级编排层不需要知道 Static Pod 怎么创建，只需要调用 Lifecycle 层的"将 member X 升级到版本 Y"接口。
## 二、Main Controller 详细设计
### 2.1 Reconcile 触发机制
```
触发源:
  ├── Etcd CR 变更事件 (Spec 变化)
  ├── 定时触发 (健康检查周期)
  ├── 子资源变更回调 (Pod 状态变化)
  └── 外部事件 (CVO 升级通知)
```
### 2.2 Reconcile 主流程（伪代码逻辑）
```
func Reconcile(ctx, etcdCR) (Result, error):

    // ===== Phase 0: 输入验证 =====
    if etcdCR.Spec.ManagementState == Unmanaged:
        return RequeueAfter(60s)  // 仅周期性上报 Status

    // ===== Phase 1: 集群发现 =====
    members = discoverEtcdMembers()
    // 从 API Server 获取所有 etcd Pod
    // 从 etcdctl member list 获取实际成员
    // 两者交叉验证

    if members == nil:
        setStatus(Degraded, "无法发现 etcd 成员")
        return RequeueAfter(5s)

    // ===== Phase 2: 前置检查 =====
    preCheckResult = runPreFlightChecks(members)
    if preCheckResult.HasBlockingIssues:
        setStatus(Degraded, preCheckResult.Message)
        return RequeueAfter(10s)

    // ===== Phase 3: 证书同步 =====
    if certsNeedRotation(members):
        syncCertificates()
        // 证书轮换可能触发 Pod 重启
        return RequeueAfter(30s)

    // ===== Phase 4: 配置同步 =====
    syncConfigMaps()
    syncRBAC()

    // ===== Phase 5: 核心生命周期管理 =====
    switch determineDesiredAction(etcdCR, members):
        case NoChange:
            // 无需变更，进入健康检查
            break

        case VersionUpgrade:
            // 编排滚动升级
            result = orchestrateRollingUpgrade(etcdCR, members)
            if result.NeedRequeue:
                return RequeueAfter(result.Delay)
            if result.Error != nil:
                setStatus(Degraded, result.Error)
                return RequeueAfter(10s)

        case MemberAdd:
            // 添加新成员
            result = addNewMember(etcdCR, members)

        case MemberRemove:
            // 移除成员
            result = removeMember(etcdCR, members)

        case ConfigChange:
            // 配置变更（如资源限制、参数调整）
            result = applyConfigChange(etcdCR, members)

    // ===== Phase 6: 健康检查与 Status 更新 =====
    healthResult = runHealthChecks(members)
    updateStatus(etcdCR, healthResult)

    // ===== Phase 7: 备份检查 =====
    if backupOverdue(etcdCR):
        triggerBackup()

    return RequeueAfter(30s)  // 常规轮询间隔
```
### 2.3 Action 判定逻辑
```
func determineDesiredAction(etcdCR, members) -> Action:

    desiredVersion = etcdCR.Status.DesiredVersion  // 由 CVO 设置
    currentVersion = getLowestMemberVersion(members)

    // 1. 版本不一致 → 升级
    if desiredVersion != currentVersion:
        return VersionUpgrade

    // 2. Spec 中定义的成员数与实际不一致 → 扩缩容
    desiredMemberCount = getDesiredMemberCount(etcdCR)
    actualMemberCount = len(members.Healthy)
    if actualMemberCount < desiredMemberCount:
        return MemberAdd
    if actualMemberCount > desiredMemberCount:
        return MemberRemove

    // 3. Pod 配置与期望不一致 → 配置变更
    for member in members:
        if podSpecDiffers(member.Pod, expectedPodSpec(etcdCR)):
            return ConfigChange

    return NoChange
```
## 三、Member 状态机设计
### 3.1 状态定义
```
┌──────────────────────────────────────────────────────────────────┐
│                   Member State Machine                            │
│                                                                   │
│                        ┌──────────┐                               │
│                        │  New     │                               │
│                        └────┬─────┘                               │
│                             │ 加入集群                             │
│                             ▼                                     │
│   ┌──────────┐      ┌──────────┐      ┌──────────────┐          │
│   │ Removing │◀─────│ Running  │─────▶│ Upgrading    │          │
│   └────┬─────┘      └──┬──┬────┘      └──────┬───────┘          │
│        │               │  │                    │                  │
│        │               │  │ 健康检查失败        │ 升级完成         │
│        │               │  ▼                    │                  │
│        │          ┌────┴────────┐              │                  │
│        │          │ Degraded    │──────────────┘                  │
│        │          └──────┬──────┘  自愈成功                       │
│        │                 │                                        │
│        │                 │ 自愈失败                                │
│        │                 ▼                                        │
│        │          ┌──────────────┐                                │
│        │          │ Repairing    │                                │
│        │          └──────┬───────┘                                │
│        │                 │                                        │
│        ▼                 ▼                                        │
│   ┌──────────┐    ┌──────────────┐                               │
│   │ Removed  │    │ Unreachable  │                               │
│   └──────────┘    └──────────────┘                               │
└──────────────────────────────────────────────────────────────────┘
```
### 3.2 状态转换表
| 当前状态 | 事件 | 目标状态 | 动作 |
|----------|------|----------|------|
| New | 加入集群命令 | Running | 创建 Static Pod + 等待 member 加入 |
| Running | 版本升级触发 | Upgrading | 更新 Pod Manifest |
| Running | 健康检查失败 | Degraded | 记录降级原因 |
| Running | 移除命令 | Removing | 执行 member remove |
| Upgrading | Pod Ready + 版本匹配 | Running | 升级完成 |
| Upgrading | 超时/健康检查失败 | Degraded | 标记降级 |
| Degraded | 自愈成功 | Running | 清除降级标记 |
| Degraded | 自愈触发 | Repairing | 执行修复操作 |
| Repairing | 修复成功 | Running | 恢复正常 |
| Repairing | 修复失败 | Unreachable | 标记不可达 |
| Removing | member 移除完成 | Removed | 清理资源 |
| Unreachable | 重新可达 | Degraded | 进入自愈评估 |
### 3.3 Target Reconciler 设计
每个 etcd member 对应一个独立的 Target Reconciler 实例，负责驱动该 member 的状态机：
```
type TargetReconciler struct {
    MemberID       string
    MemberName     string
    CurrentState   MemberState
    LastHealthTime time.Time

    // 依赖接口
    PodController   PodControllerInterface
    EtcdClient      EtcdClientInterface
    CertController  CertControllerInterface
}

func (r *TargetReconciler) Reconcile(ctx, desiredState) error:
    switch r.CurrentState:
        case Running:
            return r.reconcileRunning(ctx, desiredState)
        case Upgrading:
            return r.reconcileUpgrading(ctx, desiredState)
        case Degraded:
            return r.reconcileDegraded(ctx, desiredState)
        case Repairing:
            return r.reconcileRepairing(ctx, desiredState)
        // ...
```
**设计意图**：将每个 member 的状态管理独立化，避免一个 member 的问题影响其他 member 的调和。Main Controller 通过聚合所有 Target Reconciler 的状态来决定集群级别的操作。
## 四、滚动升级详细设计
### 4.1 升级编排器
```
type UpgradeOrchestrator struct {
    TargetVersion    string
    UpgradeStrategy  UpgradeStrategy
    Members          []MemberInfo
    UpgradeOrder     []int       // 升级顺序（member 索引）
    CurrentIndex     int         // 当前正在升级的 member 索引
    Phase            UpgradePhase
    StartTime        time.Time
    PreBackupDone    bool
}

type UpgradePhase string
const (
    UpgradePhasePreCheck    UpgradePhase = "PreCheck"
    UpgradePhaseBackup      UpgradePhase = "Backup"
    UpgradePhaseUpgrading   UpgradePhase = "Upgrading"
    UpgradePhasePostCheck   UpgradePhase = "PostCheck"
    UpgradePhaseCompleted   UpgradePhase = "Completed"
    UpgradePhaseRollback    UpgradePhase = "Rollback"
)
```
### 4.2 升级编排流程
```
func orchestrateRollingUpgrade(etcdCR, members) -> Result:

    orchestrator = loadOrResumeOrchestrator(etcdCR)

    switch orchestrator.Phase:

    // ===== PreCheck 阶段 =====
    case PreCheck:
        checks = [
            checkQuorum(members),           // 有法定人数
            checkAllMembersHealthy(members), // 所有成员健康
            checkDiskSpace(members),         // 磁盘空间充足
            checkNetworkConnectivity(members), // 网络互通
            checkVersionCompatibility(current, target), // 版本兼容
        ]
        if allPassed(checks):
            orchestrator.Phase = Backup
        else:
            setStatus(Upgradeable=False, checks.Failures)
            return Requeue(30s)

    // ===== Backup 阶段 =====
    case Backup:
        if !orchestrator.PreBackupDone:
            backupResult = performPreUpgradeBackup()
            if backupResult.Error != nil:
                // 备份失败不阻塞升级，但记录告警
                log.Warn("pre-upgrade backup failed: %v", backupResult.Error)
            orchestrator.PreBackupDone = true

        // 确定升级顺序
        orchestrator.UpgradeOrder = determineUpgradeOrder(members)
        // 策略: Leader 最后升级，减少 leader 切换次数
        orchestrator.CurrentIndex = 0
        orchestrator.Phase = Upgrading

    // ===== Upgrading 阶段 =====
    case Upgrading:
        if orchestrator.CurrentIndex >= len(orchestrator.UpgradeOrder):
            orchestrator.Phase = PostCheck
            return Continue

        targetMember = members[orchestrator.UpgradeOrder[orchestrator.CurrentIndex]]

        // 检查当前集群是否健康（每个 member 升级前都检查）
        if !isClusterHealthy(members):
            log.Warn("cluster not healthy before upgrading member %s, waiting", targetMember.Name)
            return Requeue(10s)

        // 执行单个 member 升级
        result = upgradeSingleMember(targetMember, orchestrator.TargetVersion)

        switch result:
            case Success:
                orchestrator.CurrentIndex++
                return Requeue(5s)  // 短暂等待后升级下一个

            case InProgress:
                return Requeue(10s)  // 等待 Pod 就绪

            case Failed:
                orchestrator.Phase = Rollback
                orchestrator.RollbackTarget = targetMember
                return Continue

    // ===== PostCheck 阶段 =====
    case PostCheck:
        // 最终全量健康检查
        healthResult = runFullHealthCheck(members)
        if healthResult.AllHealthy:
            orchestrator.Phase = Completed
            setStatus(Progressing=False, currentVersion=target)
        else:
            setStatus(Degraded, healthResult.Issues)
            return Requeue(30s)

    // ===== Rollback 阶段 =====
    case Rollback:
        // 将失败的 member 回退到旧版本
        rollbackResult = rollbackMember(orchestrator.RollbackTarget, previousVersion)
        if rollbackResult.Success:
            setStatus(Degraded, "upgrade rolled back due to member failure")
        else:
            setStatus(Degraded, "rollback also failed, manual intervention required")
        return Requeue(30s)
```
### 4.3 升级顺序策略
```
func determineUpgradeOrder(members) -> []int:

    leaderID = findLeader(members)

    // 基本策略: Leader 最后升级
    // 原因: Leader 切换会导致短暂的写不可用
    //       先升级 Follower 可以在升级过程中保持 Leader 稳定

    followers = members.Filter(m => m.ID != leaderID && m.Healthy)
    leader = members.Find(m => m.ID == leaderID)

    order = followers.SortedBy(healthScore)  // 健康的先升级
    order.Append(leader)

    return order
```
### 4.4 单 Member 升级流程
```
func upgradeSingleMember(member, targetVersion) -> UpgradeResult:

    // Step 1: 确认 member 当前状态
    if member.Version == targetVersion:
        return Success  // 已经是目标版本

    if !member.Healthy:
        return Failed("member not healthy, cannot upgrade")

    // Step 2: 记录升级前状态（用于回滚）
    previousManifest = readCurrentManifest(member)
    storeRollbackInfo(member, previousManifest)

    // Step 3: 更新 Static Pod Manifest
    newManifest = renderManifest(member, targetVersion)
    writeManifest(member.Node, newManifest)

    // Step 4: 等待 Pod 重建
    podReady = waitForPodReady(member, timeout=5min)
    if !podReady:
        return Failed("pod not ready after manifest update")

    // Step 5: 等待 etcd member 恢复
    memberHealthy = waitForMemberHealthy(member, timeout=3min)
    if !memberHealthy:
        return Failed("etcd member not healthy after pod restart")

    // Step 6: 验证版本
    actualVersion = getMemberVersion(member)
    if actualVersion != targetVersion:
        return Failed("version mismatch after upgrade")

    // Step 7: 验证集群一致性
    if !isClusterConsistent():
        return Failed("cluster inconsistency detected after upgrade")

    return Success
```
## 五、证书管理详细设计
### 5.1 证书体系
```
┌─────────────────────────────────────────────────────────────────┐
│                    Certificate Hierarchy                         │
│                                                                  │
│                    etcd-ca (Root CA)                             │
│                   ┌──────┴───────┐                               │
│                   │              │                               │
│            etcd-server-ca   etcd-peer-ca                        │
│            ┌─────┴─────┐   ┌─────┴─────┐                       │
│            │           │   │           │                       │
│     server.crt   server-   peer.crt   peer-                    │
│     (serving)    client-   (peer      client-                  │
│                  etcd)     comm)      apiserver-                │
│                                       etcd-client              │
│                                                                 │
│  Secret 布局:                                                    │
│    etcd-signer                   → Root CA key+cert             │
│    etcd-serving-<member>         → Server cert per member       │
│    etcd-peer-<member>            → Peer cert per member         │
│    etcd-all-bundle               → CA bundle (trust all)        │
│    kube-apiserver-etcd-client    → API Server client cert       │
└─────────────────────────────────────────────────────────────────┘
```
### 5.2 证书轮换状态机
```
┌──────────────────────────────────────────────────────────────┐
│              Certificate Rotation State Machine              │
│                                                              │
│  ┌────────────┐   检测到即将过期     ┌────────────┐           │
│  │  Stable    │───────────────────▶│  Rotating  │           │
│  │  (正常)    │                     │  (轮换中)   │           │
│  └────────────┘                     └──────┬─────┘           │
│       ▲                                  │                   │
│       │                                  │                   │
│       │  轮换完成                         ▼                   │
│  ┌────────────┐                    ┌────────────┐           │
│  │  Verifying │◀──────────────────│  Rolling   │           │
│  │  (验证中)  │   Pod 滚动完成      │  Pods      │           │
│  └────────────┘                    └────────────┘           │
│                                                              │
│  CA 轮换特殊流程:                                             │
│  ┌────────────┐    ┌──────────────┐   ┌──────────────┐       │
│  │ TrustBoth  │──▶│ TrustNewOnly │──▶│  Stable      │       │
│  │ (信任新旧) │    │ (仅信新CA)   │    │ (完成)        │       │
│  └────────────┘   └──────────────┘    └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
```
### 5.3 证书轮换编排
```
func rotateCertificates(members) -> Result:

    // Phase 1: 生成新证书
    for member in members:
        newCert = generateCert(member, validity=365d)
        writeSecret("etcd-serving-" + member.Name + "-new", newCert)

    // Phase 2: 更新 CA Bundle（信任新旧 CA）
    appendNewCAToBundle()

    // Phase 3: 滚动更新 etcd Pod（加载新证书）
    for member in members:  // 逐个，保证 quorum
        updatePodManifest(member, mountNewCert=true)
        waitForMemberHealthy(member)

    // Phase 4: 更新 API Server 客户端证书
    updateAPIServerEtcdClientCert()

    // Phase 5: 移除旧 CA 信任（仅信任新 CA）
    removeOldCAFromBundle()

    // Phase 6: 再次滚动更新 etcd Pod
    for member in members:
        updatePodManifest(member, mountNewCertOnly=true)
        waitForMemberHealthy(member)

    // Phase 7: 清理旧 Secret
    cleanupOldCertSecrets()

    return Success
```
**设计意图**：CA 轮换采用两阶段信任模型——先让所有组件同时信任新旧 CA，确保滚动过程中不会出现证书验证失败；待所有组件都使用新 CA 签发的证书后，再移除旧 CA 信任。
## 六、备份与恢复详细设计
### 6.1 备份控制器
```
type BackupController struct {
    Schedule         time.Duration   // 备份间隔
    MaxBackups       int             // 最大保留数量
    StorageBackend   StorageBackend  // 存储后端
    LastBackupTime   time.Time
    LastBackupStatus BackupStatus
}

type StorageBackend interface {
    Save(ctx, snapshot io.Reader, metadata BackupMetadata) error
    List(ctx) ([]BackupMetadata, error)
    Load(ctx, backupID string) (io.ReadCloser, error)
    Delete(ctx, backupID string) error
}

// 存储后端实现:
//   - PVCStorageBackend: 写入专用 PVC
//   - S3StorageBackend:  上传到对象存储
//   - HostPathStorageBackend: 写入宿主机路径
```
### 6.2 备份执行流程
```
func performBackup(members) -> BackupResult:

    // Step 1: 选择备份源 member
    backupMember = selectBackupMember(members)
    // 策略: 优先选择 Follower（避免影响 Leader 性能）
    //        选择延迟最低、DB 最小的 member

    // Step 2: 执行快照
    snapshotResult = etcdctl.SnapshotSave(backupMember, snapshotPath)
    // 命令: etcdctl --endpoints=<member> snapshot save <path>
    // 注意: v3.5+ 支持在线快照，不影响服务

    // Step 3: 校验快照完整性
    statusResult = etcdctl.SnapshotStatus(snapshotPath)
    // 命令: etcdctl snapshot status <path> --write-out=json
    // 检查: revision > 0, hash 合法

    // Step 4: 计算快照元数据
    metadata = BackupMetadata{
        ID:            generateBackupID(),
        Timestamp:     time.Now(),
        Revision:      statusResult.Revision,
        TotalKeys:     statusResult.TotalKeys,
        SnapshotSize:  statusResult.Size,
        EtcdVersion:   backupMember.Version,
        MemberCount:   len(members),
        SourceMember:   backupMember.Name,
    }

    // Step 5: 持久化到存储后端
    storage.Save(snapshotFile, metadata)

    // Step 6: 清理过期备份
    cleanupOldBackups(maxBackups)

    // Step 7: 更新 Etcd CR Status
    updateBackupStatus(metadata)

    return BackupResult{Success: true, Metadata: metadata}
```
### 6.3 灾难恢复流程
```
func disasterRecovery(backupID string, members) -> RecoveryResult:

    // ===== 场景判定 =====
    healthyCount = countHealthyMembers(members)
    quorum = len(members) / 2 + 1

    if healthyCount >= quorum:
        // 场景A: 仍有 quorum → 在线修复
        return repairWithQuorum(members)

    // 场景B: 失去 quorum → 从备份恢复
    return restoreFromBackup(backupID, members)

func restoreFromBackup(backupID string, members) -> RecoveryResult:

    // Step 1: 停止所有 etcd Pod
    for member in members:
        removeManifest(member)  // 删除 Static Pod Manifest
        waitForPodStopped(member)

    // Step 2: 从备份恢复数据
    snapshot = storage.Load(backupID)
    for i, member in members:
        // 清空旧数据
        removeDataDir(member)

        // 恢复快照到每个 member 的数据目录
        etcdctl.SnapshotRestore(snapshot, {
            DataDir:       member.DataDir,
            Name:          member.Name,
            InitialCluster: buildInitialClusterString(members),
            InitialAdvertisePeerURLs: member.PeerURLs,
        })

    // Step 3: 启动第一个 member
    writeManifest(members[0], initialClusterState="new")
    waitForMemberReady(members[0])

    // Step 4: 逐个启动其余 member
    for member in members[1:]:
        writeManifest(member, initialClusterState="existing")
        waitForMemberReady(member)

    // Step 5: 验证集群健康
    if isClusterHealthy(members):
        return RecoveryResult{Success: true}
    else:
        return RecoveryResult{Success: false, Error: "cluster unhealthy after restore"}
```
## 七、健康监控系统详细设计
### 7.1 健康检查体系
```
┌─────────────────────────────────────────────────────────────────┐
│                  Health Check Architecture                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Health Aggregator                       │    │
│  │  (聚合所有检查结果，计算集群级健康状态)                    │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         │                │                │                     │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐           │
│  │  Member     │  │  Cluster    │  │  Infra      │           │
│  │  Checks     │  │  Checks     │  │  Checks     │           │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │
│         │                │                │                  │
│    ┌────┴────┐     ┌────┴────┐     ┌────┴────┐               │
│    │Pod Alive│     │Quorum   │     │Disk     │               │
│    │Endpoint │     │Leader   │     │Network  │               │
│    │Alarm    │     │Revision │     │IO Latency│              │
│    │DB Size  │     │Version  │     │Memory   │               │
│    │Latency  │     │Consist. │     │CPU      │               │
│    └─────────┘     └─────────┘     └─────────┘               │
└──────────────────────────────────────────────────────────────┘
```
### 7.2 检查项详细定义
```
type HealthCheck interface {
    Name() string
    Execute(ctx, member) HealthCheckResult
    Severity() Severity  // Critical | Warning | Info
}

// ===== Member 级别检查 =====
type PodAliveCheck struct{}       // Pod 是否 Running + Ready
type EndpointHealthCheck struct{}  // etcdctl endpoint health
type AlarmCheck struct{}          // etcdctl alarm list (NOSPACE 等)
type DBSizeCheck struct{}         // DB 文件大小是否超阈值
type LatencyCheck struct{}        // 读写延迟是否超阈值
type RevisionLagCheck struct{}    // 该 member 的 revision 与 leader 的差距

// ===== Cluster 级别检查 =====
type QuorumCheck struct{}         // 是否有 N/2+1 个 member 健康
type LeaderCheck struct{}         // 是否存在 Leader
type RevisionConsistencyCheck struct{}  // 所有 member revision 是否接近
type VersionConsistencyCheck struct{}   // 所有 member 版本是否一致

// ===== Infrastructure 级别检查 =====
type DiskSpaceCheck struct{}      // 磁盘可用空间
type DiskIOCheck struct{}         // 磁盘 IO 延迟
type NetworkPartitionCheck struct{}  // 网络分区检测
type MemoryCheck struct{}         // 内存使用率
```
### 7.3 健康状态聚合
```
func aggregateHealth(memberResults) -> ClusterHealthStatus:

    status = ClusterHealthStatus{
        Available:  True,   // 默认可用
        Degraded:   False,  // 默认未降级
        Upgradeable: True,  // 默认可升级
    }

    for _, result := range memberResults:
        // Critical 级别问题 → 集群不可用
        if result.Severity == Critical:
            if affectsQuorum(result):
                status.Available = False
            status.Degraded = True
            status.Upgradeable = False

        // Warning 级别问题 → 降级但可用
        if result.Severity == Warning:
            status.Degraded = True
            // 降级状态不一定阻塞升级，取决于具体问题
            if blocksUpgrade(result):
                status.Upgradeable = False

    return status
```
### 7.4 自愈策略表
```
┌──────────────────────────────────────────────────────────────────┐
│                      Self-Healing Strategy Table                 │
│                                                                   │
│  检测到的问题          │ 自愈动作                │ 触发条件       │
│  ─────────────────────────────────────────────────────────────── │
│  Pod 崩溃             │ kubelet 自动重启         │ Pod 非 Running │
│  Pod 反复崩溃          │ 标记 Degraded + 告警     │ RestartCount>5 │
│  Member 未加入集群     │ 重新写入 Manifest        │ member list 缺失│
│  DB 空间告警(NOSPACE) │ 触发 Defrag + 告警       │ alarm 存在     │
│  DB 空间告警持续       │ 标记 Degraded            │ alarm > 10min  │
│  Member 延迟过高       │ 标记 Degraded            │ latency > 100ms│
│  Member Revision 落后  │ 等待自动追赶 + 告警      │ lag > 10000    │
│  无 Leader            │ 等待选举 + 超时告警       │ 30s 无 Leader  │
│  证书即将过期          │ 自动轮换证书             │ < 80% 生命周期 │
│  磁盘空间不足          │ 触发 Defrag + 告警       │ 可用 < 10%     │
│  磁盘 IO 延迟高        │ 标记 Degraded            │ p99 > 100ms    │
│  网络分区              │ 等待恢复 + 标记 Degraded │ member 不可达  │
└──────────────────────────────────────────────────────────────────┘
```
## 八、配置管理详细设计
### 8.1 配置层级
```
┌──────────────────────────────────────────────────────────────┐
│                   Configuration Hierarchy                    │
│                                                               │
│  Layer 1: CVO 管理清单 (不可修改)                            │
│    └── 默认 etcd 配置参数、资源限制                           │
│                                                               │
│  Layer 2: Etcd CR Spec.observedConfig (可观测配置)           │
│    └── 集群特定的配置调整（由 Operator 自动写入）             │
│                                                               │
│  Layer 3: Etcd CR Spec.unsupportedConfigOverrides (紧急覆盖) │
│    └── 紧急情况下的配置覆盖（不保证兼容性）                   │
│                                                               │
│  合并策略: Layer3 > Layer2 > Layer1                          │
│  任何 Layer3 的使用都会在 Status 中标记警告                   │
└──────────────────────────────────────────────────────────────┘
```
### 8.2 配置同步流程
```
func syncConfigMaps(etcdCR, members):

    // 1. 计算最终配置
    baseConfig = getDefaultConfig()                    // Layer 1
    observedConfig = etcdCR.Spec.ObservedConfig        // Layer 2
    overrideConfig = etcdCR.Spec.UnsupportedConfigOverrides  // Layer 3

    finalConfig = merge(baseConfig, observedConfig, overrideConfig)

    // 2. 渲染 ConfigMap
    configMap = renderConfigMap("etcd-config", finalConfig)

    // 3. 对比并更新
    currentConfigMap = getConfigMap("etcd-config")
    if !deepEqual(currentConfigMap.Data, configMap.Data):
        updateConfigMap(configMap)
        // 配置变更可能需要滚动重启 etcd Pod
        if needsPodRestart(currentConfigMap, configMap):
            scheduleRollingRestart(members)
```
## 九、Static Pod 控制器详细设计
### 9.1 Manifest 管理
```
type StaticPodController struct {
    ManifestDir  string    // /etc/kubernetes/manifests/
    TemplateDir  string    // Operator 内置模板
}

func (c *StaticPodController) SyncMemberPod(member, spec) error:

    // 1. 渲染期望的 Pod Manifest
    desiredPod = renderPodManifest(member, spec)

    // 2. 读取当前 Manifest
    currentPod = readManifest(c.ManifestDir, member.Name)

    // 3. 计算差异
    diff = computeDiff(currentPod, desiredPod)

    if diff == nil:
        return nil  // 无变更

    // 4. 分类差异
    if isOnlyImageChange(diff):
        // 仅镜像变更 → 直接写入，kubelet 会滚动重启
        writeManifest(c.ManifestDir, member.Name, desiredPod)
    else:
        // 其他变更（参数、卷等）→ 需要更谨慎
        // 先验证 Manifest 合法性
        if err := validateManifest(desiredPod); err != nil:
            return err
        writeManifest(c.ManifestDir, member.Name, desiredPod)

    // 5. 等待 Pod 状态变化
    return waitForPodTransition(member)
```
### 9.2 Sidecar 容器设计
```
┌──────────────────────────────────────────────────────────────┐
│                  etcd Pod 容器布局                             │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  etcd (主容器)                                       │    │
│  │  - etcd server 进程                                  │    │
│  │  - 暴露 2379(client) + 2380(peer) 端口               │    │
│  │  - 挂载数据目录 + 证书目录                            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  etcd-metrics (Sidecar 1)                            │    │
│  │  - 暴露 /metrics 端点 (Prometheus 格式)              │    │
│  │  - 代理 etcd v2/v3 metrics                           │    │
│  │  - 健康检查探针                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  etcd-health-monitor (Sidecar 2)                     │    │
│  │  - 周期性执行 etcdctl endpoint health                │    │
│  │  - 检测 NOSPACE alarm                                │    │
│  │  - 检测 member 状态变化                               │    │
│  │  - 将结果写入共享文件（供 Operator 读取）             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  etcd-backup (Sidecar 3, 可选)                       │    │
│  │  - 按计划执行 etcdctl snapshot save                  │    │
│  │  - 上传到备份存储                                    │    │
│  │  - 清理过期快照                                      │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```
## 十、Defrag 控制器详细设计
```
type DefragController struct {
    DBSizeThreshold    int64         // 触发 defrag 的 DB 大小阈值
    FragmentRatio      float64       // 碎片率阈值
    Schedule           time.Duration // 检查间隔
    LastDefragTime     map[string]time.Time  // per-member 上次 defrag 时间
}

func (c *DefragController) Reconcile(members):

    for member in members:
        // 1. 获取 DB 状态
        status = etcdctl.EndpointStatus(member)

        // 2. 判断是否需要 defrag
        dbSize = status.DBSize
        dbSizeInUse = status.DBSizeInUse
        fragmentRatio = float64(dbSize-dbSizeInUse) / float64(dbSize)

        needsDefrag = dbSize > c.DBSizeThreshold &&
                      fragmentRatio > c.FragmentRatio &&
                      time.Since(c.LastDefragTime[member.Name]) > 1h

        if !needsDefrag:
            continue

        // 3. 执行 defrag（一次一个 member）
        if !isClusterHealthy(members):
            log.Warn("skip defrag for %s, cluster not healthy", member.Name)
            continue

        result = etcdctl.Defrag(member)
        if result.Error != nil:
            log.Error("defrag failed for %s: %v", member.Name, result.Error)
            continue

        c.LastDefragTime[member.Name] = time.Now()

        // 4. defrag 后等待 member 恢复
        waitForMemberHealthy(member, timeout=2min)

        // 5. 检查 DB 大小变化
        newStatus = etcdctl.EndpointStatus(member)
        log.Info("defrag %s: %d -> %d bytes", member.Name, dbSize, newStatus.DBSize)
```
## 十一、Member 修复控制器详细设计
```
type MemberRepairController struct {
    MaxRepairAttempts int
    RepairBackoff     time.Duration
}

func (c *MemberRepairController) Repair(member) -> RepairResult:

    // 修复策略按严重程度递进:

    // Level 1: 重启 Pod（最轻量）
    if member.PodNotRunning:
        result = restartPod(member)
        if result.Success:
            return result

    // Level 2: 重新写入 Manifest
    if member.ManifestDrifted:
        result = rewriteManifest(member)
        if result.Success:
            return result

    // Level 3: 清理数据目录 + 从其他 member 同步
    if member.DataCorrupted:
        // 3a. 从集群中移除 member
        etcdctl.MemberRemove(member.ID)
        // 3b. 清理本地数据
        removeDataDir(member)
        // 3c. 重新添加 member（自动从其他 member 同步数据）
        etcdctl.MemberAdd(member.Name, member.PeerURLs)
        // 3d: 写入 Manifest 启动 Pod
        writeManifest(member, initialClusterState="existing")
        // 3e: 等待数据同步完成
        waitForDataSync(member)
        if isMemberHealthy(member):
            return RepairResult{Success: true}

    // Level 4: 完全重建（最激进）
    // 仅在数据严重损坏且 Level 3 失败时使用
    if member.Unrecoverable:
        // 停止 Pod
        removeManifest(member)
        // 从集群移除
        etcdctl.MemberRemove(member.ID)
        // 清理数据
        removeDataDir(member)
        // 重新添加
        etcdctl.MemberAdd(member.Name, member.PeerURLs)
        // 以空数据目录启动
        writeManifest(member, initialClusterState="existing")
        waitForMemberHealthy(member)
        if isMemberHealthy(member):
            return RepairResult{Success: true}

    return RepairResult{Success: false, Error: "all repair attempts failed"}
```
## 十二、可观测性设计
### 12.1 Metrics 指标
```
┌──────────────────────────────────────────────────────────────┐
│                     Metrics 指标体系                          │
│                                                               │
│  集群级别:                                                    │
│    etcd_cluster_members_total          成员总数               │
│    etcd_cluster_members_healthy        健康成员数             │
│    etcd_cluster_has_quorum             是否有法定人数         │
│    etcd_cluster_version                集群版本               │
│    etcd_cluster_upgrade_phase          升级阶段               │
│                                                               │
│  Member 级别:                                                 │
│    etcd_member_health_status           健康状态 (0/1/2)      │
│    etcd_member_db_size_bytes           DB 大小                │
│    etcd_member_revision                当前 revision          │
│    etcd_member_latency_seconds         读写延迟               │
│    etcd_member_version                 成员版本               │
│                                                               │
│  Operator 级别:                                               │
│    etcd_operator_reconcile_total       Reconcile 总次数       │
│    etcd_operator_reconcile_errors      Reconcile 错误次数     │
│    etcd_operator_upgrade_progress      升级进度               │
│    etcd_operator_backup_last_timestamp 最后备份时间           │
│    etcd_operator_cert_expiry_seconds   证书过期倒计时         │
└──────────────────────────────────────────────────────────────┘
```
### 12.2 Event 记录
```
关键 Event:
  Normal  :
    - MemberUpgraded       成员升级完成
    - BackupCompleted      备份完成
    - DefragCompleted      Defrag 完成
    - CertRotated          证书轮换完成
    - MemberRepaired       成员修复成功

  Warning  :
    - MemberDegraded       成员降级
    - BackupFailed         备份失败
    - DefragFailed         Defrag 失败
    - CertExpiringSoon     证书即将过期
    - DiskSpaceLow         磁盘空间不足
    - UpgradeBlocked       升级被阻塞

  Error    :
    - QuorumLost           失去法定人数
    - MemberUnreachable    成员不可达
    - RestoreFailed        恢复失败
```
## 十三、错误处理与恢复策略
### 13.1 错误分级
```
┌──────────────────────────────────────────────────────────────┐
│                    Error Severity Levels                      │
│                                                               │
│  Level 1 - Transient (瞬态错误)                              │
│    特征: 可能自动恢复                                         │
│    示例: 短暂网络抖动、Pod 临时重启                           │
│    策略: Requeue + 指数退避                                   │
│                                                               │
│  Level 2 - Persistent (持续错误)                             │
│    特征: 不会自动恢复，但集群仍可用                           │
│    示例: 单个 member 磁盘故障、证书过期                       │
│    策略: 标记 Degraded + 触发自愈 + 告警                     │
│                                                               │
│  Level 3 - Critical (严重错误)                               │
│    特征: 影响集群可用性                                       │
│    示例: 失去 quorum、多数 member 不可达                      │
│    策略: 标记 Unavailable + 阻塞升级 + 人工介入              │
│                                                               │
│  Level 4 - Catastrophic (灾难性错误)                         │
│    特征: 数据丢失风险                                         │
│    示例: 数据损坏、全部 member 不可达                         │
│    策略: 触发灾难恢复流程 + 人工介入                          │
└──────────────────────────────────────────────────────────────┘
```
### 13.2 Reconcile 错误处理
```
func safeReconcile(ctx, etcdCR) -> Result:

    result, err := reconcile(ctx, etcdCR)

    if err == nil:
        return result

    // 错误分类
    severity := classifyError(err)

    switch severity:
        case Transient:
            // 瞬态错误: 短暂 Requeue
            return RequeueAfter(5s)

        case Persistent:
            // 持续错误: 标记降级，较长 Requeue
            setStatus(Degraded, err.Error())
            return RequeueAfter(30s)

        case Critical:
            // 严重错误: 标记不可用，阻塞升级
            setStatus(Unavailable, err.Error())
            setStatus(Upgradeable=False)
            return RequeueAfter(60s)

        case Catastrophic:
            // 灾难性错误: 触发恢复流程
            setStatus(Unavailable, err.Error())
            setStatus(Upgradeable=False)
            triggerDisasterRecovery()
            return RequeueAfter(120s)
```
## 十四、关键设计决策总结
| 决策点 | 选择 | 理由 |
|--------|------|------|
| etcd 运行方式 | Static Pod | 不依赖 API Server 可用性，kubelet 直接管理 |
| 升级策略 | 逐成员滚动 | 保证 quorum，最小化不可用窗口 |
| 升级顺序 | Leader 最后 | 减少 Leader 切换次数，降低写中断 |
| 证书轮换 | 两阶段信任 | 避免滚动过程中证书验证失败 |
| 备份方式 | 在线快照 | v3.5+ 支持不影响服务的快照 |
| 自愈策略 | 渐进式修复 | 从轻量操作到重量操作递进，最小化影响 |
| 配置管理 | 三层合并 | 兼顾默认安全 + 可观测调整 + 紧急覆盖 |
| 状态管理 | CR Status + Condition | 声明式，与 CVO 生态一致 |
| 健康检查 | 多层分级 | Pod → Member → Cluster → Infra 逐层深入 |
| 错误处理 | 四级分类 | 不同严重程度不同处理策略 |
        
