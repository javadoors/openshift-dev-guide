# 采集 K8s 集群核心组件日志

## 采集 K8s 集群核心组件日志

K8s 核心组件按部署方式分两类，采集方法不同。

### 一、核心组件分类

| 部署方式 | 组件 | 运行形态 |
|---------|------|---------|
| **静态 Pod** | kube-apiserver、kube-controller-manager、kube-scheduler、etcd | 以 Static Pod 运行于 `kube-system` 命名空间 |
| **Node 级进程** | kubelet、kube-proxy | 以 systemd 服务运行于每个节点 |

### 二、静态 Pod 组件日志采集（kubectl logs）

#### 2.1 单组件日志

```bash
# 查看 apiserver 日志（默认最新 20 行）
kubectl logs -n kube-system kube-apiserver-<master-node-name>

# 查看指定时间段的日志
kubectl logs -n kube-system kube-apiserver-<master-node-name> \
  --since=2026-07-14T00:00:00Z --until=2026-07-14T12:00:00Z

# 查看最近 1 小时的日志
kubectl logs -n kube-system kube-apiserver-<master-node-name> --since=1h

# 查看最后 100 行
kubectl logs -n kube-system kube-apiserver-<master-node-name> --tail=100

# 跟踪实时日志
kubectl logs -n kube-system kube-apiserver-<master-node-name> -f

# 查看前一个容器实例的日志（容器重启后）
kubectl logs -n kube-system kube-apiserver-<master-node-name> --previous
```

#### 2.2 批量采集所有控制面组件

```bash
# 定义 master 节点名
MASTER_NODE="<master-node-name>"

# 批量导出到文件
for comp in kube-apiserver kube-controller-manager kube-scheduler etcd; do
  pod=$(kubectl get pods -n kube-system -o wide | grep "${comp}-${MASTER_NODE}" | awk '{print $1}')
  if [ -n "$pod" ]; then
    kubectl logs -n kube-system "$pod" --tail=10000 > "${comp}-${MASTER_NODE}.log" 2>&1
    echo "已采集: ${comp} -> ${comp}-${MASTER_NODE}.log"
  fi
done
```

#### 2.3 采集所有 master 节点的控制面组件

```bash
# 获取所有 master 节点
MASTER_NODES=$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o jsonpath='{.items[*].metadata.name}')

for node in $MASTER_NODES; do
  for comp in kube-apiserver kube-controller-manager kube-scheduler etcd; do
    pod=$(kubectl get pods -n kube-system -o wide | grep "${comp}-${node}" | awk '{print $1}')
    [ -n "$pod" ] && kubectl logs -n kube-system "$pod" --tail=5000 > "${comp}-${node}.log"
  done
done
```

### 三、Node 级组件日志采集（journalctl）

kubelet 和 kube-proxy 以 systemd 服务运行，需通过 `journalctl` 在节点上采集。

#### 3.1 在节点上直接采集

```bash
# SSH 到目标节点后执行

# kubelet 日志（最近 1 小时）
journalctl -u kubelet --since "1 hour ago" > kubelet.log

# kube-proxy 日志
journalctl -u kube-proxy --since "1 hour ago" > kube-proxy.log

# 实时跟踪
journalctl -u kubelet -f

# 查看指定时间段
journalctl -u kubelet --since "2026-07-14 00:00:00" --until "2026-07-14 12:00:00"

# 查看上一次启动的日志（节点重启后）
journalctl -u kubelet -b -1
```

#### 3.2 通过 kubectl 远程采集（无需 SSH）

利用 `kubectl debug` 或临时 Pod 采集：

```bash
# 在节点上创建特权 Pod 采集 journalctl
kubectl debug node/<node-name> -it --image=busybox --profile=sysadmin -- sh -c \
  "cat /host/var/log/journal/$(ls /host/var/log/journal)/system.journal" > kubelet.log

# 或者使用 nsenter 进入节点命名空间（需要特权 Pod）
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-collector
  namespace: kube-system
spec:
  nodeName: <node-name>
  hostNetwork: true
  containers:
  - name: collector
    image: alpine
    command: ["nsenter", "--target", "1", "--mount", "--", "journalctl", "-u", "kubelet", "--since", "1 hour ago"]
    securityContext:
      privileged: true
  restartPolicy: Never
EOF

kubectl logs -n kube-system log-collector > kubelet.log
kubectl delete pod -n kube-system log-collector
```

#### 3.3 从宿主机日志文件直接读取

静态 Pod 的日志也会写入节点文件系统：

```bash
# 静态 Pod 日志文件路径
ls /var/log/pods/kube-system_kube-apiserver-<node>_<pod-uid>/

# containerd 运行时的容器日志
ls /var/log/containers/

# 对应的日志文件
/var/log/containers/kube-apiserver-<node>_kube-system_kube-apiserver-<container-id>.log
  -> /var/log/pods/kube-system_kube-apiserver-<node>_<uid>/<container>/0.log
```

### 四、一键全量采集脚本

```bash
#!/bin/bash
# collect-k8s-logs.sh - 一键采集所有核心组件日志
set -e

OUTPUT_DIR="k8s-logs-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "=== 采集控制面静态 Pod 日志 ==="
MASTER_NODES=$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o jsonpath='{.items[*].metadata.name}')

for node in $MASTER_NODES; do
  echo "  采集 master 节点: $node"
  for comp in kube-apiserver kube-controller-manager kube-scheduler etcd; do
    pod=$(kubectl get pods -n kube-system -o wide 2>/dev/null | grep "${comp}-${node}" | awk '{print $1}')
    if [ -n "$pod" ]; then
      kubectl logs -n kube-system "$pod" --tail=10000 > "$OUTPUT_DIR/${comp}-${node}.log" 2>&1
      echo "    ✓ ${comp}"
    fi
  done
done

echo "=== 采集所有节点 kube-proxy 日志 ==="
NODES=$(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')
for node in $NODES; do
  pod=$(kubectl get pods -n kube-system -o wide -l k8s-app=kube-proxy | grep "$node" | awk '{print $1}')
  if [ -n "$pod" ]; then
    kubectl logs -n kube-system "$pod" --tail=10000 > "$OUTPUT_DIR/kube-proxy-${node}.log" 2>&1
    echo "  ✓ kube-proxy-${node}"
  fi
done

echo "=== 采集集群事件 ==="
kubectl get events -n kube-system --sort-by='.lastTimestamp' > "$OUTPUT_DIR/events-kube-system.txt"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' > "$OUTPUT_DIR/events-all.txt"

echo "=== 采集节点状态 ==="
kubectl describe nodes > "$OUTPUT_DIR/nodes-describe.txt"

echo ""
echo "采集完成，日志目录: $OUTPUT_DIR"
echo "打包: tar -czf ${OUTPUT_DIR}.tar.gz $OUTPUT_DIR"
```

### 五、过滤关键错误日志

```bash
# apiserver 中的错误
kubectl logs -n kube-system kube-apiserver-<node> | grep -E "ERROR|FATAL|panic|failed"

# etcd 中的告警
kubectl logs -n kube-system etcd-<node> | grep -E "ERROR|WARN|alarm"

# kubelet 中的 Pod 启动失败
journalctl -u kubelet | grep -E "Failed|Error|error"

# 按错误等级过滤（apiserver 日志格式）
kubectl logs -n kube-system kube-apiserver-<node> | \
  jq -r 'select(.level=="error" or .level=="fatal") | "\(.ts) \(.msg)"'
```

### 六、采集方式对比

| 方式 | 适用组件 | 优点 | 缺点 |
|------|---------|------|------|
| `kubectl logs` | 静态 Pod 组件 | 无需 SSH，可远程 | 受 `--log-text-info-limit` 等限制 |
| `journalctl` | kubelet、kube-proxy | 完整 systemd 日志 | 需 SSH 到节点 |
| 宿主机日志文件 | 所有容器 | 可获取历史轮转日志 | 需 SSH，路径复杂 |
| `kubectl debug` | Node 级组件 | 无需 SSH | 需创建特权 Pod |
| 采集脚本 | 全部 | 一键全量 | 产出文件较大 |

### 七、日志持久化建议

K8s 默认日志不持久化，容器重建后日志丢失。生产环境建议：

1. **节点日志轮转**：配置 kubelet `--container-log-max-files` 和 `--container-log-max-size`
2. **集中采集**：部署 Fluent Bit / Filebeat 将日志发送到 Elasticsearch / Loki
3. **journalctl 持久化**：配置 `/etc/systemd/journald.conf` 的 `Storage=persistent`
4. **审计日志**：开启 kube-apiserver `--audit-log-path` 记录 API 操作审计

### 八、针对本项目的扩展

在 BKE 升级场景中，BKEAgent 负责在节点上执行升级命令，其日志采集方式：

```bash
# BKEAgent 以 systemd 服务运行
journalctl -u bkeagent --since "1 hour ago"

# BKEAgent 日志文件（若配置了文件日志）
ls /var/log/bkeagent/

# 升级过程中容器运行时日志（containerd）
journalctl -u containerd --since "1 hour ago"
```

升级失败时，建议同时采集：**控制面组件日志 + BKEAgent 日志 + containerd 日志 + kubelet 日志**，覆盖完整调用链。

