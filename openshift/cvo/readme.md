# CVO管理的Operator
**结论：CVO 通过下发 release payload 来协调并升级集群的“核心 Operators”（Cluster Operators），这些包括认证、网络、镜像注册表、监控、存储、机器/控制平面相关等；具体清单随 OpenShift 版本与 payload 而异。**   [docs.redhat.com](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/operators/index)
## 核心说明
- **CVO 的职责**：读取 `ClusterVersion`，拉取 release payload（包含一组 Operator manifests/CSV），按 runlevel 顺序应用并监控各 Operator 状态以推进升级。**CVO 本身不单独实现每个功能，而是下发并协调这些 Operator。**   [docs.redhat.com](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/operators/index)  
- **版本差异**：不同 OpenShift 版本或发行版的 payload 会包含不同的 Operators，下面列举常见且由 CVO 管理的“平台级” Operators（非穷尽）。
## 常见由 CVO 管理的 Operators（示例）
- **authentication**（认证）  
- **cloud-credential**（云凭证）  
- **cluster-autoscaler**（自动扩缩）  
- **cluster-config-operator**（集群配置）  
- **cluster-image-registry**（镜像注册表）  
- **cluster-monitoring / monitoring**（监控）  
- **cluster-network-operator / network**（网络）  
- **cluster-storage-operator / storage / csi 驱动相关**（存储）  
- **cluster-version-operator (CVO)**（自身）  
- **console**（Web 控制台）  
- **kube-apiserver / kube-controller-manager / kube-scheduler**（Kubernetes 核心 Operator 封装）  
- **machine-api / machine-approver / machine-config-operator (MCO)**（机器与不可变 OS 管理）  
（以上为常见项，实际以 `oc get clusteroperator` 返回为准。）

[docs.redhat.com](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/operators/index) 
## list
- Cluster Version Operator (CVO)：负责集群版本升级与 Payload 应用。
- Authentication Operator：管理 OAuth、身份认证、用户登录。
- Cloud Credential Operator：管理云平台凭证（AWS、Azure、GCP 等）。
- Cluster Image Registry Operator：提供内置镜像仓库。
- Cluster Monitoring Operator：部署 Prometheus、Alertmanager、Grafana 等监控组件。
- Cluster Network Operator：管理集群网络（SDN、CNI 插件）。
- Cluster Storage Operator：管理存储插件与持久卷。
- Ingress Operator：管理路由与负载均衡。
- DNS Operator：管理集群 DNS 服务。
- etcd Operator：管理 etcd 集群的部署与升级。
- Kubernetes API Server Operator：管理 API Server。
- Kubernetes Controller Manager Operator：管理控制器组件。
- OpenShift Controller Manager Operator：管理 OpenShift 特有控制器。
- Console Operator：提供 Web 控制台。
- Cluster Autoscaler Operator：自动扩缩容节点。
- Config Operator：管理集群配置。
- CSI 驱动相关 Operator：如 csi-snapshot-controller。
## 快速验证命令
- **列出集群级 Operator 状态**：`oc get clusteroperator`。  
- **列出已安装 Operators（OLM）**：`oc get operators -n <namespace>` 或 `oc get csv -n <namespace>`。 
## 建议与注意
- **升级前检查 payload**：在升级前用离线/测试环境先验证 payload 中包含的 Operators 与 runlevel 顺序。  
- **关注 MCO 与重启窗口**：涉及不可变 OS 的变更会触发节点重启，需安排维护窗口并验证回滚路径。   [docs.redhat.com](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/operators/index)

