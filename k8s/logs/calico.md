# Calico

## 采集 Calico 日志

Calico 主要组件分两类：以 Pod 运行的 `calico-node` 和 `calico-kube-controllers`，以及节点上的 `calicoctl` 诊断工具。

### 一、Calico 组件分类

| 组件 | 部署方式 | 运行内容 | 命名空间 |
|------|---------|---------|---------|
| **calico-node** | DaemonSet | `felix`（策略/端点管理）+ `bird`（BGP 路由）+ `confd` | kube-system / calico-system |
| **calico-kube-controllers** | Deployment | IPAM 同步、节点状态同步、策略控制器 | kube-system / calico-system |
| **calicoctl** | 节点二进制 | 诊断/配置工具 | 节点级 |

### 二、采集 calico-node 日志（kubectl logs）

#### 2.1 查找 calico-node Pod

```bash
# 查看所有 calico-node Pod（注意命名空间可能是 kube-system 或 calico-system）
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
# 或
kubectl get pods -n calico-system -l k8s-app=calico-node -o wide

# 获取指定节点的 calico-node Pod
NODE_NAME="<node-name>"
POD_NAME=$(kubectl get pods -n kube-system -l k8s-app=calico-node -o wide | grep "$NODE_NAME" | awk '{print $1}')
```

#### 2.2 查看容器日志

calico-node Pod 默认包含**一个容器**（calico-node），但内部运行多个进程：

```bash
NS=kube-system  # 或 calico-system

# 查看日志（默认最新）
kubectl logs -n $NS $POD_NAME

# 跟踪实时日志
kubectl logs -n $NS $POD_NAME -f

# 最近 1 小时
kubectl logs -n $NS $POD_NAME --since=1h

# 最后 200 行
kubectl logs -n $NS $POD_NAME --tail=200

# 上一次容器实例（容器重启后）
kubectl logs -n $NS $POD_NAME --previous
```

#### 2.3 区分 felix 和 bird 进程日志

calico-node 容器内 `felix` 和 `bird` 日志混在一起，可通过进程标识过滤：

```bash
# 过滤 felix 日志（策略/端点）
kubectl logs -n $NS $POD_NAME | grep -i "felix"

# 过滤 bird 日志（BGP 路由）
kubectl logs -n $NS $POD_NAME | grep -iE "bird|BGP"

# 过滤 confd 日志
kubectl logs -n $NS $POD_NAME | grep -i "confd"
```

#### 2.4 批量采集所有节点

```bash
#!/bin/bash
OUTPUT_DIR="calico-logs-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

# 自动探测命名空间
NS=$(kubectl get ds -A | grep calico-node | awk '{print $1}')
echo "calico-node 命名空间: $NS"

# 采集所有 calico-node
for pod in $(kubectl get pods -n $NS -l k8s-app=calico-node -o jsonpath='{.items[*].metadata.name}'); do
  node=$(kubectl get pod -n $NS $pod -o jsonpath='{.spec.nodeName}')
  kubectl logs -n $NS $pod --tail=10000 > "$OUTPUT_DIR/calico-node-${node}.log" 2>&1
  echo "  ✓ calico-node-${node}"
done

echo "采集完成: $OUTPUT_DIR"
```

### 三、采集 calico-kube-controllers 日志

```bash
# 查找 Pod
kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
# 或
kubectl get pods -n calico-system -l k8s-app=calico-kube-controllers

# 查看日志
kubectl logs -n $NS -l k8s-app=calico-kube-controllers

# 指定 Pod
kubectl logs -n $NS <calico-kube-controllers-pod> --tail=5000
```

### 四、进入容器查看进程级日志文件

calico-node 容器内进程会将日志写入文件，可进入容器直接查看：

```bash
kubectl exec -n $NS $POD_NAME -it -- bash

# 容器内常用日志路径
# felix 日志
cat /var/log/calico/felix/current  2>/dev/null || ls /var/log/calico/felix/
# bird 日志
cat /var/log/calico/bird/current   2>/dev/null || ls /var/log/calico/bird/
# confd 日志
cat /var/log/calico/confd/current  2>/dev/null || ls /var/log/calico/confd/

# 实时查看 felix 日志
tail -f /var/log/calico/felix/current
```

**注意**：日志文件路径因 Calico 版本而异，新版（v3.21+）默认输出到 stdout，不再写文件。

### 五、使用 calicoctl 采集诊断信息

`calicoctl` 是 Calico 的诊断工具，可采集集群状态快照。

#### 5.1 在节点上使用 calicoctl

```bash
# 若节点已安装 calicoctl
calicoctl node status
# 输出 BGP 邻居关系、路由状态

# 查看 felix 运行状态
calicoctl node diags

# 保存诊断信息到压缩包
calicoctl node diags --log-path=/tmp/calico-diags
```

#### 5.2 通过 kubectl 运行 calicoctl（无需节点安装）

```bash
# 下载 calicoctl manifest
kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml

# 或直接用 Pod 执行
kubectl exec -n kube-system calicoctl -- calicoctl get nodes
kubectl exec -n kube-system calicoctl -- calicoctl node status
```

#### 5.3 关键诊断命令

```bash
# 查看所有节点状态
calicoctl get nodes -o wide

# 查看 BGP 邻居状态（排查网络不通）
calicoctl node status

# 查看 IPAM 分配情况
calicoctl ipam show
calicoctl ipam show --ip=<problem-ip>

# 查看工作负载端点
calicoctl get workloadEndpoint -o wide

# 查看策略
calicoctl get policy -A -o wide

# 查看 IP 池
calicoctl get ippool -o yaml
```

### 六、节点宿主机日志（systemd 部署场景）

部分场景下 Calico 以 systemd 服务运行（非容器化）：

```bash
# 查看 Felix 日志
journalctl -u calico-felix --since "1 hour ago"

# 查看节点上的 calico-node 服务
journalctl -u calico-node --since "1 hour ago"

# 查看历史日志文件（按日期）
ls /var/log/calico/
```

### 七、一键全量采集脚本

```bash
#!/bin/bash
# collect-calico-logs.sh - 一键采集 Calico 全量日志与诊断信息
set -e

OUTPUT_DIR="calico-logs-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

# 自动探测命名空间
NS=$(kubectl get ds -A 2>/dev/null | grep calico-node | awk '{print $1}' | head -1)
[ -z "$NS" ] && NS=kube-system
echo "=== Calico 命名空间: $NS ==="

echo "=== 1. 采集 calico-node 日志 ==="
for pod in $(kubectl get pods -n $NS -l k8s-app=calico-node -o jsonpath='{.items[*].metadata.name}'); do
  node=$(kubectl get pod -n $NS $pod -o jsonpath='{.spec.nodeName}')
  kubectl logs -n $NS $pod --tail=10000 > "$OUTPUT_DIR/calico-node-${node}.log" 2>&1
  echo "  ✓ calico-node-${node}"
done

echo "=== 2. 采集 calico-kube-controllers 日志 ==="
for pod in $(kubectl get pods -n $NS -l k8s-app=calico-kube-controllers -o jsonpath='{.items[*].metadata.name}'); do
  kubectl logs -n $NS $pod --tail=10000 > "$OUTPUT_DIR/${pod}.log" 2>&1
  echo "  ✓ ${pod}"
done

echo "=== 3. 采集 Calico 资源状态 ==="
kubectl get pods -n $NS -l k8s-app=calico-node -o wide > "$OUTPUT_DIR/calico-node-pods.txt"
kubectl get pods -n $NS -l k8s-app=calico-kube-controllers -o wide > "$OUTPUT_DIR/calico-kube-controllers-pods.txt"
kubectl describe pods -n $NS -l k8s-app=calico-node > "$OUTPUT_DIR/calico-node-describe.txt"

echo "=== 4. 采集 Calico 自定义资源 ==="
kubectl get installation -A -o yaml > "$OUTPUT_DIR/installation.yaml" 2>&1 || true
kubectl get ippool -A -o yaml > "$OUTPUT_DIR/ippool.yaml" 2>&1 || true
kubectl get bgpconfig -A -o yaml > "$OUTPUT_DIR/bgpconfig.yaml" 2>&1 || true
kubectl get bgppeer -A -o yaml > "$OUTPUT_DIR/bgppeer.yaml" 2>&1 || true
kubectl get felixconfiguration -A -o yaml > "$OUTPUT_DIR/felixconfiguration.yaml" 2>&1 || true

echo "=== 5. 采集 felix 诊断信息 ==="
for pod in $(kubectl get pods -n $NS -l k8s-app=calico-node -o jsonpath='{.items[*].metadata.name}'); do
  node=$(kubectl get pod -n $NS $pod -o jsonpath='{.spec.nodeName}')
  kubectl exec -n $NS $pod -- calico-node -diagnostics 2>&1 > "$OUTPUT_DIR/felix-diags-${node}.txt" || \
    echo "  ! ${node} 不支持 diagnostics"
done

echo ""
echo "采集完成: $OUTPUT_DIR"
echo "打包: tar -czf ${OUTPUT_DIR}.tar.gz $OUTPUT_DIR"
```

### 八、过滤关键错误日志

```bash
# felix 策略错误
kubectl logs -n $NS $POD_NAME | grep -iE "felix.*error|policy.*fail"

# BGP 邻居异常
kubectl logs -n $NS $POD_NAME | grep -iE "bird|bgp" | grep -iE "error|down|fail"

# 端点创建失败
kubectl logs -n $NS $POD_NAME | grep -iE "endpoint.*fail|workload.*fail"

# IPAM 分配失败
kubectl logs -n $NS $POD_NAME | grep -iE "ipam.*fail|assign.*fail|no.*free.*ip"

# 连接 kube-apiserver 失败
kubectl logs -n $NS $POD_NAME | grep -iE "apiserver|connection.*refused|timeout"
```

### 九、采集方式对比

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| `kubectl logs` | 容器化部署 | 远程采集，无需 SSH | 日志受 K8s 轮转限制 |
| 容器内日志文件 | 需要进程级日志 | felix/bird 分离 | 需 exec 进入容器 |
| `journalctl` | systemd 部署 | 完整 systemd 日志 | 需 SSH |
| `calicoctl node status` | BGP 排查 | 直观显示邻居关系 | 仅当前节点 |
| `calicoctl node diags` | 全量诊断 | 打包所有诊断信息 | 产出大 |
| 一键脚本 | 全量排查 | 覆盖完整 | 产出较大 |

### 十、常见故障对应日志关键词

| 故障现象 | 关键日志关键词 | 检查组件 |
|---------|--------------|---------|
| Pod 无法获取 IP | `ipam`、`no free IPs`、`assign fail` | calico-node + kube-controllers |
| 跨节点 Pod 不通 | `BGP`、`peer down`、`route not found` | calico-node (bird) |
| 策略不生效 | `policy`、`endpoint fail`、`iptables` | calico-node (felix) |
| 端点创建失败 | `workloadEndpoint`、`iface`、`veth` | calico-node (felix) |
| 与 apiserver 通信失败 | `apiserver`、`connection refused`、`watch` | calico-kube-controllers |

### 十一、针对 BKE 升级场景的建议

在 BKE 升级场景中，Calico 可能受影响的情况：

1. **containerd 升级**：容器运行时重启会导致 calico-node 容器重建，建议升级前后都采集一次日志对比
2. **master 升级**：apiserver 重启期间 calico-kube-controllers 可能 watch 失败，关注 `connection refused` 日志
3. **网络插件版本变更**：若升级涉及 Calico 版本变化，务必升级前后都采集 `installation` CR 和 felix 配置

```bash
# 升级前快照
./collect-calico-logs.sh
mv calico-logs-* calico-logs-pre-upgrade

# 升级后快照
./collect-calico-logs.sh
mv calico-logs-* calico-logs-post-upgrade

# 对比关键资源变化
diff calico-logs-pre-upgrade/ippool.yaml calico-logs-post-upgrade/ippool.yaml
```
