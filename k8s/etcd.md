# etcd备份数据的命令行方案
**在 Kubernetes 集群中，备份 etcd 数据的最佳命令行方案是使用 `etcdctl snapshot save`，结合证书参数（`--cacert`、`--cert`、`--key`）来保证安全性。恢复时则使用 `etcdctl snapshot restore` 或 `etcdutl restore`。**  
## 📑 etcd 备份命令行流程
### 1. 登录控制平面节点
- SSH 到运行 etcd 的控制平面节点。  
- 确认已安装 `etcdctl` 和 `etcdutl` 工具。  
### 2. 获取证书与端点信息
- 在 `/etc/kubernetes/manifests/etcd.yaml` 中找到以下参数：  
  - `--endpoints=https://127.0.0.1:2379`  
  - `--cacert=/etc/kubernetes/pki/etcd/ca.crt`  
  - `--cert=/etc/kubernetes/pki/etcd/server.crt`  
  - `--key=/etc/kubernetes/pki/etcd/server.key`  
### 3. 执行备份命令
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d%H%M).db
```
- **输出文件**：`etcd-snapshot-YYYYMMDDHHMM.db`  
- 建议存放在安全目录并定期同步到远程存储。  
### 4. 验证快照
```bash
ETCDCTL_API=3 etcdutl snapshot status /backup/etcd-snapshot-20260422.db
```
- 可查看快照的元信息（版本、大小、修订号）。  
## 📑 etcd 恢复命令行流程
```bash
ETCDCTL_API=3 etcdutl snapshot restore /backup/etcd-snapshot-20260422.db \
  --data-dir /var/lib/etcd-restored
```
- 恢复后需更新 etcd 的 `--data-dir` 指向新目录，并重启 etcd 服务。  
## ⚠️ 风险与最佳实践
- **定期备份**：建议每日快照，并保留至少 7 天历史。  
- **异地存储**：避免单点故障，快照应上传至对象存储或远程服务器。  
- **一致性**：在多节点 etcd 集群中，任意一个节点的快照即可恢复整个集群。  
- **恢复测试**：定期演练恢复流程，确保灾难发生时能快速恢复。  
## ✅ 总结
- **备份命令**：`etcdctl snapshot save`  
- **验证命令**：`etcdutl snapshot status`  
- **恢复命令**：`etcdutl snapshot restore`  
- **最佳实践**：定期、异地、带验证的快照备份，并演练恢复。  

# k8s集群中3节点etcd逐节点滚动升级的shell脚本方案
在一个 **3 节点 etcd 集群**中做逐节点滚动升级时，最佳实践是 **动态获取 etcd 节点信息**，而不是手工写死节点名。这样可以避免节点变更或扩容时脚本失效。下面给你一个可直接套用的 **Shell 脚本方案**：  
## 📑 动态获取节点并逐节点升级
```bash
#!/bin/bash
# 动态获取 etcd 节点并逐个升级
# 需在控制平面节点执行，确保有 kubectl 权限

NEW_VERSION="v3.5.13"
BACKUP_DIR="/backup/etcd"

# 1. 获取 etcd Pod 列表（假设运行在 kube-system 命名空间）
ETCD_PODS=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[*].metadata.name}')

for POD in $ETCD_PODS; do
  NODE=$(kubectl get pod $POD -n kube-system -o jsonpath='{.spec.nodeName}')
  echo "=== 开始升级节点: $NODE (Pod: $POD) ==="

  # 2. 备份快照（只需一次，但这里每个节点都做冗余备份）
  ssh $NODE "ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save $BACKUP_DIR/${NODE}-snapshot-$(date +%Y%m%d%H%M).db"

  # 3. 停止 etcd 服务（静态 Pod，需先移除 manifest）
  ssh $NODE "mv /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/etcd.yaml.bak"

  # 等待 Pod 停止
  sleep 10

  # 4. 升级 etcd 二进制
  ssh $NODE "curl -L https://github.com/etcd-io/etcd/releases/download/${NEW_VERSION}/etcd-${NEW_VERSION}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz"
  ssh $NODE "tar -xzf /tmp/etcd.tar.gz -C /tmp && cp /tmp/etcd-${NEW_VERSION}-linux-amd64/etcd* /usr/local/bin/"

  # 5. 恢复 manifest，重新启动 etcd
  ssh $NODE "mv /etc/kubernetes/manifests/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml"

  # 等待 Pod 启动
  sleep 20

  # 6. 验证节点健康
  ssh $NODE "ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint health"

  echo "=== 节点 $NODE 升级完成 ==="
done

echo "=== 所有 etcd 节点升级完成 ==="
```
## ⚙️ 方案要点
- **动态发现节点**：通过 `kubectl get pods -n kube-system -l component=etcd` 获取 etcd Pod，再解析出对应的节点名。  
- **逐节点升级**：循环每个节点，保证集群始终有多数节点存活。  
- **快照备份**：虽然只需一次，但脚本中每个节点都做冗余备份，提升安全性。  
- **静态 Pod 管理**：etcd 在 kubeadm 部署中是静态 Pod，需通过移动 manifest 文件来停止/启动。  
- **健康检查**：升级后立即用 `etcdctl endpoint health` 验证。  
## ✅ 总结
- **快照只需一次**，但在脚本中可以选择每个节点都备份，增加冗余。  
- **动态获取节点信息**，避免硬编码，适应集群变更。  
- **逐节点滚动升级**，保证集群始终可用。  
