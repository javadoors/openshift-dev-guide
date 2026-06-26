# TLS SAN
在 **containerd** 中配置 **TLS SAN (Subject Alternative Name)**，主要用于它作为 **gRPC 服务端**时，确保客户端（如 `ctr`、`crictl`、kubelet）在 TLS 握手时能正确验证服务端证书。  

## 📌 配置位置
- 在 `config.toml` 中的 **[grpc]** 段落下配置：  
```toml
[grpc]
  tls_cert = "/etc/containerd/certs/containerd.crt"
  tls_key  = "/etc/containerd/certs/containerd.key"
  tls_san  = ["127.0.0.1", "myhost.local", "containerd.example.com"]
```
- `tls_san` 字段指定证书的 SAN 列表。  

## 🏗️ 通信对象
- **本地客户端**：如 `ctr`、`crictl`，通过 socket 或 TCP 地址连接。  
- **kubelet**：在 Kubernetes 集群中，kubelet 通过 CRI 与 containerd 通信，需验证 SAN。  
- **远程 API 客户端**：跨主机管理工具或监控系统访问时，SAN 必须包含远程域名/IP。  

## 📊 与镜像仓库 TLS 的区别
- **containerd 主配置的 SAN**：仅用于 gRPC 服务端证书校验。  
- **镜像仓库 TLS 配置**：在 `/etc/containerd/certs.d/<registry>/hosts.toml` 中单独配置，用于拉取镜像时验证仓库证书。  

## ⚠️ 注意事项
- **域名/IP 必须匹配**：客户端连接时使用的地址必须在 SAN 中，否则报错：  
  `x509: certificate is not valid for any names`。  
- **修改 SAN 后需重建证书**：并重启 containerd。  
- **区分服务端与客户端**：containerd 的 SAN 配置是服务端证书，仓库 TLS 校验是客户端证书。  

✅ 总结：**containerd 的 TLS SAN 配置用于 gRPC 服务端证书校验，确保 ctr、crictl、kubelet 或远程客户端能正确验证服务端身份。镜像仓库的 TLS 校验则在 hosts.toml 中单独配置，不依赖 containerd 主配置的 SAN。**  


