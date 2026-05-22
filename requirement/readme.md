# 需求
基于Cluster-API完成如下需求，请给出方案设计：
1. 用户提供机器列表(机器已完成OS的安装)，提供主机名、IP、用户名与密码等基本信息
2. 用户提供如下服务：NFS、NTP、镜像仓库、二进制安装源、chart仓库、外部负载均衡器
3. K8s控制面有如下要求：
   - etcd支持外接配置
   - etcd可通过标签指定要安装的节点，并进行自动化安装
   - api-server可通过标签指定要安装的节点，支持扩缩容,并可指定负载均衡器
   - scheduler可通过标签指定要安装的节点，支持主备
   - controller-manager可通过标签指定要安装的节点，支持主备
   - kubelet对某些节点的配置需要支持定制化
   - containerd对某些节点的配置需要支持定制化
   - calico可通过cluster-api的扩展机制进行安装
   - coredns可通过cluster-api的扩展机制进行安装
   - kube-proxy 可通过cluster-api的扩展机制进行安装
4. 其它应用可通过cluster-api的扩展机制进行安装
   - 支持按照应用间的依赖关系进行拓扑化安装
