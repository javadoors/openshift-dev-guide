# beadmConfigSpec:apiServer: certSANs
在 `kubeadm` 的配置文件（`ClusterConfiguration` 或 `InitConfiguration`）里，`apiServer.certSANs` 字段的作用是：**为 API Server 证书添加额外的 Subject Alternative Names (SANs)**。  
## 🔧 为什么需要 `certSANs`
- **API Server 默认证书**：当你用 `kubeadm init` 初始化集群时，API Server 会生成一个 TLS 证书，默认只包含主机名和 IP。
- **多入口访问**：如果你需要通过不同的域名、IP 或负载均衡地址访问 API Server，而这些地址不在默认证书的 SAN 列表里，就会出现 TLS 校验错误。
- **解决办法**：在 `certSANs` 中显式添加这些额外的域名或 IP，保证客户端（如 `kubectl`、kubelet、控制器）连接时不会报证书无效。
## 📄 配置示例
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  certSANs:
  - "127.0.0.1"
  - "10.0.0.1"
  - "k8s-api.example.com"
  - "loadbalancer.example.com"
```
这样生成的 API Server 证书就会包含这些 SANs，允许你通过这些地址安全访问。
## 🧠 使用场景
- **多节点高可用集群**：需要通过负载均衡 VIP 或域名访问 API Server
- **开发/测试环境**：常用 `127.0.0.1` 或自定义域名访问
- **跨网络访问**：不同子网或外部客户端需要用不同 IP/域名访问 API Server
## 🚨 注意事项
- 如果忘记配置 `certSANs`，后续通过非默认地址访问 API Server 时会遇到 TLS 校验失败。
- 修改后需要重新生成证书，可以用：
  ```bash
  kubeadm init phase certs apiserver --config kubeadm.yaml
  ```
- 证书更新后，记得重启 API Server Pod。

总结：**`certSANs` 就是告诉 kubeadm 在生成 API Server 证书时，把额外的 IP/域名也加入到证书的 SAN 列表里，确保多入口访问时 TLS 校验通过。**


要不要我帮你写一个 **高可用集群的 kubeadm 配置模板**，里面预置好 `certSANs`，适合负载均衡场景？
