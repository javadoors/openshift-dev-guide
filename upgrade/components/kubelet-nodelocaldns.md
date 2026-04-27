# 升级 kubelet 与 NodeLocal DNSCache（nodelocaldns）的关系
**升级 kubelet 与 NodeLocal DNSCache（nodelocaldns）的关系在于：kubelet 负责为 Pod 配置 DNS，而 NodeLocal DNSCache 是在每个节点上运行的本地 DNS 缓存代理。两者结合后，Pod 的 DNS 查询会优先走节点本地缓存，从而减少延迟和 conntrack 负载，提高稳定性。**  

## 📑 关系说明
- **kubelet 的角色**  
  - 在 Pod 启动时，kubelet 会生成 Pod 的 `/etc/resolv.conf` 文件。  
  - 默认情况下，Pod 的 DNS 指向集群的 kube-dns/CoreDNS Service IP。  

- **NodeLocal DNSCache 的角色**  
  - 以 DaemonSet 部署在每个节点上，监听一个本地 IP（通常是 `169.254.20.10`）。  
  - Pod 的 DNS 查询会直接发到本地缓存代理，而不是跨节点访问 kube-dns Service。  
  - 本地代理在缓存未命中时再去查询 kube-dns/CoreDNS。  

- **两者的结合点**  
  - kubelet 的 `--cluster-dns` 参数需要指向 NodeLocal DNSCache 的本地监听 IP。  
  - 当 NodeLocal DNSCache 部署后，kubelet 会为 Pod 配置 `/etc/resolv.conf`，让 Pod 的 DNS 查询走本地缓存。  
  - 如果 kube-proxy 使用 IPVS 模式，则必须显式修改 kubelet 的 `--cluster-dns` 参数；否则 NodeLocal DNSCache 会同时监听 kube-dns Service IP 和本地 IP，无需额外修改。  [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)  [Stack Overflow](https://stackoverflow.com/questions/71920188/how-to-enable-node-local-dns-cache-on-eks)  

## 📊 优势
| 优势 | 说明 |
|------|------|
| **降低延迟** | Pod DNS 查询直接走本地代理，避免跨节点访问。 |
| **减少 conntrack 压力** | 跳过 iptables DNAT 和 UDP conntrack，降低表溢出风险。 |
| **提高稳定性** | DNS 查询可升级为 TCP，减少 UDP 丢包导致的超时。 |
| **增强可观测性** | 在节点级别收集 DNS 请求指标，便于排查问题。 |
## DNS 查询路径图
直观展示 Pod → kubelet → NodeLocal DNSCache → CoreDNS 的交互流程：  
```mermaid
flowchart LR
    P[Pod] --> K[kubelet<br/>配置 /etc/resolv.conf]
    K --> N[NodeLocal DNSCache<br/>本地缓存代理 (169.254.20.10)]
    N --> C[CoreDNS/kube-dns Service<br/>集群级 DNS]
    C --> N
    N --> P
```
📑 图解说明
- **Pod**：在启动时，kubelet 为其生成 `/etc/resolv.conf`，DNS 指向 NodeLocal DNSCache 的本地 IP。  
- **kubelet**：负责为 Pod 注入 DNS 配置，确保查询走本地代理。  
- **NodeLocal DNSCache**：运行在每个节点上，先尝试缓存命中；未命中时转发到 CoreDNS。  
- **CoreDNS/kube-dns**：集群级 DNS 服务，负责最终解析域名。  
- **返回路径**：CoreDNS 返回结果 → NodeLocal DNSCache 缓存 → Pod。  

这样你就能直观理解：**Pod 的 DNS 查询先走 kubelet 配置的本地代理，再由 NodeLocal DNSCache 缓存或转发到 CoreDNS，最后返回结果给 Pod**。  

## ⚠️ 注意事项
- 部署 NodeLocal DNSCache 后，必须确保 kubelet 的 `--cluster-dns` 参数正确指向本地缓存 IP，否则 Pod 仍会走 kube-dns Service。  
- 如果节点上 containerd/kubelet 重启，NodeLocal DNSCache Pod 也会短暂不可用，Pod 的 DNS 查询可能回退到 kube-dns Service。  
- 建议在大规模集群或高 QPS DNS 查询场景启用，以避免 DNS 成为瓶颈。  

✅ 总结：**kubelet 决定 Pod 的 DNS 配置，NodeLocal DNSCache 提供本地缓存代理。两者结合后，Pod 的 DNS 查询路径由 kubelet 指向本地代理，从而提升性能和稳定性。**  

# 
