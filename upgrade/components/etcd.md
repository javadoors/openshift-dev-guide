# etcd3 三节点集群升级的完备方案
下面给你一个 **etcd3 三节点集群升级的完备方案**，包含数据备份与恢复，并讲清楚原理与设计思路，同时分析对数据一致性和数据丢失的影响。  

## 📑 原理与设计思路
1. **Raft 共识机制**  
   - etcd 使用 Raft 算法保证强一致性。只要多数节点（quorum）可用，集群就能继续提供服务。  
   - 在 3 节点集群中，至少 2 个节点在线才能维持 quorum。  

2. **滚动升级**  
   - 逐个节点停机、替换二进制、重启并验证。  
   - 保证在升级过程中始终有多数节点在线，避免集群不可用。  

3. **数据持久化**  
   - etcd 数据存储在 WAL（Write Ahead Log）和快照文件中。  
   - 升级时必须保留这些文件，否则会丢失历史数据。  

4. **快照恢复机制**  
   - 使用 `etcdctl snapshot save/restore` 可以完整备份和恢复集群状态。  
   - 快照包含集群的 KV 状态和元信息，恢复时可重建集群。  

## 📊 升级步骤（3 节点集群）
1. **升级前准备**  
   - 确认所有节点版本一致：`etcdctl member list`。  
   - 健康检查：`etcdctl endpoint health`。  
   - 快照备份：  
     ```bash
     etcdctl snapshot save /backup/etcd-$(date +%F).db
     ```
   - 数据目录备份：  
     ```bash
     systemctl stop etcd
     tar -czf /backup/etcd-data-$(date +%F).tar.gz /var/lib/etcd
     systemctl start etcd
     ```

2. **滚动升级流程**  
   - 停止第一个节点：`systemctl stop etcd`  
   - 替换二进制（安装新版本）。  
   - 启动并验证：`systemctl start etcd && etcdctl endpoint health`  
   - 确认该节点重新加入集群。  
   - 依次升级剩余两个节点。  

3. **升级后验证**  
   - 确认所有成员版本一致：`etcdctl member list`。  
   - 检查集群健康：`etcdctl endpoint status --write-out=table`。  

## 🔄 数据恢复方案（升级失败时）
1. 停止所有节点：`systemctl stop etcd`  
2. 使用快照恢复：  
   ```bash
   etcdctl snapshot restore /backup/etcd-2026-04-27.db \
     --data-dir /var/lib/etcd-restored
   ```
3. 替换原数据目录为恢复后的目录。  
4. 启动集群：`systemctl start etcd`  
   - 集群会恢复到备份时的状态。  

## ⚠️ 数据一致性与丢失分析
- **一致性**：  
  - Raft 保证强一致性。滚动升级时始终保持多数节点在线，因此不会破坏一致性。  
  - 升级过程中，leader 可能发生切换，但不会导致数据不一致。  

- **数据丢失风险**：  
  - 如果升级前未做快照备份，且数据目录损坏或误删，可能导致数据丢失。  
  - 如果 quorum 丧失（例如同时停机两个节点），可能导致集群不可用，但不会丢失已持久化的数据。  
  - 快照恢复会回到备份时的状态，备份之后的写入会丢失。  

## ✅ 总结
- **设计思路**：依赖 Raft 共识，采用滚动升级，保证多数节点在线。  
- **数据保障**：升级前必须做快照和数据目录备份。  
- **恢复机制**：升级失败时通过快照恢复，保证集群状态一致。  
- **一致性与丢失**：升级不会破坏数据一致性，但如果未备份，可能在故障时丢失数据；快照恢复会丢失备份之后的写入。  
