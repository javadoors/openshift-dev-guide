# 网络插件刷新导致 kubelet 心跳中断的详细原理

Calico 初始化时确实会导致 **宿主机网络栈刷新**，而这会间接影响到 kubelet 与 API Server 的心跳。下面是详细原理拆解：  

## 📖 原理机制
- **CNI 配置刷新**  
  - Calico 在 `/etc/cni/net.d/` 写入新的配置文件，替换原有的 Pod 网络插件配置。  
  - kubelet 在创建 Pod 时会调用 CNI 插件，如果配置文件更新，kubelet 会重新加载。  

- **iptables 规则重建**  
  - Felix 组件会安装大量 iptables 规则，用于 NAT、流量转发和 NetworkPolicy。  
  - 在规则重建过程中，宿主机的转发链可能短暂不可用，导致 kubelet 与 API Server 的 TCP 连接中断。  

- **路由表更新**  
  - Calico 为 Pod CIDR 添加路由条目，若启用 BGP，还会建立邻居关系。  
  - 路由表刷新时，宿主机的网络路径可能暂时不可达。  

- **网络设备创建**  
  - 创建 veth pair、隧道设备（IP-in-IP/VXLAN），这些操作会修改宿主机的网络接口。  
  - 在设备切换过程中，宿主机网络栈可能出现短暂抖动。  

## ⚡ kubelet 心跳中断的过程
1. kubelet 与 API Server 通过宿主机网络保持心跳。  
2. Calico 初始化时修改 iptables/路由/设备，导致宿主机网络栈短暂不可用。  
3. kubelet 心跳失败，API Server 将节点标记为 `NotReady`。  
4. 网络稳定后，心跳恢复，节点重新变为 `Ready`。  

## 🎯 总结
- Calico 初始化会修改宿主机网络（iptables、路由、设备、CNI 配置）。  
- 这些修改过程可能导致 **宿主机网络短暂中断**。  
- kubelet 与 API Server 的心跳依赖宿主机网络，因此会出现 **心跳中断 → 节点 NotReady → 恢复** 的现象。  
