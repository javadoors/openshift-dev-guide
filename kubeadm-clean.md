# 载整个 Kubernetes 集群
要卸载整个 Kubernetes 集群，需要区分你是 **用 kubeadm 搭建的集群**，还是通过 **Cluster API/其它工具** 部署的。根据你给出的环境（节点显示 `control-plane`，操作系统 openEuler/Ubuntu，容器运行时 containerd），看起来是一个 **kubeadm 初始化的集群**。下面给出标准卸载流程：
## 📌 卸载步骤（kubeadm 集群）
### 1. 在所有节点上停止并清理 kubelet
```bash
sudo systemctl stop kubelet
sudo systemctl disable kubelet
```
### 2. 在所有节点上重置 kubeadm
```bash
sudo kubeadm reset -f
```
- 这会清理集群的配置、证书、网络规则。  
- `-f` 表示强制执行，不需要交互确认。
### 3. 清理 CNI 网络配置
```bash
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /var/lib/etcd
```
### 4. 删除集群相关的二进制和配置
```bash
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes
```
### 5. 在所有节点上清理容器运行时（containerd）
```bash
sudo systemctl stop containerd
sudo systemctl disable containerd
sudo rm -rf /var/lib/containerd
```
## 📊 注意事项
- 如果你只想删除 **集群对象** 而保留节点，可以直接 `kubectl delete node <name>`。  
- 如果你要彻底卸载整个集群（包括控制平面和工作节点），必须在所有节点上执行 `kubeadm reset`。  
- 如果是云环境（AWS、Azure 等），还需要在云平台上删除虚拟机和网络资源。  

✅ 总结：  
- **卸载集群** = 在所有节点上执行 `kubeadm reset` + 清理网络和配置。  
- **彻底清理** = 同时删除 containerd 数据目录和 Kubernetes 配置。  

# 扩展版的卸载脚本
不仅清理集群配置和网络，还会卸载 Kubernetes 二进制包（kubeadm、kubectl、kubelet）并清理日志，让环境尽可能恢复到“裸机”状态。  
```bash
#!/bin/bash
# Kubernetes 集群彻底卸载脚本
# 在所有节点（控制平面和工作节点）上执行

set -e

echo "=== 停止 kubelet 和 containerd ==="
sudo systemctl stop kubelet || true
sudo systemctl disable kubelet || true
sudo systemctl stop containerd || true
sudo systemctl disable containerd || true

echo "=== kubeadm reset ==="
sudo kubeadm reset -f

echo "=== 清理 CNI 网络配置 ==="
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /var/lib/etcd

echo "=== 删除 Kubernetes 配置文件 ==="
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes

echo "=== 删除 containerd 数据目录 ==="
sudo rm -rf /var/lib/containerd

echo "=== 清理 iptables 规则 ==="
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X

echo "=== 清理网络接口 ==="
sudo ip link delete cni0 || true
sudo ip link delete flannel.1 || true

echo "=== 卸载 Kubernetes 二进制包 ==="
# 针对 Debian/Ubuntu
if command -v apt-get >/dev/null 2>&1; then
  sudo apt-get remove -y kubeadm kubectl kubelet
  sudo apt-get autoremove -y
fi

# 针对 CentOS/RHEL/Fedora
if command -v yum >/dev/null 2>&1; then
  sudo yum remove -y kubeadm kubectl kubelet
fi
if command -v dnf >/dev/null 2>&1; then
  sudo dnf remove -y kubeadm kubectl kubelet
fi

echo "=== 清理日志文件 ==="
sudo rm -rf /var/log/pods
sudo rm -rf /var/log/containers
sudo rm -rf /var/log/kube*
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

echo "=== 卸载完成，节点已恢复到裸机状态 ==="
```
## 📊 脚本功能总结
- **集群清理**：`kubeadm reset` + 删除 etcd、CNI、kubelet 数据。  
- **网络清理**：iptables 规则、CNI 虚拟网卡。  
- **运行时清理**：删除 containerd 数据目录。  
- **二进制卸载**：移除 kubeadm、kubectl、kubelet。  
- **日志清理**：清理 Pod/容器日志，压缩并清理 systemd 日志。  

✅ 执行后，你的节点会尽可能恢复到“裸机”状态，可以重新部署新的 Kubernetes 集群或其他服务。  

## Dockerfile/Ansible Playbook
### 📌 Dockerfile 版本
这个 Dockerfile 会构建一个镜像，包含卸载脚本，可以在容器里运行或挂载到宿主机执行。
```dockerfile
FROM ubuntu:22.04

# 安装必要工具
RUN apt-get update && apt-get install -y \
    iproute2 iptables systemd \
    kubeadm kubectl kubelet \
    containerd \
    && apt-get clean

# 添加卸载脚本
COPY reset-k8s.sh /usr/local/bin/reset-k8s.sh
RUN chmod +x /usr/local/bin/reset-k8s.sh

ENTRYPOINT ["/usr/local/bin/reset-k8s.sh"]
```

> 使用方法：  
> 1. 将之前写好的 `reset-k8s.sh` 脚本放在同目录。  
> 2. 构建镜像：`docker build -t k8s-reset .`  
> 3. 在目标节点运行容器并挂载宿主机：`docker run --rm --privileged -v /:/host k8s-reset`  
### 📌 Ansible Playbook 版本
这个 Playbook 可以在多节点环境里批量执行卸载。
```yaml
---
- name: Reset Kubernetes cluster
  hosts: all
  become: yes
  tasks:
    - name: Stop kubelet and containerd
      service:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - kubelet
        - containerd

    - name: Run kubeadm reset
      command: kubeadm reset -f

    - name: Remove Kubernetes directories
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/cni/net.d
        - /var/lib/cni/
        - /var/lib/kubelet
        - /var/lib/etcd
        - /etc/kubernetes
        - /var/lib/containerd
        - ~/.kube

    - name: Flush iptables rules
      command: iptables -F

    - name: Remove CNI interfaces
      command: ip link delete {{ item }}
      ignore_errors: yes
      loop:
        - cni0
        - flannel.1

    - name: Uninstall Kubernetes binaries (Debian/Ubuntu)
      apt:
        name: "{{ item }}"
        state: absent
      loop:
        - kubeadm
        - kubectl
        - kubelet
      when: ansible_os_family == "Debian"

    - name: Uninstall Kubernetes binaries (RedHat)
      yum:
        name: "{{ item }}"
        state: absent
      loop:
        - kubeadm
        - kubectl
        - kubelet
      when: ansible_os_family == "RedHat"

    - name: Clean logs
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/log/pods
        - /var/log/containers
        - /var/log/kube*
```
 ✅ 总结
- **Dockerfile**：适合容器化环境，单节点执行。  
- **Ansible Playbook**：适合多节点批量卸载，自动化程度高。  

这样你就可以选择在不同场景下使用不同的工具来彻底卸载 Kubernetes 集群。  
