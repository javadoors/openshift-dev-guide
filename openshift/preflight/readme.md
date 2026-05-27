# 预检工具
在 **OpenShift 安装集群之前**，确实有预检工具和机制，用来确保环境满足安装要求，避免后续部署失败。它们主要分为以下几类：  

## 📌 OpenShift 安装前的预检工具与机制
- **openshift-install 内置检查**  
  - 当你运行 `openshift-install create cluster` 时，安装器会自动检查部分环境条件，例如：DNS 解析、云平台 API 权限、Pull Secret 是否有效。  
  - 如果条件不满足，会直接报错并阻止安装。  
- **oc adm diagnostics**  
  - 在 OpenShift 3.x 中提供的诊断工具，可以检查节点配置、网络、权限等。  
  - 在 OpenShift 4.x 中部分功能被替代，但仍有类似的健康检查命令。  
- **Preflight Checks**  
  - 在 OpenShift 4.x 安装文档中，官方推荐在安装前进行一系列手动或自动检查：  
    - 确认操作系统版本（RHEL CoreOS 或兼容版本）。  
    - 确认 CPU、内存、磁盘空间满足最低要求。  
    - 确认网络连通性、DNS、NTP 时钟同步。  
    - 确认防火墙和端口开放情况。  
- **Red Hat OpenShift Preflight CLI**  
  - 主要用于 **镜像和 Operator 的认证检查**，不是集群安装必需，但在 ISV/企业场景下常用。  

## ⚖️ 典型安装前检查流程
1. **环境检查**：操作系统、硬件资源、NTP、SELinux/AppArmor。  
2. **网络检查**：节点间连通性、DNS、端口、防火墙。  
3. **依赖检查**：Pull Secret、云平台 API 权限、镜像仓库可访问性。  
4. **安装器预检**：运行 `openshift-install`，自动验证配置文件和环境。  

## 🚀 总结
- OpenShift 在安装集群前有 **内置预检机制**（`openshift-install`）和 **诊断工具**（如 `oc adm diagnostics`）。  
- 官方文档也提供了 **Preflight Checks 列表**，涵盖环境、网络、依赖、安全等方面。  
- 在企业场景下，还可以结合 **Preflight CLI** 做镜像和 Operator 的合规检查。  
