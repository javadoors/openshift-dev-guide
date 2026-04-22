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
