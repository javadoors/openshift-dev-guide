# CoreDNS

**CoreDNS 在 Kubernetes 中的域名功能范围主要覆盖集群内的 Service、Pod、命名空间以及集群域后缀，不涉及 NodeName。它既能解析内部服务域名，也能转发外部域名查询。**

## 📌 CoreDNS 域名功能范围

- **Service 域名**  
  格式：`<service>.<namespace>.svc.<cluster-domain>`  
  例如：`nginx.default.svc.cluster.local`  
  用于服务发现，Pod 通过域名访问 Service。

- **Pod 域名**  
  格式：`<pod-ip>.<namespace>.pod.<cluster-domain>`  
  例如：`10-244-1-5.default.pod.cluster.local`  
  用于直接解析 Pod IP（通常不推荐，更多用于调试）。

- **Namespace**  
  命名空间隔离域名空间，避免不同应用的 Service 名称冲突。

- **Cluster Domain**  
  由 kubelet 的 `--cluster-domain` 参数决定，默认是 `cluster.local`。  
  所有 Service 和 Pod 域名都会以此为后缀。

- **外部域名解析**  
  CoreDNS 会将非集群域名的查询转发到上游 DNS（如系统配置的 DNS 或自定义的外部服务器）。

## 📊 CoreDNS 支持的记录类型

| 记录类型 | 用途 | 示例 |
|----------|------|------|
| **A/AAAA** | IPv4/IPv6 地址解析 | `nginx.default.svc.cluster.local → 10.96.0.15` |
| **SRV** | 服务端口解析 | `_http._tcp.nginx.default.svc.cluster.local` |
| **PTR** | 反向解析 Pod IP | `10-244-1-5.default.pod.cluster.local` |
| **CNAME** | 别名映射 | 自定义域名映射到 Service |

## 🔑 关键结论
- CoreDNS 的域名功能范围覆盖 **Service、Pod、Namespace、Cluster Domain**。  
- **NodeName 不会生成 DNS 域名**，它仅在 API Server 中标识 Pod 所在节点。  
- CoreDNS 还能处理 **外部域名解析**，通过上游 DNS 转发。  
- 可通过 **Corefile 配置**扩展功能，例如 `hosts` 插件添加节点域名，或 `forward` 插件转发外部查询。
-   [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/) 

---

要不要我帮你画一张 **Mermaid 图**，展示 Pod 发起 DNS 查询时，CoreDNS 如何区分内部 Service 域名和外部域名，并标注 NodeName 在 API Server 中的作用但不进入 DNS？
