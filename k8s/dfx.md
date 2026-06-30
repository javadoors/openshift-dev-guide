# Kubernetes控制平面节点重装命令清单

下面是一份 **Kubernetes 控制平面节点安全重装命令清单**，涵盖准备、卸载、重装和验证四个阶段，确保集群数据和服务稳定。  

## 1. **准备阶段**
```bash
# 备份 etcd 数据
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 备份 Kubernetes 配置和证书
cp -r /etc/kubernetes /backup/kubernetes-config
cp -r /var/lib/kubelet /backup/kubelet-data
```

## 2. **卸载阶段**
```bash
# 停止核心服务
systemctl stop kubelet
systemctl stop containerd

# 卸载组件
apt remove -y kubelet kubeadm kubectl containerd
# 或者使用 yum
yum remove -y kubelet kubeadm kubectl containerd

# 清理节点配置
kubeadm reset -f
rm -rf /etc/cni/net.d
iptables -F && ipvsadm -C
```

## 3. **重装阶段**
```bash
# 重新安装组件
apt install -y kubelet kubeadm kubectl containerd
# 或 yum install -y kubelet kubeadm kubectl containerd

# 恢复配置文件
cp -r /backup/kubernetes-config/* /etc/kubernetes/
cp -r /backup/kubelet-data/* /var/lib/kubelet/

# 初始化控制平面
kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml

# 恢复 etcd 数据（如需要）
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd
```

## 4. **验证阶段**
```bash
# 检查节点状态
kubectl get nodes

# 检查 Pod 状态
kubectl get pods -A

# 检查 etcd 健康
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## 注意事项
- **证书恢复**：必须确保 `/etc/kubernetes/pki` 下的证书完整，否则控制平面无法启动。  
- **版本兼容**：kubelet、kubeadm、kubectl 与 API Server 版本必须一致。  
- **etcd数据安全**：操作 etcd 前务必做好快照备份。  

✅ **结论**：控制平面节点的重装比工作节点更复杂，涉及 etcd 和 API Server，必须严格按照 **备份 → 卸载 → 重装 → 恢复 → 验证** 的流程执行，才能保证集群安全。  


# Kubernetes核心组件安全重装流程

**在生产环境中，卸载并重新安装 Kubernetes 核心组件（如 kubelet、containerd、kube-proxy、etcd）会导致节点上的容器被终止，节点进入 `NotReady` 状态。安全的流程是：先驱逐节点，再卸载重装组件，最后恢复配置并验证节点状态。**

## 核心组件安全重装流程

### 1. **准备阶段**
- **备份配置文件**：如 `/var/lib/kubelet/config.yaml`、`/etc/containerd/config.toml`、`/etc/kubernetes/pki`。  
- **确认集群高可用**：确保 Pod 有多副本，避免单节点宕机导致业务中断。  
- **驱逐节点**：执行 `kubectl drain <node> --ignore-daemonsets --delete-local-data`，迁移 Pod。  

---

### 2. **卸载阶段**
- **停止服务**：`systemctl stop kubelet containerd kube-proxy`。  
- **卸载组件**：使用包管理器卸载，如 `apt remove kubelet containerd` 或 `yum remove kubelet containerd`。  
- **清理残留**：  
  - `kubeadm reset -f` 清理节点配置和证书。  
  - 删除 CNI 配置：`rm -rf /etc/cni/net.d`。  
  - 清理网络规则：`iptables -F && ipvsadm -C`。  [Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)  [Github](https://github.com/kubernetes/website/blob/main/content/en/docs/reference/setup-tools/kubeadm/kubeadm-reset.md)  

### 3. **重装阶段**
- **重新安装组件**：`apt install kubelet kubeadm kubectl containerd`。  
- **恢复配置文件**：将备份的配置文件复制回原路径。  
- **重新加入集群**：  
  - 工作节点：`kubeadm join <control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>`  
  - 控制平面节点：使用 `kubeadm init` 或 `kubeadm join --control-plane`。  

### 4. **验证阶段**
- **检查节点状态**：`kubectl get nodes`，确认节点为 `Ready`。  
- **检查 Pod 状态**：`kubectl get pods -A`，确认 Pod 正常运行。  
- **网络验证**：测试 Service 与 Pod 网络连通性。  

## 风险与注意事项
- **证书问题**：若 `/etc/kubernetes/pki` 未正确恢复，节点无法加入集群。  
- **版本兼容性**：确保 kubelet、kubeadm 与 API Server 版本一致。  
- **数据丢失**：控制平面节点卸载 etcd 时需谨慎，避免集群数据丢失。  
- **网络插件**：CNI 配置未清理或未恢复可能导致 Pod 网络异常。  [deepwiki.com](https://deepwiki.com/kubernetes-sigs/kubespray/8.3-cluster-reset-and-teardown)  

✅ **结论**：安全重装 Kubernetes 核心组件的关键是 **先驱逐节点 → 卸载清理 → 重装恢复 → 验证状态**。这样可以避免业务中断并保证集群稳定。  


# k8s的核心组件卸载重新安装影响k8s节点上运行的容器吗？

**卸载并重新安装 Kubernetes 的核心组件（如 kubelet、kube-proxy、containerd、etcd 等）会对节点上运行的容器产生明显影响：容器会被终止，节点可能进入 `NotReady` 状态，直到组件恢复并重新与集群通信。**

## 影响范围
- **kubelet**  
  节点代理，负责容器生命周期管理。卸载后，Pod 无法被调度或管理，已有容器会被终止。  

- **containerd**  
  容器运行时，负责实际创建和运行容器。卸载后，所有容器进程会停止。  

- **kube-proxy**  
  负责 Service 的网络转发。卸载后，节点上的服务流量无法正确转发，影响网络访问。  

- **etcd**（仅限控制平面节点）  
  集群的状态存储。卸载或异常会导致整个集群无法正常工作，而不仅仅是单个节点。  

## 操作风险
- **节点状态**：卸载核心组件后，节点会进入 `NotReady`，调度器停止分配 Pod。  
- **业务中断**：正在运行的容器会被终止，服务可能不可用。  
- **配置丢失**：若未备份配置文件，重新安装后可能无法恢复原有环境。  
- **版本不兼容**：核心组件与 API Server 或运行时版本不匹配可能导致节点无法恢复。  

## 安全操作流程
1. **驱逐节点**：`kubectl drain <node>` 将 Pod 迁移到其他节点。  
2. **卸载并重装组件**：根据需要卸载并重新安装 kubelet、containerd 等。  
3. **恢复配置**：确保 `/var/lib/kubelet`、`/etc/containerd/config.toml` 等配置文件正确。  
4. **重启服务**：`systemctl restart kubelet` 和 `systemctl restart containerd`。  
5. **验证节点恢复**：`kubectl get nodes` 确认节点状态为 `Ready`，检查 Pod 是否正常运行。  

✅ **结论**：卸载并重装 Kubernetes 核心组件会导致节点上的容器被终止，节点进入 `NotReady`。在生产环境中，必须先驱逐节点，再进行操作，并确保配置和版本兼容，才能安全恢复。  

# kubelet卸载重新安装影响k8s节点上运行的容器吗？

**卸载并重新安装 `kubelet` 会对 Kubernetes 节点上的容器产生直接影响：容器会被终止，节点会进入 `NotReady` 状态，直到 `kubelet` 恢复并重新与运行时（如 containerd）和 API Server 建立连接。**

## 影响分析
- **节点管理**  
  `kubelet` 是节点的核心代理，负责与容器运行时交互并上报节点状态。卸载后，节点无法与集群通信，调度器会停止在该节点分配 Pod。  

- **容器运行**  
  已经运行的容器会失去管理，通常会被终止。即使容器进程暂时存活，缺少 kubelet 管理，它们不会被 Kubernetes 视为有效 Pod。  

- **服务可用性**  
  如果没有提前迁移 Pod，业务服务可能会中断。高可用部署（多副本、Pod 分布在不同节点）可以降低影响。  

## 正确操作步骤
1. **驱逐节点**  
   使用 `kubectl drain <node>` 将 Pod 安全迁移到其他节点。  

2. **卸载并重装 kubelet**  
   - 卸载旧版本：`apt remove kubelet` 或 `yum remove kubelet`  
   - 安装新版本：`apt install kubelet`  
   - 确保配置文件 `/var/lib/kubelet/config.yaml` 与集群一致。  

3. **重启服务**  
   - `systemctl restart kubelet`  
   - 确认 kubelet 能正常连接 API Server 与运行时。  

4. **验证节点恢复**  
   - `kubectl get nodes` 查看节点是否恢复为 `Ready`  
   - `kubectl get pods -A` 检查 Pod 是否重新调度并运行正常  

## 常见风险
- **证书失效**：kubelet 使用的 TLS 证书若未正确恢复，节点无法加入集群。  
- **版本不兼容**：kubelet 与 API Server 或 containerd 版本不匹配可能导致通信失败。  
- **配置丢失**：未备份 `/var/lib/kubelet` 下的配置文件可能导致节点无法识别。  

✅ **结论**：卸载重装 kubelet 会导致节点上的容器被终止，节点进入 `NotReady`。最佳实践是先驱逐节点，再重装并验证配置，最后恢复调度。  

# containerd卸载重新安装影响k8s节点上运行的容器吗？

**卸载并重新安装 `containerd` 会直接影响 Kubernetes 节点上正在运行的容器：容器会被终止，节点可能进入 `NotReady` 状态，直到 `containerd` 和 `kubelet` 服务恢复正常。要避免业务中断，必须做好 Pod 驱逐和配置恢复。**

## 影响分析
- **容器生命周期**  
  Kubernetes 依赖 `containerd` 提供 CRI 接口来创建和管理容器。卸载后，所有容器会停止运行，Pod 无法启动。  [CSDN问答](https://ask.csdn.net/questions/8685389)  

- **节点状态**  
  当 `containerd` 缺失或异常时，`kubelet` 无法与运行时通信，节点会被标记为 `NotReady`，调度器会停止在该节点分配新 Pod。  [CSDN问答](https://ask.csdn.net/questions/8685389)  

- **服务中断风险**  
  如果没有提前驱逐 Pod 或做好高可用部署，业务服务可能会短暂不可用。  [oneuptime.com](https://oneuptime.com/blog/post/2026-02-09-migrate-docker-containerd-existing-clusters/view)  

## 正确操作步骤
1. **驱逐节点**  
   使用 `kubectl drain <node>` 将节点上的 Pod 迁移到其他节点，避免业务中断。  

2. **卸载并重装 containerd**  
   - 卸载旧版本：`apt remove containerd.io` 或 `yum remove containerd.io`  
   - 安装新版本：`apt install containerd.io`  
   - 生成并修改配置文件 `/etc/containerd/config.toml`，确保启用 CRI 插件和正确的 cgroup 驱动。  [CSDN问答](https://ask.csdn.net/questions/8685389)  

3. **重启服务**  
   - `systemctl restart containerd`  
   - `systemctl restart kubelet`  

4. **验证节点恢复**  
   - `kubectl get nodes` 确认节点状态为 `Ready`  
   - `kubectl get pods -A` 检查 Pod 是否正常运行  

## 常见问题与风险
- **镜像拉取失败**：若仓库认证或网络配置未恢复，Pod 会卡在 `ImagePullBackOff`。  
- **CRI 接口异常**：配置文件未启用 `[plugins."io.containerd.grpc.v1.cri"]` 会导致 kubelet 无法通信。  
- **CNI 网络失效**：containerd 未正确配置时，Pod 网络无法初始化。  
- **版本不兼容**：containerd 与 kubelet/Kubernetes 版本不匹配可能导致节点无法恢复。  [CSDN问答](https://ask.csdn.net/questions/8685389)  

✅ **结论**：卸载重装 `containerd` 会中断节点上的容器运行。最佳实践是先驱逐节点，再重装并验证配置，最后恢复节点调度。这样可以避免业务中断。  

# 
