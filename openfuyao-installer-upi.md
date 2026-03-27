# upi
# openFuyao UPI自动化安装方案
基于Ubuntu操作系统且无法使用Ignition的场景，我为您重新设计openFuyao UPI自动化安装方案。
## openFuyao UPI自动化安装方案（Ubuntu场景）
### 一、总体架构设计
#### 1.1 Asset-Based架构（保留核心设计）
```go
package asset

type Asset interface {
    Dependencies() []Asset
    Generate(ctx context.Context, parents Parents) error
    Name() string
}

type WritableAsset interface {
    Asset
    Files() []*File
    Load(FileFetcher) (found bool, err error)
}
```
#### 1.2 Ubuntu场景Asset清单
```go
var UPIUbuntuAssets = []asset.WritableAsset{
    // 1. 配置类Asset
    &InstallConfig{},
    &ClusterID{},
    &NetworkingConfig{},
    &DNSConfig{},
    
    // 2. 证书类Asset
    &RootCA{},
    &EtcdCA{},
    &KubeCA{},
    &FrontProxyCA{},
    &ServiceAccountKey{},
    &APIServerCert{},
    &KubeletCert{},
    &EtcdCert{},
    
    // 3. Kubeconfig类Asset
    &AdminKubeconfig{},
    &KubeletKubeconfig{},
    &ControllerManagerKubeconfig{},
    &SchedulerKubeconfig{},
    
    // 4. Ubuntu节点配置Asset（替代Ignition）
    &CloudInitConfig{},          // Cloud-Init配置
    &NodeSetupScripts{},         // 节点设置脚本
    &KubeadmInitConfig{},        // Kubeadm初始化配置
    &KubeadmJoinConfig{},        // Kubeadm Join配置
    
    // 5. 清单类Asset
    &KubeadmConfig{},
    &MachineConfigs{},
    &AddonManifests{},
    &CNIManifests{},
    &CSIManifests{},
    
    // 6. 状态类Asset
    &ClusterMetadata{},
    &InstallState{},
    &NodeInventory{},            // 节点清单
}
```
### 二、核心Asset设计详解
#### 2.1 InstallConfig Asset
```go
package installconfig

type InstallConfig struct {
    Config *v1beta1.BKECluster
}

type NodeInfo struct {
    Hostname string   `json:"hostname"`
    IP       string   `json:"ip"`
    Roles    []string `json:"roles"` // master, worker, etcd
    SSHKey   string   `json:"sshKey,omitempty"`
    SSHUser  string   `json:"sshUser"`
}

func (i *InstallConfig) Dependencies() []asset.Asset {
    return []asset.Asset{
        &BaseDomain{},
        &ClusterName{},
        &Platform{},
        &Networking{},
        &NodeInventory{},  // 节点清单
    }
}

func (i *InstallConfig) Generate(ctx context.Context, parents asset.Parents) error {
    baseDomain := &BaseDomain{}
    clusterName := &ClusterName{}
    platform := &Platform{}
    networking := &Networking{}
    nodeInventory := &NodeInventory{}
    parents.Get(baseDomain, clusterName, platform, networking, nodeInventory)
    
    i.Config = &v1beta1.BKECluster{
        ObjectMeta: metav1.ObjectMeta{
            Name: clusterName.Name,
        },
        Spec: v1beta1.BKEClusterSpec{
            ClusterConfig: v1beta1.ClusterConfig{
                BaseDomain: baseDomain.Domain,
                Platform:   platform.Type,
                Networking: networking.Config,
            },
            Nodes: nodeInventory.Nodes,
        },
    }
    
    defaults.SetInstallConfigDefaults(i.Config)
    
    if err := validation.ValidateInstallConfig(i.Config); err != nil {
        return err
    }
    
    return nil
}
```
#### 2.2 Cloud-Init配置Asset（替代Ignition）
```go
package cloudinit

type CloudInitConfig struct {
    MasterConfig *CloudInitMaster
    WorkerConfig *CloudInitWorker
    Files        []*asset.File
}

type CloudInitMaster struct {
    Hostname    string
    IP          string
    APIServer   string
    ClusterName string
    Certs       map[string][]byte
    Kubeconfigs map[string][]byte
}

func (c *CloudInitConfig) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &tls.RootCA{},
        &tls.KubeCA{},
        &tls.EtcdCA{},
        &tls.ServiceAccountKey{},
        &kubeconfig.KubeletKubeconfig{},
        &kubeconfig.AdminKubeconfig{},
    }
}

func (c *CloudInitConfig) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    rootCA := &tls.RootCA{}
    kubeCA := &tls.KubeCA{}
    etcdCA := &tls.EtcdCA{}
    parents.Get(installConfig, rootCA, kubeCA, etcdCA)
    
    // 为每个Master节点生成Cloud-Init配置
    for _, node := range installConfig.Config.Spec.Nodes {
        if contains(node.Roles, "master") {
            masterConfig := &CloudInitMaster{
                Hostname:    node.Hostname,
                IP:          node.IP,
                APIServer:   installConfig.Config.Spec.ControlPlaneEndpoint,
                ClusterName: installConfig.Config.Name,
                Certs: map[string][]byte{
                    "ca.crt":     rootCA.Cert(),
                    "ca.key":     rootCA.Key(),
                    "etcd-ca.crt": etcdCA.Cert(),
                    "etcd-ca.key": etcdCA.Key(),
                    "kube-ca.crt": kubeCA.Cert(),
                    "kube-ca.key": kubeCA.Key(),
                },
            }
            
            // 生成Cloud-Init YAML
            cloudInitYAML := c.generateMasterCloudInit(masterConfig)
            
            c.Files = append(c.Files, &asset.File{
                Filename: fmt.Sprintf("cloud-init/%s-master.yaml", node.Hostname),
                Data:     cloudInitYAML,
            })
        }
    }
    
    // 为每个Worker节点生成Cloud-Init配置
    for _, node := range installConfig.Config.Spec.Nodes {
        if contains(node.Roles, "worker") {
            workerConfig := &CloudInitWorker{
                Hostname:  node.Hostname,
                IP:        node.IP,
                APIServer: installConfig.Config.Spec.ControlPlaneEndpoint,
            }
            
            cloudInitYAML := c.generateWorkerCloudInit(workerConfig)
            
            c.Files = append(c.Files, &asset.File{
                Filename: fmt.Sprintf("cloud-init/%s-worker.yaml", node.Hostname),
                Data:     cloudInitYAML,
            })
        }
    }
    
    return nil
}

func (c *CloudInitConfig) generateMasterCloudInit(master *CloudInitMaster) []byte {
    cloudInit := map[string]interface{}{
        "hostname": master.Hostname,
        "manage_etc_hosts": true,
        "package_update": true,
        "package_upgrade": true,
        "packages": []string{
            "apt-transport-https",
            "ca-certificates",
            "curl",
            "gnupg",
            "lsb-release",
            "containerd",
            "kubelet",
            "kubeadm",
            "kubectl",
        },
        "write_files": []map[string]interface{}{
            {
                "path": "/etc/kubernetes/pki/ca.crt",
                "content": string(master.Certs["ca.crt"]),
                "permissions": "0644",
            },
            {
                "path": "/etc/kubernetes/pki/ca.key",
                "content": string(master.Certs["ca.key"]),
                "permissions": "0600",
            },
            {
                "path": "/etc/kubernetes/pki/etcd/ca.crt",
                "content": string(master.Certs["etcd-ca.crt"]),
                "permissions": "0644",
            },
            {
                "path": "/etc/kubernetes/pki/etcd/ca.key",
                "content": string(master.Certs["etcd-ca.key"]),
                "permissions": "0600",
            },
            {
                "path": "/etc/kubernetes/kubeadm-config.yaml",
                "content": c.generateKubeadmConfig(master),
                "permissions": "0644",
            },
            {
                "path": "/etc/containerd/config.toml",
                "content": c.generateContainerdConfig(),
                "permissions": "0644",
            },
        },
        "runcmd": []string{
            "systemctl enable containerd",
            "systemctl start containerd",
            "systemctl enable kubelet",
            "kubeadm init --config /etc/kubernetes/kubeadm-config.yaml",
            "mkdir -p $HOME/.kube",
            "cp -i /etc/kubernetes/admin.conf $HOME/.kube/config",
            "chown $(id -u):$(id -g) $HOME/.kube/config",
        },
    }
    
    data, _ := yaml.Marshal(cloudInit)
    return data
}
```
#### 2.3 节点设置脚本Asset（替代Ignition）
```go
package nodesetup

type NodeSetupScripts struct {
    MasterScript []byte
    WorkerScript []byte
    Files        []*asset.File
}

func (n *NodeSetupScripts) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &tls.RootCA{},
        &tls.KubeCA{},
        &tls.EtcdCA{},
        &kubeconfig.KubeletKubeconfig{},
    }
}

func (n *NodeSetupScripts) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    rootCA := &tls.RootCA{}
    kubeCA := &tls.KubeCA{}
    etcdCA := &tls.EtcdCA{}
    parents.Get(installConfig, rootCA, kubeCA, etcdCA)
    
    // 生成Master节点设置脚本
    masterScript := n.generateMasterSetupScript(installConfig, rootCA, kubeCA, etcdCA)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/setup-master.sh",
        Data:     masterScript,
    })
    
    // 生成Worker节点设置脚本
    workerScript := n.generateWorkerSetupScript(installConfig, rootCA, kubeCA)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/setup-worker.sh",
        Data:     workerScript,
    })
    
    // 生成通用配置脚本
    commonScript := n.generateCommonSetupScript()
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/setup-common.sh",
        Data:     commonScript,
    })
    
    return nil
}

func (n *NodeSetupScripts) generateMasterSetupScript(
    config *installconfig.InstallConfig,
    rootCA *tls.RootCA,
    kubeCA *tls.KubeCA,
    etcdCA *tls.EtcdCA,
) []byte {
    script := `#!/bin/bash
set -euo pipefail

MASTER_SETUP_SCRIPT

echo "=== BKE Master Node Setup ==="
echo "Hostname: $(hostname)"
echo "IP: $(hostname -I | awk '{print $1}')"

# 1. 配置系统参数
echo "Configuring system parameters..."
cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 2. 关闭Swap
echo "Disabling swap..."
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 3. 加载内核模块
echo "Loading kernel modules..."
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# 4. 安装Containerd
echo "Installing containerd..."
if ! command -v containerd &> /dev/null; then
    apt-get update
    apt-get install -y containerd
fi

# 5. 配置Containerd
echo "Configuring containerd..."
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# 配置SystemdCgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# 6. 安装Kubernetes组件
echo "Installing Kubernetes components..."
if ! command -v kubeadm &> /dev/null; then
    # 添加Kubernetes仓库
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
    
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
fi

# 7. 创建证书目录
echo "Creating certificate directories..."
mkdir -p /etc/kubernetes/pki
mkdir -p /etc/kubernetes/pki/etcd

# 8. 写入证书
echo "Writing certificates..."
cat > /etc/kubernetes/pki/ca.crt <<EOFCERT
` + string(rootCA.Cert()) + `
EOFCERT

cat > /etc/kubernetes/pki/ca.key <<EOFCERT
` + string(rootCA.Key()) + `
EOFCERT

cat > /etc/kubernetes/pki/etcd/ca.crt <<EOFCERT
` + string(etcdCA.Cert()) + `
EOFCERT

cat > /etc/kubernetes/pki/etcd/ca.key <<EOFCERT
` + string(etcdCA.Key()) + `
EOFCERT

# 9. 创建Kubeadm配置
echo "Creating kubeadm configuration..."
cat > /etc/kubernetes/kubeadm-config.yaml <<EOFKUBEADM
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: ` + config.Config.Spec.KubernetesVersion + `
controlPlaneEndpoint: "` + config.Config.Spec.ControlPlaneEndpoint + `"
networking:
  dnsDomain: ` + config.Config.Spec.Networking.DNSDomain + `
  podSubnet: ` + config.Config.Spec.Networking.PodSubnet + `
  serviceSubnet: ` + config.Config.Spec.Networking.ServiceSubnet + `
etcd:
  local:
    dataDir: /var/lib/etcd
    serverCertSANs:
      - "` + config.Config.Spec.ControlPlaneEndpoint + `"
    peerCertSANs:
      - "` + config.Config.Spec.ControlPlaneEndpoint + `"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $(hostname -I | awk '{print $1}')
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  kubeletExtraArgs:
    cgroup-driver: systemd
EOFKUBEADM

# 10. 启动Kubelet
echo "Starting kubelet..."
systemctl enable kubelet
systemctl start kubelet

# 11. 执行Kubeadm Init（仅在第一个Master节点）
if [ ! -f /etc/kubernetes/admin.conf ]; then
    echo "Initializing Kubernetes cluster..."
    kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --upload-certs
    
    # 配置kubectl
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    
    # 生成Join命令
    kubeadm token create --print-join-command > /tmp/kubeadm-join.sh
    chmod +x /tmp/kubeadm-join.sh
fi

echo "=== Master node setup completed ==="
`
    return []byte(script)
}

func (n *NodeSetupScripts) generateWorkerSetupScript(
    config *installconfig.InstallConfig,
    rootCA *tls.RootCA,
    kubeCA *tls.KubeCA,
) []byte {
    script := `#!/bin/bash
set -euo pipefail

WORKER_SETUP_SCRIPT

echo "=== BKE Worker Node Setup ==="
echo "Hostname: $(hostname)"
echo "IP: $(hostname -I | awk '{print $1}')"

# 1-6: 与Master相同的系统配置（略）
# ...

# 7. 创建证书目录
echo "Creating certificate directories..."
mkdir -p /etc/kubernetes/pki

# 8. 写入CA证书
echo "Writing CA certificates..."
cat > /etc/kubernetes/pki/ca.crt <<EOFCERT
` + string(rootCA.Cert()) + `
EOFCERT

# 9. 等待Join命令可用
echo "Waiting for join command..."
while [ ! -f /tmp/kubeadm-join.sh ]; do
    sleep 5
done

# 10. 执行Join
echo "Joining cluster..."
/tmp/kubeadm-join.sh

echo "=== Worker node setup completed ==="
`
    return []byte(script)
}
```
#### 2.4 Kubeadm配置Asset
```go
package kubeadm

type KubeadmInitConfig struct {
    Config *kubeadmv1beta3.ClusterConfiguration
    Files  []*asset.File
}

func (k *KubeadmInitConfig) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &tls.RootCA{},
        &tls.EtcdCA{},
        &tls.ServiceAccountKey{},
    }
}

func (k *KubeadmInitConfig) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    rootCA := &tls.RootCA{}
    etcdCA := &tls.EtcdCA{}
    saKey := &tls.ServiceAccountKey{}
    parents.Get(installConfig, rootCA, etcdCA, saKey)
    
    k.Config = &kubeadmv1beta3.ClusterConfiguration{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "kubeadm.k8s.io/v1beta3",
            Kind:       "ClusterConfiguration",
        },
        KubernetesVersion: installConfig.Config.Spec.KubernetesVersion,
        ControlPlaneEndpoint: installConfig.Config.Spec.ControlPlaneEndpoint,
        Networking: kubeadmv1beta3.Networking{
            DNSDomain:     installConfig.Config.Spec.Networking.DNSDomain,
            PodSubnet:     installConfig.Config.Spec.Networking.PodSubnet,
            ServiceSubnet: installConfig.Config.Spec.Networking.ServiceSubnet,
        },
        Etcd: kubeadmv1beta3.Etcd{
            Local: &kubeadmv1beta3.LocalEtcd{
                DataDir: "/var/lib/etcd",
                ServerCertSANs: []string{
                    installConfig.Config.Spec.ControlPlaneEndpoint,
                },
                PeerCertSANs: []string{
                    installConfig.Config.Spec.ControlPlaneEndpoint,
                },
            },
        },
        CertificatesDir: "/etc/kubernetes/pki",
        ImageRepository: installConfig.Config.Spec.ImageRepo.Domain + "/kubernetes",
    }
    
    data, _ := yaml.Marshal(k.Config)
    k.Files = []*asset.File{
        {
            Filename: "kubeadm/kubeadm-init.yaml",
            Data:     data,
        },
    }
    
    return nil
}

type KubeadmJoinConfig struct {
    Config *kubeadmv1beta3.JoinConfiguration
    Files  []*asset.File
}

func (k *KubeadmJoinConfig) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &tls.RootCA{},
    }
}

func (k *KubeadmJoinConfig) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    rootCA := &tls.RootCA{}
    parents.Get(installConfig, rootCA)
    
    k.Config = &kubeadmv1beta3.JoinConfiguration{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "kubeadm.k8s.io/v1beta3",
            Kind:       "JoinConfiguration",
        },
        Discovery: kubeadmv1beta3.Discovery{
            BootstrapToken: &kubeadmv1beta3.BootstrapTokenDiscovery{
                APIServerEndpoint: installConfig.Config.Spec.ControlPlaneEndpoint,
                Token:             "", // 将在运行时生成
                CACertHashes: []string{
                    "sha256:" + rootCA.Hash(),
                },
            },
        },
        NodeRegistration: kubeadmv1beta3.NodeRegistrationOptions{
            CRISocket: "unix:///var/run/containerd/containerd.sock",
            KubeletExtraArgs: map[string]string{
                "cgroup-driver": "systemd",
            },
        },
    }
    
    data, _ := yaml.Marshal(k.Config)
    k.Files = []*asset.File{
        {
            Filename: "kubeadm/kubeadm-join.yaml",
            Data:     data,
        },
    }
    
    return nil
}
```
### 三、节点配置执行器设计
#### 3.1 SSH执行器
```go
package executor

type SSHExecutor struct {
    client     *ssh.Client
    host       string
    port       int
    user       string
    privateKey []byte
}

func NewSSHExecutor(host string, port int, user string, privateKey []byte) (*SSHExecutor, error) {
    signer, err := ssh.ParsePrivateKey(privateKey)
    if err != nil {
        return nil, err
    }
    
    config := &ssh.ClientConfig{
        User: user,
        Auth: []ssh.AuthMethod{
            ssh.PublicKeys(signer),
        },
        HostKeyCallback: ssh.InsecureIgnoreHostKey(),
        Timeout:         30 * time.Second,
    }
    
    client, err := ssh.Dial("tcp", fmt.Sprintf("%s:%d", host, port), config)
    if err != nil {
        return nil, err
    }
    
    return &SSHExecutor{
        client:     client,
        host:       host,
        port:       port,
        user:       user,
        privateKey: privateKey,
    }, nil
}

func (e *SSHExecutor) Execute(command string) (string, error) {
    session, err := e.client.NewSession()
    if err != nil {
        return "", err
    }
    defer session.Close()
    
    output, err := session.CombinedOutput(command)
    return string(output), err
}

func (e *SSHExecutor) UploadFile(localPath, remotePath string) error {
    session, err := e.client.NewSession()
    if err != nil {
        return err
    }
    defer session.Close()
    
    // 读取本地文件
    data, err := os.ReadFile(localPath)
    if err != nil {
        return err
    }
    
    // 使用SCP上传
    scpCommand := fmt.Sprintf("cat > %s", remotePath)
    session.Stdin = bytes.NewReader(data)
    
    return session.Run(scpCommand)
}

func (e *SSHExecutor) UploadDirectory(localDir, remoteDir string) error {
    // 使用sftp上传整个目录
    sftpClient, err := sftp.NewClient(e.client)
    if err != nil {
        return err
    }
    defer sftpClient.Close()
    
    return filepath.Walk(localDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        relPath, _ := filepath.Rel(localDir, path)
        remotePath := filepath.Join(remoteDir, relPath)
        
        if info.IsDir() {
            return sftpClient.MkdirAll(remotePath)
        }
        
        localFile, err := os.Open(path)
        if err != nil {
            return err
        }
        defer localFile.Close()
        
        remoteFile, err := sftpClient.Create(remotePath)
        if err != nil {
            return err
        }
        defer remoteFile.Close()
        
        _, err = io.Copy(remoteFile, localFile)
        return err
    })
}

func (e *SSHExecutor) Close() error {
    return e.client.Close()
}
```
#### 3.2 节点配置管理器
```go
package nodemanager

type NodeConfigurator struct {
    installDir string
    state      *state.Manager
}

func (n *NodeConfigurator) ConfigureNode(ctx context.Context, node *NodeInfo, role string) error {
    // 1. 建立SSH连接
    executor, err := executor.NewSSHExecutor(
        node.IP,
        22,
        node.SSHUser,
        []byte(node.SSHKey),
    )
    if err != nil {
        return fmt.Errorf("failed to connect to node %s: %v", node.Hostname, err)
    }
    defer executor.Close()
    
    // 2. 上传配置文件
    if err := n.uploadConfigs(ctx, executor, node, role); err != nil {
        return err
    }
    
    // 3. 上传脚本
    if err := n.uploadScripts(ctx, executor, role); err != nil {
        return err
    }
    
    // 4. 执行配置脚本
    if err := n.executeSetup(ctx, executor, node, role); err != nil {
        return err
    }
    
    // 5. 验证节点状态
    if err := n.verifyNode(ctx, executor, node, role); err != nil {
        return err
    }
    
    return nil
}

func (n *NodeConfigurator) uploadConfigs(ctx context.Context, executor *executor.SSHExecutor, node *NodeInfo, role string) error {
    // 上传证书
    certDir := filepath.Join(n.installDir, "certs")
    if err := executor.UploadDirectory(certDir, "/etc/kubernetes/pki"); err != nil {
        return err
    }
    
    // 上传kubeconfig
    kubeconfigDir := filepath.Join(n.installDir, "kubeconfigs")
    if err := executor.UploadDirectory(kubeconfigDir, "/etc/kubernetes"); err != nil {
        return err
    }
    
    // 上传kubeadm配置
    kubeadmConfig := filepath.Join(n.installDir, fmt.Sprintf("kubeadm/kubeadm-%s.yaml", role))
    if err := executor.UploadFile(kubeadmConfig, "/etc/kubernetes/kubeadm-config.yaml"); err != nil {
        return err
    }
    
    return nil
}

func (n *NodeConfigurator) uploadScripts(ctx context.Context, executor *executor.SSHExecutor, role string) error {
    scriptDir := filepath.Join(n.installDir, "scripts")
    
    // 上传通用脚本
    if err := executor.UploadFile(
        filepath.Join(scriptDir, "setup-common.sh"),
        "/tmp/setup-common.sh",
    ); err != nil {
        return err
    }
    
    // 上传角色特定脚本
    scriptName := fmt.Sprintf("setup-%s.sh", role)
    if err := executor.UploadFile(
        filepath.Join(scriptDir, scriptName),
        fmt.Sprintf("/tmp/%s", scriptName),
    ); err != nil {
        return err
    }
    
    return nil
}

func (n *NodeConfigurator) executeSetup(ctx context.Context, executor *executor.SSHExecutor, node *NodeInfo, role string) error {
    // 执行通用配置
    output, err := executor.Execute("chmod +x /tmp/setup-common.sh && /tmp/setup-common.sh")
    if err != nil {
        return fmt.Errorf("failed to execute common setup: %v\nOutput: %s", err, output)
    }
    
    // 执行角色特定配置
    scriptName := fmt.Sprintf("setup-%s.sh", role)
    output, err = executor.Execute(fmt.Sprintf("chmod +x /tmp/%s && /tmp/%s", scriptName, scriptName))
    if err != nil {
        return fmt.Errorf("failed to execute %s setup: %v\nOutput: %s", role, err, output)
    }
    
    return nil
}

func (n *NodeConfigurator) verifyNode(ctx context.Context, executor *executor.SSHExecutor, node *NodeInfo, role string) error {
    // 检查kubelet状态
    output, err := executor.Execute("systemctl is-active kubelet")
    if err != nil || strings.TrimSpace(output) != "active" {
        return fmt.Errorf("kubelet is not running on node %s", node.Hostname)
    }
    
    // 检查containerd状态
    output, err = executor.Execute("systemctl is-active containerd")
    if err != nil || strings.TrimSpace(output) != "active" {
        return fmt.Errorf("containerd is not running on node %s", node.Hostname)
    }
    
    // 如果是Master节点，检查API Server
    if contains(role, "master") {
        output, err = executor.Execute("curl -k https://localhost:6443/healthz")
        if err != nil || strings.TrimSpace(output) != "ok" {
            return fmt.Errorf("API server is not healthy on node %s", node.Hostname)
        }
    }
    
    return nil
}
```
### 四、UPI场景自动化流程
#### 4.1 主安装流程
```go
package installer

type UPIInstaller struct {
    assetStore      *store.Store
    stateManager    *state.Manager
    nodeConfigurator *nodemanager.NodeConfigurator
    installDir      string
}

func (u *UPIInstaller) Install(ctx context.Context) error {
    // 阶段1: 生成配置资产
    log.Info("Phase 1: Generating configuration assets...")
    if err := u.generateConfigAssets(ctx); err != nil {
        return err
    }
    u.saveProgress("config", 100)
    
    // 阶段2: 生成证书资产
    log.Info("Phase 2: Generating certificate assets...")
    if err := u.generateCertificateAssets(ctx); err != nil {
        return err
    }
    u.saveProgress("certificates", 100)
    
    // 阶段3: 生成Kubeconfig资产
    log.Info("Phase 3: Generating kubeconfig assets...")
    if err := u.generateKubeconfigAssets(ctx); err != nil {
        return err
    }
    u.saveProgress("kubeconfigs", 100)
    
    // 阶段4: 生成Kubeadm配置
    log.Info("Phase 4: Generating kubeadm configurations...")
    if err := u.generateKubeadmConfigs(ctx); err != nil {
        return err
    }
    u.saveProgress("kubeadm", 100)
    
    // 阶段5: 生成节点设置脚本
    log.Info("Phase 5: Generating node setup scripts...")
    if err := u.generateNodeSetupScripts(ctx); err != nil {
        return err
    }
    u.saveProgress("scripts", 100)
    
    // 阶段6: 配置Master节点
    log.Info("Phase 6: Configuring master nodes...")
    if err := u.configureMasterNodes(ctx); err != nil {
        return err
    }
    u.saveProgress("master-nodes", 100)
    
    // 阶段7: 配置Worker节点
    log.Info("Phase 7: Configuring worker nodes...")
    if err := u.configureWorkerNodes(ctx); err != nil {
        return err
    }
    u.saveProgress("worker-nodes", 100)
    
    // 阶段8: 安装网络插件
    log.Info("Phase 8: Installing network plugin...")
    if err := u.installNetworkPlugin(ctx); err != nil {
        return err
    }
    u.saveProgress("network", 100)
    
    // 阶段9: 安装存储插件
    log.Info("Phase 9: Installing storage plugin...")
    if err := u.installStoragePlugin(ctx); err != nil {
        return err
    }
    u.saveProgress("storage", 100)
    
    // 阶段10: 等待集群就绪
    log.Info("Phase 10: Waiting for cluster to be ready...")
    if err := u.waitForClusterReady(ctx); err != nil {
        return err
    }
    u.saveProgress("cluster-ready", 100)
    
    log.Info("Installation completed successfully!")
    return nil
}
```
#### 4.2 Master节点配置流程
```go
func (u *UPIInstaller) configureMasterNodes(ctx context.Context) error {
    installConfig := u.getInstallConfig()
    masterNodes := u.getMasterNodes()
    
    // 按顺序配置Master节点（第一个Master节点需要执行kubeadm init）
    for i, node := range masterNodes {
        log.Infof("Configuring master node %s (%d/%d)...", node.Hostname, i+1, len(masterNodes))
        
        // 更新状态
        u.saveProgress("master-nodes", i*100/len(masterNodes))
        
        // 配置节点
        if err := u.configureMasterNode(ctx, node, i == 0); err != nil {
            return fmt.Errorf("failed to configure master node %s: %v", node.Hostname, err)
        }
        
        // 如果是第一个Master节点，获取Join命令
        if i == 0 {
            if err := u.retrieveJoinCommand(ctx, node); err != nil {
                return err
            }
        }
        
        // 等待节点就绪
        if err := u.waitForNodeReady(ctx, node); err != nil {
            return err
        }
    }
    
    return nil
}

func (u *UPIInstaller) configureMasterNode(ctx context.Context, node *NodeInfo, isFirst bool) error {
    configurator := nodemanager.NewNodeConfigurator(u.installDir, u.stateManager)
    
    // 配置节点
    if err := configurator.ConfigureNode(ctx, node, "master"); err != nil {
        return err
    }
    
    // 如果不是第一个Master节点，需要Join
    if !isFirst {
        joinCmd, err := u.getJoinCommand()
        if err != nil {
            return err
        }
        
        // 执行Join
        executor, err := executor.NewSSHExecutor(node.IP, 22, node.SSHUser, []byte(node.SSHKey))
        if err != nil {
            return err
        }
        defer executor.Close()
        
        output, err := executor.Execute(joinCmd)
        if err != nil {
            return fmt.Errorf("failed to join master node: %v\nOutput: %s", err, output)
        }
    }
    
    return nil
}

func (u *UPIInstaller) retrieveJoinCommand(ctx context.Context, firstMaster *NodeInfo) error {
    executor, err := executor.NewSSHExecutor(
        firstMaster.IP,
        22,
        firstMaster.SSHUser,
        []byte(firstMaster.SSHKey),
    )
    if err != nil {
        return err
    }
    defer executor.Close()
    
    // 获取Join命令
    output, err := executor.Execute("kubeadm token create --print-join-command")
    if err != nil {
        return err
    }
    
    // 保存Join命令
    joinCmdFile := filepath.Join(u.installDir, "kubeadm-join-command.sh")
    if err := os.WriteFile(joinCmdFile, []byte(output), 0755); err != nil {
        return err
    }
    
    return nil
}
```
#### 4.3 Worker节点配置流程
```go
func (u *UPIInstaller) configureWorkerNodes(ctx context.Context) error {
    workerNodes := u.getWorkerNodes()
    
    // 并发配置Worker节点
    g, ctx := errgroup.WithContext(ctx)
    
    for i, node := range workerNodes {
        node := node
        i := i
        
        g.Go(func() error {
            log.Infof("Configuring worker node %s (%d/%d)...", node.Hostname, i+1, len(workerNodes))
            
            // 更新状态
            u.saveProgress("worker-nodes", i*100/len(workerNodes))
            
            // 配置节点
            configurator := nodemanager.NewNodeConfigurator(u.installDir, u.stateManager)
            if err := configurator.ConfigureNode(ctx, node, "worker"); err != nil {
                return fmt.Errorf("failed to configure worker node %s: %v", node.Hostname, err)
            }
            
            // 执行Join
            joinCmd, err := u.getJoinCommand()
            if err != nil {
                return err
            }
            
            executor, err := executor.NewSSHExecutor(node.IP, 22, node.SSHUser, []byte(node.SSHKey))
            if err != nil {
                return err
            }
            defer executor.Close()
            
            output, err := executor.Execute(joinCmd)
            if err != nil {
                return fmt.Errorf("failed to join worker node: %v\nOutput: %s", err, output)
            }
            
            // 等待节点就绪
            return u.waitForNodeReady(ctx, node)
        })
    }
    
    return g.Wait()
}
```
### 五、配置文件示例
#### 5.1 install-config.yaml
```yaml
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: my-cluster
spec:
  clusterConfig:
    baseDomain: example.com
    platform:
      type: baremetal  # 或 vsphere, none
      userProvisionedDNS: true
      userProvidedLB: true
      dnsServers:
        - 192.168.1.10
        - 192.168.1.11
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    controlPlaneEndpoint: 192.168.1.100:6443
    kubernetesVersion: v1.28.8
    imageRepo:
      domain: registry.example.com
      namespace: kubernetes
    
  nodes:
    - hostname: master-0
      ip: 192.168.1.100
      roles:
        - master
        - etcd
      sshUser: root
      sshKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
        
    - hostname: master-1
      ip: 192.168.1.101
      roles:
        - master
        - etcd
      sshUser: root
      sshKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
        
    - hostname: master-2
      ip: 192.168.1.102
      roles:
        - master
        - etcd
      sshUser: root
      sshKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
        
    - hostname: worker-0
      ip: 192.168.1.110
      roles:
        - worker
      sshUser: root
      sshKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
        
    - hostname: worker-1
      ip: 192.168.1.111
      roles:
        - worker
      sshUser: root
      sshKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
```
### 六、命令行接口
#### 6.1 安装命令
```bash
# 生成安装配置
bkeadm install config --config install-config.yaml

# 生成证书
bkeadm install certs

# 生成Kubeconfig
bkeadm install kubeconfig

# 生成Kubeadm配置
bkeadm install kubeadm-config

# 生成节点设置脚本
bkeadm install scripts

# 配置节点
bkeadm install configure-nodes

# 完整安装
bkeadm install --config install-config.yaml

# 恢复安装
bkeadm install --resume
```
#### 6.2 输出目录结构
```
my-cluster/
├── .bke_install_state.json      # 安装状态
├── install-config.yaml           # 安装配置
├── certs/                        # 证书
│   ├── ca.crt
│   ├── ca.key
│   ├── etcd/
│   │   ├── ca.crt
│   │   └── ca.key
│   └── ...
├── kubeconfigs/                  # Kubeconfig文件
│   ├── admin.conf
│   ├── kubelet.conf
│   └── ...
├── kubeadm/                      # Kubeadm配置
│   ├── kubeadm-init.yaml
│   └── kubeadm-join.yaml
├── scripts/                      # 节点设置脚本
│   ├── setup-common.sh
│   ├── setup-master.sh
│   └── setup-worker.sh
├── manifests/                    # Kubernetes清单
│   ├── cni/
│   ├── csi/
│   └── addons/
└── kubeadm-join-command.sh      # Join命令
```
### 七、关键优势
1. **Ubuntu原生支持**：使用Shell脚本和Cloud-Init，无需依赖CoreOS
2. **Asset-Based架构**：保留OpenShift Installer的核心设计，易于维护和扩展
3. **SSH自动化**：通过SSH自动配置所有节点，无需手动操作
4. **状态持久化**：支持断点续传，提高可靠性
5. **并发执行**：Worker节点并发配置，提高效率
6. **可观测性**：清晰的进度显示和错误报告

这个方案针对Ubuntu场景进行了优化，使用Shell脚本和SSH自动化替代Ignition，同时保留了Asset-Based架构的核心优势。

# Cloud-Init
Cloud-Init 是在机器首次启动时使用的。它的设计目标就是在虚拟机或云主机第一次开机时，自动完成初始化配置。  
## 📌 Cloud-Init 的工作机制
1. **首次启动触发**  
   - 当虚拟机或云主机第一次启动时，Cloud-Init 会读取数据源（Data Source），例如：  
     - 云平台提供的 metadata service（AWS EC2、OpenStack、Azure 等）  
     - ISO/CD-ROM 注入的配置文件  
     - 用户自定义的配置脚本  
2. **执行初始化任务**  
   - 设置主机名、用户账号、SSH key。  
   - 配置网络（IP、DNS、路由）。  
   - 安装软件包、运行脚本。  
   - 挂载磁盘、调整文件系统。  
3. **生命周期**  
   - Cloud-Init 默认只在 **首次启动**时运行这些初始化任务。  
   - 后续启动时，它不会重复执行完整初始化，而是进入“已完成”状态。  
   - 如果需要再次运行，可以手动清理 Cloud-Init 状态文件，然后重启。  
## 🔄 与 Ignition 的对比
- **Cloud-Init**：通用云环境初始化工具，支持多种 Linux 发行版（RHEL、Ubuntu、CentOS 等）。  
- **Ignition**：专为 CoreOS/RHCOS 设计，只在首次启动时应用配置，之后由 MachineConfig Operator 管理。  

✅ **结论**：Cloud-Init 的核心特性就是 **在机器首次启动时执行初始化配置**，后续启动不会重复运行，除非手动重置状态。  
## 📊 Cloud-Init vs Ignition 对比表
| 特性 | **Cloud-Init** | **Ignition** |
|------|----------------|--------------|
| **适用系统** | 通用 Linux 发行版（RHEL、Ubuntu、CentOS 等） | 专为 CoreOS/RHCOS 设计 |
| **运行时机** | 首次启动时执行初始化任务（可手动重置后再次运行） | 仅在首次启动时应用配置，之后不可重复 |
| **配置内容** | 用户账号、SSH key、网络、软件包安装、脚本执行 | 文件系统分区、挂载、系统服务、kubelet 配置 |
| **数据源** | 云平台 metadata service、ISO/CD-ROM、用户提供的配置文件 | `openshift-install` 生成的 Ignition 文件 |
| **灵活性** | 高度灵活，支持多种发行版和云环境 | 专注于 OpenShift/RHCOS，功能更固定 |
| **后续管理** | OS 配置需用户或自动化工具维护 | 后续由 MachineConfig Operator 统一管理 |
| **典型场景** | 云主机初始化（AWS、Azure、OpenStack 等） | OpenShift 集群节点自动化安装与配置 |
### ✅ 总结
- **Cloud-Init**：通用型工具，适合各种云环境和 Linux 发行版，灵活但需要用户维护。  
- **Ignition**：专为 RHCOS/OpenShift 设计，自动化程度高，后续由集群统一管理。  

    
# 基于不使用Cloud-Init的要求，优化openFuyao UPI自动化安装方案。
## openFuyao UPI自动化安装方案（Ubuntu + 纯SSH方案）
### 一、总体架构设计
#### 1.1 Asset-Based架构（保留核心设计）
```go
package asset

type Asset interface {
    Dependencies() []Asset
    Generate(ctx context.Context, parents Parents) error
    Name() string
}

type WritableAsset interface {
    Asset
    Files() []*File
    Load(FileFetcher) (found bool, err error)
}
```
#### 1.2 Ubuntu场景Asset清单（优化版）
```go
var UPIUbuntuAssets = []asset.WritableAsset{
    // 1. 配置类Asset
    &InstallConfig{},
    &ClusterID{},
    &NetworkingConfig{},
    &DNSConfig{},
    &NodeInventory{},
    
    // 2. 证书类Asset
    &RootCA{},
    &EtcdCA{},
    &KubeCA{},
    &FrontProxyCA{},
    &ServiceAccountKey{},
    &APIServerCert{},
    &KubeletCert{},
    &EtcdCert{},
    &FrontProxyClientCert{},
    
    // 3. Kubeconfig类Asset
    &AdminKubeconfig{},
    &KubeletKubeconfig{},
    &ControllerManagerKubeconfig{},
    &SchedulerKubeconfig{},
    
    // 4. 配置文件Asset（纯Shell脚本方案）
    &NodeSetupScripts{},         // 节点设置脚本
    &KubeadmInitConfig{},        // Kubeadm初始化配置
    &KubeadmJoinConfig{},        // Kubeadm Join配置
    &ContainerdConfig{},         // Containerd配置
    &SystemConfig{},             // 系统配置
    &NetworkConfig{},            // 网络配置
    
    // 5. 清单类Asset
    &CNIManifests{},
    &CSIManifests{},
    &AddonManifests{},
    
    // 6. 状态类Asset
    &ClusterMetadata{},
    &InstallState{},
}
```
### 二、核心Asset设计详解
#### 2.1 节点设置脚本Asset（核心）
```go
package nodesetup

type NodeSetupScripts struct {
    CommonScript      []byte
    MasterInitScript  []byte
    MasterJoinScript  []byte
    WorkerJoinScript  []byte
    Files             []*asset.File
}

func (n *NodeSetupScripts) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &tls.RootCA{},
        &tls.KubeCA{},
        &tls.EtcdCA{},
        &tls.ServiceAccountKey{},
        &tls.APIServerCert{},
        &tls.KubeletCert{},
        &tls.EtcdCert{},
        &kubeconfig.AdminKubeconfig{},
        &kubeconfig.KubeletKubeconfig{},
        &kubeadm.KubeadmInitConfig{},
        &kubeadm.KubeadmJoinConfig{},
    }
}

func (n *NodeSetupScripts) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    rootCA := &tls.RootCA{}
    kubeCA := &tls.KubeCA{}
    etcdCA := &tls.EtcdCA{}
    saKey := &tls.ServiceAccountKey{}
    apiServerCert := &tls.APIServerCert{}
    kubeletCert := &tls.KubeletCert{}
    etcdCert := &tls.EtcdCert{}
    parents.Get(installConfig, rootCA, kubeCA, etcdCA, saKey, apiServerCert, kubeletCert, etcdCert)
    
    // 1. 生成通用配置脚本
    n.CommonScript = n.generateCommonScript(installConfig)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/common.sh",
        Data:     n.CommonScript,
    })
    
    // 2. 生成Master初始化脚本（第一个Master节点）
    n.MasterInitScript = n.generateMasterInitScript(installConfig, rootCA, kubeCA, etcdCA, saKey, apiServerCert, etcdCert)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/master-init.sh",
        Data:     n.MasterInitScript,
    })
    
    // 3. 生成Master Join脚本（后续Master节点）
    n.MasterJoinScript = n.generateMasterJoinScript(installConfig, rootCA, kubeCA, etcdCA, saKey)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/master-join.sh",
        Data:     n.MasterJoinScript,
    })
    
    // 4. 生成Worker Join脚本
    n.WorkerJoinScript = n.generateWorkerJoinScript(installConfig, rootCA, kubeCA)
    n.Files = append(n.Files, &asset.File{
        Filename: "scripts/worker-join.sh",
        Data:     n.WorkerJoinScript,
    })
    
    return nil
}

func (n *NodeSetupScripts) generateCommonScript(config *installconfig.InstallConfig) []byte {
    script := `#!/bin/bash
set -euo pipefail

COMMON_SCRIPT

BKE_COMMON_DIR="/etc/bke"
BKE_LOG_DIR="/var/log/bke"
BKE_BIN_DIR="/usr/local/bin"

log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a ${BKE_LOG_DIR}/setup.log
}

log_info() {
    log "INFO" "$@"
}

log_error() {
    log "ERROR" "$@"
}

check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "This script must be run as root"
        exit 1
    fi
}

configure_system() {
    log_info "Configuring system parameters..."
    
    # 配置内核参数
    cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.forwarding        = 1
net.ipv6.conf.all.forwarding        = 1
EOF
    
    sysctl --system
    log_info "System parameters configured"
}

configure_modules() {
    log_info "Configuring kernel modules..."
    
    # 加载内核模块
    cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
    
    modprobe overlay
    modprobe br_netfilter
    modprobe ip_vs
    modprobe ip_vs_rr
    modprobe ip_vs_wrr
    modprobe ip_vs_sh
    modprobe nf_conntrack
    
    log_info "Kernel modules configured"
}

disable_swap() {
    log_info "Disabling swap..."
    
    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    
    log_info "Swap disabled"
}

configure_hostname() {
    local hostname=$1
    local ip=$2
    
    log_info "Configuring hostname: $hostname"
    
    # 设置主机名
    hostnamectl set-hostname $hostname
    
    # 配置/etc/hosts
    grep -q "$hostname" /etc/hosts || echo "$ip $hostname" >> /etc/hosts
    
    log_info "Hostname configured"
}

install_dependencies() {
    log_info "Installing dependencies..."
    
    apt-get update
    apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg \
        lsb-release \
        jq \
        socat \
        conntrack \
        ipset \
        ipvsadm
    
    log_info "Dependencies installed"
}

install_containerd() {
    log_info "Installing containerd..."
    
    if ! command -v containerd &> /dev/null; then
        # 下载并安装containerd
        apt-get install -y containerd
        
        # 创建配置目录
        mkdir -p /etc/containerd
        
        log_info "Containerd installed"
    else
        log_info "Containerd already installed"
    fi
}

configure_containerd() {
    log_info "Configuring containerd..."
    
    # 生成默认配置
    containerd config default > /etc/containerd/config.toml
    
    # 配置SystemdCgroup
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    
    # 配置镜像仓库
    sed -i '/\[plugins."io.containerd.grpc.v1.cri".registry.mirrors\]/a\        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]\n          endpoint = ["https://registry.example.com"]' /etc/containerd/config.toml
    
    # 重启containerd
    systemctl restart containerd
    systemctl enable containerd
    
    log_info "Containerd configured"
}

install_kubernetes() {
    local kubernetes_version=$1
    local image_repo=$2
    
    log_info "Installing Kubernetes $kubernetes_version..."
    
    if ! command -v kubeadm &> /dev/null; then
        # 添加Kubernetes仓库
        curl -fsSL https://pkgs.k8s.io/core:/stable:/${kubernetes_version}/deb/Release.key | \
            gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${kubernetes_version}/deb/ /" | \
            tee /etc/apt/sources.list.d/kubernetes.list
        
        apt-get update
        apt-get install -y kubelet kubeadm kubectl
        apt-mark hold kubelet kubeadm kubectl
        
        log_info "Kubernetes installed"
    else
        log_info "Kubernetes already installed"
    fi
}

create_directories() {
    log_info "Creating directories..."
    
    mkdir -p ${BKE_COMMON_DIR}
    mkdir -p ${BKE_LOG_DIR}
    mkdir -p ${BKE_BIN_DIR}
    mkdir -p /etc/kubernetes/pki
    mkdir -p /etc/kubernetes/pki/etcd
    mkdir -p /var/lib/etcd
    
    log_info "Directories created"
}

write_certificates() {
    local cert_dir=$1
    
    log_info "Writing certificates..."
    
    if [[ -d "$cert_dir" ]]; then
        cp -r ${cert_dir}/* /etc/kubernetes/pki/
        chmod 600 /etc/kubernetes/pki/*.key
        chmod 600 /etc/kubernetes/pki/etcd/*.key
        chmod 644 /etc/kubernetes/pki/*.crt
        chmod 644 /etc/kubernetes/pki/etcd/*.crt
        
        log_info "Certificates written"
    else
        log_error "Certificate directory not found: $cert_dir"
        exit 1
    fi
}

write_kubeconfigs() {
    local kubeconfig_dir=$1
    
    log_info "Writing kubeconfigs..."
    
    if [[ -d "$kubeconfig_dir" ]]; then
        cp -r ${kubeconfig_dir}/* /etc/kubernetes/
        chmod 600 /etc/kubernetes/*.conf
        
        log_info "Kubeconfigs written"
    else
        log_error "Kubeconfig directory not found: $kubeconfig_dir"
        exit 1
    fi
}

wait_for_api_server() {
    local api_server=$1
    local max_retries=30
    local retry_interval=10
    
    log_info "Waiting for API server $api_server to be ready..."
    
    for i in $(seq 1 $max_retries); do
        if curl -k -s https://${api_server}/healthz | grep -q "ok"; then
            log_info "API server is ready"
            return 0
        fi
        
        log_info "API server not ready, retrying ($i/$max_retries)..."
        sleep $retry_interval
    done
    
    log_error "API server is not ready after $max_retries retries"
    return 1
}

verify_node() {
    log_info "Verifying node status..."
    
    # 检查kubelet
    if ! systemctl is-active --quiet kubelet; then
        log_error "Kubelet is not running"
        return 1
    fi
    
    # 检查containerd
    if ! systemctl is-active --quiet containerd; then
        log_error "Containerd is not running"
        return 1
    fi
    
    log_info "Node verification passed"
    return 0
}

# 主函数
main() {
    check_root
    
    mkdir -p ${BKE_LOG_DIR}
    
    log_info "=== BKE Common Setup Started ==="
    
    configure_system
    configure_modules
    disable_swap
    install_dependencies
    install_containerd
    configure_containerd
    
    log_info "=== BKE Common Setup Completed ==="
}

# 如果直接执行，则运行main
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
`
    return []byte(script)
}

func (n *NodeSetupScripts) generateMasterInitScript(
    config *installconfig.InstallConfig,
    rootCA *tls.RootCA,
    kubeCA *tls.KubeCA,
    etcdCA *tls.EtcdCA,
    saKey *tls.ServiceAccountKey,
    apiServerCert *tls.APIServerCert,
    etcdCert *tls.EtcdCert,
) []byte {
    script := `#!/bin/bash
set -euo pipefail

MASTER_INIT_SCRIPT

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source ${SCRIPT_DIR}/common.sh

main() {
    check_root
    
    local hostname=$1
    local ip=$2
    local api_server=$3
    local kubernetes_version="` + config.Config.Spec.KubernetesVersion + `"
    local image_repo="` + config.Config.Spec.ImageRepo.Domain + `"
    
    log_info "=== BKE Master Init Started ==="
    log_info "Hostname: $hostname"
    log_info "IP: $ip"
    log_info "API Server: $api_server"
    
    # 1. 运行通用配置
    main
    
    # 2. 配置主机名
    configure_hostname $hostname $ip
    
    # 3. 安装Kubernetes
    install_kubernetes $kubernetes_version $image_repo
    
    # 4. 创建目录
    create_directories
    
    # 5. 写入证书
    write_certificates "${SCRIPT_DIR}/../certs"
    
    # 6. 写入Kubeconfig
    write_kubeconfigs "${SCRIPT_DIR}/../kubeconfigs"
    
    # 7. 创建Kubeadm配置
    create_kubeadm_config $api_server $kubernetes_version
    
    # 8. 初始化集群
    kubeadm_init
    
    # 9. 配置kubectl
    configure_kubectl
    
    # 10. 生成Join命令
    generate_join_command
    
    # 11. 验证节点
    verify_node
    
    log_info "=== BKE Master Init Completed ==="
}

create_kubeadm_config() {
    local api_server=$1
    local kubernetes_version=$2
    
    log_info "Creating kubeadm configuration..."
    
    cat > /etc/kubernetes/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: ${kubernetes_version}
controlPlaneEndpoint: "${api_server}"
networking:
  dnsDomain: ` + config.Config.Spec.Networking.DNSDomain + `
  podSubnet: ` + config.Config.Spec.Networking.PodSubnet + `
  serviceSubnet: ` + config.Config.Spec.Networking.ServiceSubnet + `
etcd:
  local:
    dataDir: /var/lib/etcd
    serverCertSANs:
      - "${api_server%%:*}"
      - "${api_server}"
    peerCertSANs:
      - "${api_server%%:*}"
      - "${api_server}"
apiServer:
  certSANs:
    - "${api_server%%:*}"
    - "${api_server}"
    - "127.0.0.1"
    - "localhost"
controllerManager:
  extraArgs:
    cluster-signing-cert-file: /etc/kubernetes/pki/ca.crt
    cluster-signing-key-file: /etc/kubernetes/pki/ca.key
scheduler:
  extraArgs:
    bind-address: 0.0.0.0
imageRepository: ` + config.Config.Spec.ImageRepo.Domain + `/kubernetes
certificatesDir: /etc/kubernetes/pki
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $(hostname -I | awk '{print $1}')
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  kubeletExtraArgs:
    cgroup-driver: systemd
    container-runtime: remote
    container-runtime-endpoint: unix:///var/run/containerd/containerd.sock
EOF
    
    log_info "Kubeadm configuration created"
}

kubeadm_init() {
    log_info "Initializing Kubernetes cluster..."
    
    # 启动kubelet
    systemctl enable kubelet
    systemctl start kubelet
    
    # 执行kubeadm init
    kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --upload-certs --ignore-preflight-errors=all
    
    log_info "Kubernetes cluster initialized"
}

configure_kubectl() {
    log_info "Configuring kubectl..."
    
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    
    log_info "Kubectl configured"
}

generate_join_command() {
    log_info "Generating join command..."
    
    # 生成token
    kubeadm token create --print-join-command > ${BKE_COMMON_DIR}/kubeadm-join-command.sh
    chmod +x ${BKE_COMMON_DIR}/kubeadm-join-command.sh
    
    # 生成control-plane join命令
    local join_cmd=$(cat ${BKE_COMMON_DIR}/kubeadm-join-command.sh)
    local cert_key=$(kubeadm init phase upload-certs --upload-certs 2>&1 | grep -A1 "Using certificate key" | tail -n1)
    
    echo "${join_cmd} --control-plane --certificate-key ${cert_key}" > ${BKE_COMMON_DIR}/kubeadm-join-control-plane.sh
    chmod +x ${BKE_COMMON_DIR}/kubeadm-join-control-plane.sh
    
    log_info "Join commands generated"
}

# 执行主函数
main "$@"
`
    return []byte(script)
}

func (n *NodeSetupScripts) generateMasterJoinScript(
    config *installconfig.InstallConfig,
    rootCA *tls.RootCA,
    kubeCA *tls.KubeCA,
    etcdCA *tls.EtcdCA,
    saKey *tls.ServiceAccountKey,
) []byte {
    script := `#!/bin/bash
set -euo pipefail

MASTER_JOIN_SCRIPT

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source ${SCRIPT_DIR}/common.sh

main() {
    check_root
    
    local hostname=$1
    local ip=$2
    local api_server=$3
    local join_command_file=$4
    local kubernetes_version="` + config.Config.Spec.KubernetesVersion + `"
    local image_repo="` + config.Config.Spec.ImageRepo.Domain + `"
    
    log_info "=== BKE Master Join Started ==="
    log_info "Hostname: $hostname"
    log_info "IP: $ip"
    log_info "API Server: $api_server"
    
    # 1. 运行通用配置
    main
    
    # 2. 配置主机名
    configure_hostname $hostname $ip
    
    # 3. 安装Kubernetes
    install_kubernetes $kubernetes_version $image_repo
    
    # 4. 创建目录
    create_directories
    
    # 5. 写入证书
    write_certificates "${SCRIPT_DIR}/../certs"
    
    # 6. 写入Kubeconfig
    write_kubeconfigs "${SCRIPT_DIR}/../kubeconfigs"
    
    # 7. 等待API Server就绪
    wait_for_api_server $api_server
    
    # 8. 执行Join
    master_join $join_command_file
    
    # 9. 配置kubectl
    configure_kubectl
    
    # 10. 验证节点
    verify_node
    
    log_info "=== BKE Master Join Completed ==="
}

master_join() {
    local join_command_file=$1
    
    log_info "Joining control plane..."
    
    if [[ ! -f "$join_command_file" ]]; then
        log_error "Join command file not found: $join_command_file"
        exit 1
    fi
    
    # 启动kubelet
    systemctl enable kubelet
    systemctl start kubelet
    
    # 执行join
    bash $join_command_file
    
    log_info "Control plane joined"
}

configure_kubectl() {
    log_info "Configuring kubectl..."
    
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    
    log_info "Kubectl configured"
}

# 执行主函数
main "$@"
`
    return []byte(script)
}

func (n *NodeSetupScripts) generateWorkerJoinScript(
    config *installconfig.InstallConfig,
    rootCA *tls.RootCA,
    kubeCA *tls.KubeCA,
) []byte {
    script := `#!/bin/bash
set -euo pipefail

WORKER_JOIN_SCRIPT

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source ${SCRIPT_DIR}/common.sh

main() {
    check_root
    
    local hostname=$1
    local ip=$2
    local api_server=$3
    local join_command_file=$4
    local kubernetes_version="` + config.Config.Spec.KubernetesVersion + `"
    local image_repo="` + config.Config.Spec.ImageRepo.Domain + `"
    
    log_info "=== BKE Worker Join Started ==="
    log_info "Hostname: $hostname"
    log_info "IP: $ip"
    log_info "API Server: $api_server"
    
    # 1. 运行通用配置
    main
    
    # 2. 配置主机名
    configure_hostname $hostname $ip
    
    # 3. 安装Kubernetes
    install_kubernetes $kubernetes_version $image_repo
    
    # 4. 创建目录
    create_directories
    
    # 5. 写入CA证书
    write_ca_certificates "${SCRIPT_DIR}/../certs"
    
    # 6. 等待API Server就绪
    wait_for_api_server $api_server
    
    # 7. 执行Join
    worker_join $join_command_file
    
    # 8. 验证节点
    verify_node
    
    log_info "=== BKE Worker Join Completed ==="
}

write_ca_certificates() {
    local cert_dir=$1
    
    log_info "Writing CA certificates..."
    
    if [[ -d "$cert_dir" ]]; then
        mkdir -p /etc/kubernetes/pki
        
        # 只复制CA证书
        cp ${cert_dir}/ca.crt /etc/kubernetes/pki/
        cp ${cert_dir}/ca.key /etc/kubernetes/pki/ 2>/dev/null || true
        
        chmod 644 /etc/kubernetes/pki/*.crt
        chmod 600 /etc/kubernetes/pki/*.key 2>/dev/null || true
        
        log_info "CA certificates written"
    else
        log_error "Certificate directory not found: $cert_dir"
        exit 1
    fi
}

worker_join() {
    local join_command_file=$1
    
    log_info "Joining cluster as worker..."
    
    if [[ ! -f "$join_command_file" ]]; then
        log_error "Join command file not found: $join_command_file"
        exit 1
    fi
    
    # 启动kubelet
    systemctl enable kubelet
    systemctl start kubelet
    
    # 执行join
    bash $join_command_file
    
    log_info "Worker joined cluster"
}

# 执行主函数
main "$@"
`
    return []byte(script)
}
```
#### 2.2 证书Asset（优化版）
```go
package tls

type RootCA struct {
    CertData []byte
    KeyData  []byte
    Files    []*asset.File
}

func (r *RootCA) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.ClusterID{},
    }
}

func (r *RootCA) Generate(ctx context.Context, parents asset.Parents) error {
    clusterID := &installconfig.ClusterID{}
    parents.Get(clusterID)
    
    // 生成根CA证书
    cert, key, err := r.generateCACertificate(clusterID.Name)
    if err != nil {
        return err
    }
    
    r.CertData = cert
    r.KeyData = key
    
    r.Files = []*asset.File{
        {
            Filename: "certs/ca.crt",
            Data:     cert,
        },
        {
            Filename: "certs/ca.key",
            Data:     key,
        },
    }
    
    return nil
}

func (r *RootCA) generateCACertificate(clusterName string) ([]byte, []byte, error) {
    // 使用crypto库生成证书
    priv, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        return nil, nil, err
    }
    
    template := x509.Certificate{
        SerialNumber: big.NewInt(1),
        Subject: pkix.Name{
            CommonName:   fmt.Sprintf("kubernetes-ca-%s", clusterName),
            Organization: []string{"kubernetes"},
        },
        NotBefore:             time.Now(),
        NotAfter:              time.Now().Add(10 * 365 * 24 * time.Hour),
        KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
        BasicConstraintsValid: true,
        IsCA:                  true,
    }
    
    certData, err := x509.CreateCertificate(rand.Reader, &template, &template, &priv.PublicKey, priv)
    if err != nil {
        return nil, nil, err
    }
    
    certPEM := pem.EncodeToMemory(&pem.Block{
        Type:  "CERTIFICATE",
       	Bytes: certData,
    })
    
    keyPEM := pem.EncodeToMemory(&pem.Block{
        Type:  "RSA PRIVATE KEY",
        Bytes: x509.MarshalPKCS1PrivateKey(priv),
    })
    
    return certPEM, keyPEM, nil
}
```
### 三、节点配置执行器设计（优化版）
#### 3.1 SSH执行器（增强版）
```go
package executor

type SSHExecutor struct {
    client     *ssh.Client
    sftpClient *sftp.Client
    host       string
    port       int
    user       string
    privateKey []byte
    timeout    time.Duration
}

func NewSSHExecutor(host string, port int, user string, privateKey []byte) (*SSHExecutor, error) {
    signer, err := ssh.ParsePrivateKey(privateKey)
    if err != nil {
        return nil, fmt.Errorf("failed to parse private key: %v", err)
    }
    
    config := &ssh.ClientConfig{
        User: user,
        Auth: []ssh.AuthMethod{
            ssh.PublicKeys(signer),
        },
        HostKeyCallback: ssh.InsecureIgnoreHostKey(),
        Timeout:         30 * time.Second,
    }
    
    client, err := ssh.Dial("tcp", fmt.Sprintf("%s:%d", host, port), config)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to %s:%d: %v", host, port, err)
    }
    
    sftpClient, err := sftp.NewClient(client)
    if err != nil {
        client.Close()
        return nil, fmt.Errorf("failed to create sftp client: %v", err)
    }
    
    return &SSHExecutor{
        client:     client,
        sftpClient: sftpClient,
        host:       host,
        port:       port,
        user:       user,
        privateKey: privateKey,
        timeout:    30 * time.Minute,
    }, nil
}

func (e *SSHExecutor) Execute(command string) (string, error) {
    session, err := e.client.NewSession()
    if err != nil {
        return "", fmt.Errorf("failed to create session: %v", err)
    }
    defer session.Close()
    
    var stdout, stderr bytes.Buffer
    session.Stdout = &stdout
    session.Stderr = &stderr
    
    err = session.Run(command)
    output := stdout.String()
    
    if err != nil {
        return output, fmt.Errorf("command failed: %v\nStderr: %s", err, stderr.String())
    }
    
    return output, nil
}

func (e *SSHExecutor) ExecuteWithOutput(command string, stdout io.Writer, stderr io.Writer) error {
    session, err := e.client.NewSession()
    if err != nil {
        return err
    }
    defer session.Close()
    
    session.Stdout = stdout
    session.Stderr = stderr
    
    return session.Run(command)
}

func (e *SSHExecutor) UploadFile(localPath, remotePath string) error {
    localFile, err := os.Open(localPath)
    if err != nil {
        return err
    }
    defer localFile.Close()
    
    remoteFile, err := e.sftpClient.Create(remotePath)
    if err != nil {
        return err
    }
    defer remoteFile.Close()
    
    _, err = io.Copy(remoteFile, localFile)
    return err
}

func (e *SSHExecutor) UploadData(data []byte, remotePath string) error {
    remoteFile, err := e.sftpClient.Create(remotePath)
    if err != nil {
        return err
    }
    defer remoteFile.Close()
    
    _, err = remoteFile.Write(data)
    return err
}

func (e *SSHExecutor) UploadDirectory(localDir, remoteDir string) error {
    // 创建远程目录
    if err := e.sftpClient.MkdirAll(remoteDir); err != nil {
        return err
    }
    
    return filepath.Walk(localDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        relPath, _ := filepath.Rel(localDir, path)
        remotePath := filepath.Join(remoteDir, relPath)
        
        if info.IsDir() {
            return e.sftpClient.MkdirAll(remotePath)
        }
        
        return e.UploadFile(path, remotePath)
    })
}

func (e *SSHExecutor) DownloadFile(remotePath, localPath string) error {
    remoteFile, err := e.sftpClient.Open(remotePath)
    if err != nil {
        return err
    }
    defer remoteFile.Close()
    
    localFile, err := os.Create(localPath)
    if err != nil {
        return err
    }
    defer localFile.Close()
    
    _, err = io.Copy(localFile, remoteFile)
    return err
}

func (e *SSHExecutor) Close() error {
    if e.sftpClient != nil {
        e.sftpClient.Close()
    }
    if e.client != nil {
        return e.client.Close()
    }
    return nil
}
```
#### 3.2 节点配置管理器（优化版）
```go
package nodemanager

type NodeConfigurator struct {
    installDir string
    state      *state.Manager
    executor   *executor.SSHExecutor
}

func NewNodeConfigurator(installDir string, state *state.Manager) *NodeConfigurator {
    return &NodeConfigurator{
        installDir: installDir,
        state:      state,
    }
}

func (n *NodeConfigurator) ConfigureNode(ctx context.Context, node *NodeInfo, role string, isFirst bool) error {
    log.Infof("Configuring %s node %s (%s)", role, node.Hostname, node.IP)
    
    // 1. 建立SSH连接
    var err error
    n.executor, err = executor.NewSSHExecutor(
        node.IP,
        22,
        node.SSHUser,
        []byte(node.SSHKey),
    )
    if err != nil {
        return fmt.Errorf("failed to connect to node %s: %v", node.Hostname, err)
    }
    defer n.executor.Close()
    
    // 2. 创建远程目录
    if err := n.createRemoteDirectories(); err != nil {
        return err
    }
    
    // 3. 上传所有文件
    if err := n.uploadAllFiles(role); err != nil {
        return err
    }
    
    // 4. 执行配置脚本
    if err := n.executeSetupScript(node, role, isFirst); err != nil {
        return err
    }
    
    // 5. 验证节点状态
    if err := n.verifyNode(role); err != nil {
        return err
    }
    
    // 6. 如果是第一个Master节点，获取Join命令
    if role == "master" && isFirst {
        if err := n.retrieveJoinCommands(); err != nil {
            return err
        }
    }
    
    log.Infof("Node %s configured successfully", node.Hostname)
    return nil
}

func (n *NodeConfigurator) createRemoteDirectories() error {
    dirs := []string{
        "/etc/bke",
        "/var/log/bke",
        "/etc/kubernetes/pki",
        "/etc/kubernetes/pki/etcd",
        "/var/lib/etcd",
    }
    
    for _, dir := range dirs {
        if _, err := n.executor.Execute(fmt.Sprintf("mkdir -p %s", dir)); err != nil {
            return err
        }
    }
    
    return nil
}

func (n *NodeConfigurator) uploadAllFiles(role string) error {
    // 1. 上传通用脚本
    commonScript := filepath.Join(n.installDir, "scripts/common.sh")
    if err := n.executor.UploadFile(commonScript, "/tmp/bke/common.sh"); err != nil {
        return fmt.Errorf("failed to upload common.sh: %v", err)
    }
    
    // 2. 上传角色特定脚本
    var scriptName string
    switch role {
    case "master":
        scriptName = "master-init.sh"
    case "worker":
        scriptName = "worker-join.sh"
    }
    
    scriptPath := filepath.Join(n.installDir, "scripts", scriptName)
    if err := n.executor.UploadFile(scriptPath, fmt.Sprintf("/tmp/bke/%s", scriptName)); err != nil {
        return fmt.Errorf("failed to upload %s: %v", scriptName, err)
    }
    
    // 3. 上传证书目录
    certDir := filepath.Join(n.installDir, "certs")
    if err := n.executor.UploadDirectory(certDir, "/tmp/bke/certs"); err != nil {
        return fmt.Errorf("failed to upload certs: %v", err)
    }
    
    // 4. 上传kubeconfig目录
    kubeconfigDir := filepath.Join(n.installDir, "kubeconfigs")
    if err := n.executor.UploadDirectory(kubeconfigDir, "/tmp/bke/kubeconfigs"); err != nil {
        return fmt.Errorf("failed to upload kubeconfigs: %v", err)
    }
    
    // 5. 上传kubeadm配置
    kubeadmConfig := filepath.Join(n.installDir, "kubeadm/kubeadm-init.yaml")
    if err := n.executor.UploadFile(kubeadmConfig, "/tmp/bke/kubeadm-init.yaml"); err != nil {
        return fmt.Errorf("failed to upload kubeadm config: %v", err)
    }
    
    return nil
}

func (n *NodeConfigurator) executeSetupScript(node *NodeInfo, role string, isFirst bool) error {
    var scriptPath string
    var args []string
    
    switch role {
    case "master":
        if isFirst {
            scriptPath = "/tmp/bke/master-init.sh"
            args = []string{
                node.Hostname,
                node.IP,
                fmt.Sprintf("%s:6443", node.IP), // API Server地址
            }
        } else {
            scriptPath = "/tmp/bke/master-join.sh"
            args = []string{
                node.Hostname,
                node.IP,
                n.getAPIServerEndpoint(),
                "/tmp/bke/kubeadm-join-control-plane.sh",
            }
        }
    case "worker":
        scriptPath = "/tmp/bke/worker-join.sh"
        args = []string{
            node.Hostname,
            node.IP,
            n.getAPIServerEndpoint(),
            "/tmp/bke/kubeadm-join-command.sh",
        }
    }
    
    // 设置执行权限
    if _, err := n.executor.Execute(fmt.Sprintf("chmod +x %s", scriptPath)); err != nil {
        return err
    }
    
    // 执行脚本
    command := fmt.Sprintf("%s %s", scriptPath, strings.Join(args, " "))
    
    // 实时输出日志
    var stdout, stderr bytes.Buffer
    err := n.executor.ExecuteWithOutput(command, &stdout, &stderr)
    
    // 输出日志
    log.Infof("Script output:\n%s", stdout.String())
    if stderr.Len() > 0 {
        log.Warnf("Script stderr:\n%s", stderr.String())
    }
    
    return err
}

func (n *NodeConfigurator) verifyNode(role string) error {
    // 检查kubelet状态
    output, err := n.executor.Execute("systemctl is-active kubelet")
    if err != nil || strings.TrimSpace(output) != "active" {
        return fmt.Errorf("kubelet is not running")
    }
    
    // 检查containerd状态
    output, err = n.executor.Execute("systemctl is-active containerd")
    if err != nil || strings.TrimSpace(output) != "active" {
        return fmt.Errorf("containerd is not running")
    }
    
    // 如果是Master节点，检查API Server
    if role == "master" {
        output, err = n.executor.Execute("curl -k -s https://localhost:6443/healthz")
        if err != nil || strings.TrimSpace(output) != "ok" {
            return fmt.Errorf("API server is not healthy")
        }
    }
    
    return nil
}

func (n *NodeConfigurator) retrieveJoinCommands() error {
    // 下载Join命令
    joinCmdPath := filepath.Join(n.installDir, "kubeadm-join-command.sh")
    if err := n.executor.DownloadFile("/etc/bke/kubeadm-join-command.sh", joinCmdPath); err != nil {
        return err
    }
    
    // 下载Control Plane Join命令
    joinControlPlanePath := filepath.Join(n.installDir, "kubeadm-join-control-plane.sh")
    if err := n.executor.DownloadFile("/etc/bke/kubeadm-join-control-plane.sh", joinControlPlanePath); err != nil {
        return err
    }
    
    return nil
}

func (n *NodeConfigurator) getAPIServerEndpoint() string {
    // 从配置中获取API Server地址
    // 这里简化处理，实际应该从InstallConfig中获取
    return "192.168.1.100:6443"
}
```
### 四、UPI场景自动化流程（优化版）
#### 4.1 主安装流程
```go
func (u *UPIInstaller) Install(ctx context.Context) error {
    // 阶段1: 生成配置资产
    log.Info("Phase 1: Generating configuration assets...")
    if err := u.generateConfigAssets(ctx); err != nil {
        return fmt.Errorf("phase 1 failed: %v", err)
    }
    u.saveProgress("config", 100)
    
    // 阶段2: 生成证书资产
    log.Info("Phase 2: Generating certificate assets...")
    if err := u.generateCertificateAssets(ctx); err != nil {
        return fmt.Errorf("phase 2 failed: %v", err)
    }
    u.saveProgress("certificates", 100)
    
    // 阶段3: 生成Kubeconfig资产
    log.Info("Phase 3: Generating kubeconfig assets...")
    if err := u.generateKubeconfigAssets(ctx); err != nil {
        return fmt.Errorf("phase 3 failed: %v", err)
    }
    u.saveProgress("kubeconfigs", 100)
    
    // 阶段4: 生成Kubeadm配置
    log.Info("Phase 4: Generating kubeadm configurations...")
    if err := u.generateKubeadmConfigs(ctx); err != nil {
        return fmt.Errorf("phase 4 failed: %v", err)
    }
    u.saveProgress("kubeadm", 100)
    
    // 阶段5: 生成节点设置脚本
    log.Info("Phase 5: Generating node setup scripts...")
    if err := u.generateNodeSetupScripts(ctx); err != nil {
        return fmt.Errorf("phase 5 failed: %v", err)
    }
    u.saveProgress("scripts", 100)
    
    // 阶段6: 配置第一个Master节点
    log.Info("Phase 6: Configuring first master node...")
    if err := u.configureFirstMasterNode(ctx); err != nil {
        return fmt.Errorf("phase 6 failed: %v", err)
    }
    u.saveProgress("first-master", 100)
    
    // 阶段7: 配置其他Master节点
    log.Info("Phase 7: Configuring other master nodes...")
    if err := u.configureOtherMasterNodes(ctx); err != nil {
        return fmt.Errorf("phase 7 failed: %v", err)
    }
    u.saveProgress("other-masters", 100)
    
    // 阶段8: 配置Worker节点
    log.Info("Phase 8: Configuring worker nodes...")
    if err := u.configureWorkerNodes(ctx); err != nil {
        return fmt.Errorf("phase 8 failed: %v", err)
    }
    u.saveProgress("workers", 100)
    
    // 阶段9: 安装网络插件
    log.Info("Phase 9: Installing network plugin...")
    if err := u.installNetworkPlugin(ctx); err != nil {
        return fmt.Errorf("phase 9 failed: %v", err)
    }
    u.saveProgress("network", 100)
    
    // 阶段10: 等待集群就绪
    log.Info("Phase 10: Waiting for cluster to be ready...")
    if err := u.waitForClusterReady(ctx); err != nil {
        return fmt.Errorf("phase 10 failed: %v", err)
    }
    u.saveProgress("cluster-ready", 100)
    
    log.Info("Installation completed successfully!")
    u.printSuccessMessage()
    
    return nil
}
```
#### 4.2 节点配置流程
```go
func (u *UPIInstaller) configureFirstMasterNode(ctx context.Context) error {
    masterNodes := u.getMasterNodes()
    if len(masterNodes) == 0 {
        return fmt.Errorf("no master nodes found")
    }
    
    firstMaster := masterNodes[0]
    log.Infof("Configuring first master node: %s (%s)", firstMaster.Hostname, firstMaster.IP)
    
    configurator := nodemanager.NewNodeConfigurator(u.installDir, u.stateManager)
    
    return configurator.ConfigureNode(ctx, firstMaster, "master", true)
}

func (u *UPIInstaller) configureOtherMasterNodes(ctx context.Context) error {
    masterNodes := u.getMasterNodes()
    if len(masterNodes) <= 1 {
        log.Info("No other master nodes to configure")
        return nil
    }
    
    otherMasters := masterNodes[1:]
    
    // 并发配置其他Master节点
    g, ctx := errgroup.WithContext(ctx)
    
    for i, node := range otherMasters {
        node := node
        i := i
        
        g.Go(func() error {
            log.Infof("Configuring master node %s (%d/%d)", node.Hostname, i+1, len(otherMasters))
            
            configurator := nodemanager.NewNodeConfigurator(u.installDir, u.stateManager)
            return configurator.ConfigureNode(ctx, node, "master", false)
        })
    }
    
    return g.Wait()
}

func (u *UPIInstaller) configureWorkerNodes(ctx context.Context) error {
    workerNodes := u.getWorkerNodes()
    if len(workerNodes) == 0 {
        log.Info("No worker nodes to configure")
        return nil
    }
    
    // 并发配置Worker节点
    g, ctx := errgroup.WithContext(ctx)
    
    for i, node := range workerNodes {
        node := node
        i := i
        
        g.Go(func() error {
            log.Infof("Configuring worker node %s (%d/%d)", node.Hostname, i+1, len(workerNodes))
            
            configurator := nodemanager.NewNodeConfigurator(u.installDir, u.stateManager)
            return configurator.ConfigureNode(ctx, node, "worker", false)
        })
    }
    
    return g.Wait()
}
```
### 五、配置文件示例（优化版）
#### 5.1 install-config.yaml
```yaml
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: my-cluster
spec:
  clusterConfig:
    baseDomain: example.com
    platform:
      type: baremetal
      userProvisionedDNS: true
      userProvidedLB: true
      dnsServers:
        - 192.168.1.10
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    controlPlaneEndpoint: 192.168.1.100:6443
    kubernetesVersion: v1.28.8
    imageRepo:
      domain: registry.example.com
      namespace: kubernetes
    
  nodes:
    - hostname: master-0
      ip: 192.168.1.100
      roles:
        - master
        - etcd
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
      
    - hostname: master-1
      ip: 192.168.1.101
      roles:
        - master
        - etcd
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
      
    - hostname: worker-0
      ip: 192.168.1.110
      roles:
        - worker
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
```
### 六、命令行接口（优化版）
```bash
# 完整安装
bkeadm install --config install-config.yaml

# 分步安装
bkeadm install config --config install-config.yaml
bkeadm install certs
bkeadm install kubeconfig
bkeadm install kubeadm-config
bkeadm install scripts
bkeadm install nodes

# 恢复安装
bkeadm install --resume

# 查看安装状态
bkeadm install status
```
### 七、输出目录结构（优化版）
```
my-cluster/
├── .bke_install_state.json      # 安装状态
├── install-config.yaml           # 安装配置
├── certs/                        # 证书
│   ├── ca.crt
│   ├── ca.key
│   ├── apiserver.crt
│   ├── apiserver.key
│   ├── etcd/
│   │   ├── ca.crt
│   │   ├── ca.key
│   │   ├── server.crt
│   │   └── server.key
│   └── ...
├── kubeconfigs/                  # Kubeconfig文件
│   ├── admin.conf
│   ├── kubelet.conf
│   ├── controller-manager.conf
│   └── scheduler.conf
├── kubeadm/                      # Kubeadm配置
│   └── kubeadm-init.yaml
├── scripts/                      # 节点设置脚本
│   ├── common.sh
│   ├── master-init.sh
│   ├── master-join.sh
│   └── worker-join.sh
├── manifests/                    # Kubernetes清单
│   ├── cni/
│   ├── csi/
│   └── addons/
├── kubeadm-join-command.sh      # Worker Join命令
└── kubeadm-join-control-plane.sh # Master Join命令
```
### 八、关键优势
1. **纯SSH方案**：无需依赖Cloud-Init，直接通过SSH执行Shell脚本
2. **模块化脚本**：通用脚本和角色特定脚本分离，易于维护
3. **完整日志**：所有操作都有详细日志记录，便于排查问题
4. **并发执行**：Worker节点并发配置，提高效率
5. **状态持久化**：支持断点续传，提高可靠性
6. **错误处理**：完善的错误检查和恢复机制
7. **可观测性**：清晰的进度显示和状态报告

这个优化方案完全基于SSH + Shell脚本，不依赖Cloud-Init，更适合Ubuntu等传统Linux发行版，同时保留了Asset-Based架构的核心优势。

# kubeconfig
**kubeconfig 文件**（`admin.conf`、`kubelet.conf`、`controller-manager.conf`、`scheduler.conf`）是 Kubernetes 集群中不同组件的 **身份认证配置文件**，它们的作用如下：
## 📌 各 kubeconfig 文件作用
- **admin.conf**  
  - 提供集群管理员使用的 kubeconfig。  
  - 包含 API Server 地址、证书、用户凭证。  
  - 用于 `kubectl` 命令行工具连接和管理集群。  
- **kubelet.conf**  
  - 提供给 **kubelet** 进程使用的 kubeconfig。  
  - kubelet 通过它与 API Server 通信，注册节点、上报状态、接收 Pod 调度信息。  
- **controller-manager.conf**  
  - 提供给 **kube-controller-manager** 使用的 kubeconfig。  
  - 控制器通过它访问 API Server，执行副本控制、节点管理、服务账号等控制逻辑。  
- **scheduler.conf**  
  - 提供给 **kube-scheduler** 使用的 kubeconfig。  
  - 调度器通过它访问 API Server，获取待调度 Pod 信息，并将调度结果写回 API Server。  
## 🔄 总结
这些 kubeconfig 文件的本质是 **不同组件的“身份凭证 + 集群访问配置”**，保证它们能安全地与 API Server 通信。  
- `admin.conf` → 给人用（kubectl 管理集群）。  
- `kubelet.conf` → 给节点用（kubelet）。  
- `controller-manager.conf` → 给控制器用。  
- `scheduler.conf` → 给调度器用。  
## kubeconfig和 API Server 之间的通信关系
```mermaid
flowchart TD
    subgraph Kubeconfigs
        A[admin.conf]
        B[kubelet.conf]
        C[controller-manager.conf]
        D[scheduler.conf]
    end

    subgraph Components
        A -->|kubectl CLI 使用| API[API Server]
        B -->|kubelet 节点注册/状态上报| API
        C -->|控制器访问集群资源| API
        D -->|调度器获取 Pod/写入调度结果| API
    end

    API -->|统一认证与授权| E[集群资源 (etcd, Pods, Services...)]
```
🔎 图解说明
- **admin.conf** → 提供管理员身份凭证，供 `kubectl` 使用。  
- **kubelet.conf** → 节点 kubelet 使用，负责注册和状态上报。  
- **controller-manager.conf** → 控制器使用，执行副本控制、节点管理等逻辑。  
- **scheduler.conf** → 调度器使用，获取待调度 Pod 并写入调度结果。  
- **API Server** → 所有组件的统一入口，负责认证、授权和资源操作。  

这样你就能一眼看出 **不同 kubeconfig 文件如何让各组件安全地与 API Server 通信**。  

# 基于Cluster API的实现方案，我为您重新设计openFuyao UPI自动化安装架构。
## openFuyao UPI自动化安装方案（基于Cluster API）
### 一、总体架构设计
#### 1.1 Cluster API核心概念
```go
// Cluster API核心资源
type Cluster struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   ClusterSpec
    Status ClusterStatus
}

type Machine struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   MachineSpec
    Status MachineStatus
}

type MachineDeployment struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   MachineDeploymentSpec
    Status MachineDeploymentStatus
}
```
#### 1.2 架构组件
```
┌─────────────────────────────────────────────────────────────┐
│                    openFuyao Installer                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Asset Generation Engine                     │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │  │
│  │  │ Config     │  │ Certificate│  │ Kubeadm    │    │  │
│  │  │ Assets     │  │ Assets     │  │ Config     │    │  │
│  │  └────────────┘  └────────────┘  └────────────┘    │  │
│  └──────────────────────────────────────────────────────┘  │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Cluster API Resource Generator                │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │  │
│  │  │ Cluster    │  │ Machine    │  │ Machine    │    │  │
│  │  │ CRD        │  │ CRD        │  │ Deployment │    │  │
│  │  └────────────┘  └────────────┘  └────────────┘    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                  Cluster API Management                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Infrastructure Provider                   │  │
│  │            (UPI BareMetal Provider)                    │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               Bootstrap Provider                       │  │
│  │            (Kubeadm Bootstrap Provider)                │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Control Plane Provider                       │  │
│  │         (Kubeadm Control Plane Provider)               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                   Target Cluster                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Master Node  │  │ Master Node  │  │ Worker Node  │    │
│  │   (Ubuntu)   │  │   (Ubuntu)   │  │   (Ubuntu)   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```
### 二、Cluster API CRD定义
#### 2.1 自定义Infrastructure Provider
```go
// api/v1alpha1/baremetalcluster_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
)

// BareMetalCluster 定义UPI场景下的集群基础设施
type BareMetalCluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   BareMetalClusterSpec   `json:"spec,omitempty"`
    Status BareMetalClusterStatus `json:"status,omitempty"`
}

type BareMetalClusterSpec struct {
    // 控制平面端点
    ControlPlaneEndpoint clusterv1.APIEndpoint `json:"controlPlaneEndpoint"`
    
    // 基础域名
    BaseDomain string `json:"baseDomain"`
    
    // 网络配置
    NetworkSpec NetworkSpec `json:"networkSpec,omitempty"`
    
    // SSH配置
    SSHSpec SSHSpec `json:"sshSpec,omitempty"`
    
    // 镜像仓库配置
    ImageRegistry ImageRegistrySpec `json:"imageRegistry,omitempty"`
    
    // 用户提供的DNS
    UserProvidedDNS bool `json:"userProvidedDNS,omitempty"`
    
    // 用户提供的负载均衡器
    UserProvidedLB bool `json:"userProvidedLB,omitempty"`
}

type BareMetalClusterStatus struct {
    // 就绪状态
    Ready bool `json:"ready"`
    
    // 条件
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
    
    // 失败原因
    FailureReason string `json:"failureReason,omitempty"`
    
    // 失败消息
    FailureMessage string `json:"failureMessage,omitempty"`
}

type NetworkSpec struct {
    // DNS域名
    DNSDomain string `json:"dnsDomain,omitempty"`
    
    // Pod子网
    PodSubnet string `json:"podSubnet,omitempty"`
    
    // Service子网
    ServiceSubnet string `json:"serviceSubnet,omitempty"`
    
    // DNS服务器
    DNSServers []string `json:"dnsServers,omitempty"`
}

type SSHSpec struct {
    // SSH用户
    User string `json:"user"`
    
    // SSH私钥
    PrivateKey string `json:"privateKey,omitempty"`
    
    // SSH私钥文件路径
    PrivateKeyFile string `json:"privateKeyFile,omitempty"`
}

type ImageRegistrySpec struct {
    // 镜像仓库域名
    Domain string `json:"domain"`
    
    // 命名空间
    Namespace string `json:"namespace,omitempty"`
}

// BareMetalMachine 定义UPI场景下的机器
type BareMetalMachine struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   BareMetalMachineSpec   `json:"spec,omitempty"`
    Status BareMetalMachineStatus `json:"status,omitempty"`
}

type BareMetalMachineSpec struct {
    // 主机名
    Hostname string `json:"hostname"`
    
    // IP地址
    IPAddress string `json:"ipAddress"`
    
    // 角色
    Roles []string `json:"roles,omitempty"`
    
    // SSH配置
    SSHSpec SSHSpec `json:"sshSpec,omitempty"`
    
    // Provider ID
    ProviderID string `json:"providerID,omitempty"`
}

type BareMetalMachineStatus struct {
    // 就绪状态
    Ready bool `json:"ready"`
    
    // 条件
    Conditions clusterv1.Conditions `json:"conditions,omitempty"`
    
    // 失败原因
    FailureReason string `json:"failureReason,omitempty"`
    
    // 失败消息
    FailureMessage string `json:"failureMessage,omitempty"`
    
    // 节点引用
    NodeRef *corev1.ObjectReference `json:"nodeRef,omitempty"`
}
```
#### 2.2 自定义Bootstrap Provider
```go
// api/v1alpha1/kubeadmconfigtemplate_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    bootstrapv1 "sigs.k8s.io/cluster-api/bootstrap/kubeadm/api/v1beta1"
)

// BKEKubeadmConfigTemplate 定义Kubeadm配置模板
type BKEKubeadmConfigTemplate struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   BKEKubeadmConfigTemplateSpec   `json:"spec,omitempty"`
    Status BKEKubeadmConfigTemplateStatus `json:"status,omitempty"`
}

type BKEKubeadmConfigTemplateSpec struct {
    // Kubeadm配置
    Template BKEKubeadmConfigTemplateResource `json:"template"`
}

type BKEKubeadmConfigTemplateResource struct {
    Spec BKEKubeadmConfigSpec `json:"spec"`
}

type BKEKubeadmConfigSpec struct {
    // Kubeadm初始化配置
    InitConfiguration *bootstrapv1.InitConfiguration `json:"initConfiguration,omitempty"`
    
    // Kubeadm集群配置
    ClusterConfiguration *bootstrapv1.ClusterConfiguration `json:"clusterConfiguration,omitempty"`
    
    // Kubeadm Join配置
    JoinConfiguration *bootstrapv1.JoinConfiguration `json:"joinConfiguration,omitempty"`
    
    // 文件
    Files []bootstrapv1.File `json:"files,omitempty"`
    
    // PreKubeadmCommands
    PreKubeadmCommands []string `json:"preKubeadmCommands,omitempty"`
    
    // PostKubeadmCommands
    PostKubeadmCommands []string `json:"postKubeadmCommands,omitempty"`
    
    // 用户
    Users []bootstrapv1.User `json:"users,omitempty"`
    
    // NTP配置
    NTP *bootstrapv1.NTP `json:"ntp,omitempty"`
    
    // 格式化磁盘
    DiskSetup *bootstrapv1.DiskSetup `json:"diskSetup,omitempty"`
    
    // 额外参数
    AdditionalOptions map[string]string `json:"additionalOptions,omitempty"`
}
```
### 三、Asset-Based架构与Cluster API集成
#### 3.1 Cluster API资源生成Asset
```go
package clusterapi

import (
    "context"
    "fmt"
    
    "github.com/openshift/installer/pkg/asset"
    "github.com/openshift/installer/pkg/asset/installconfig"
    "github.com/openshift/installer/pkg/asset/tls"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
    bootstrapv1 "sigs.k8s.io/cluster-api/bootstrap/kubeadm/api/v1beta1"
    controlplanev1 "sigs.k8s.io/cluster-api/controlplane/kubeadm/api/v1beta1"
)

// ClusterAsset 生成Cluster API Cluster资源
type ClusterAsset struct {
    Cluster *clusterv1.Cluster
    Files   []*asset.File
}

func (c *ClusterAsset) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &installconfig.ClusterID{},
        &BareMetalClusterAsset{},
    }
}

func (c *ClusterAsset) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    clusterID := &installconfig.ClusterID{}
    bareMetalCluster := &BareMetalClusterAsset{}
    parents.Get(installConfig, clusterID, bareMetalCluster)
    
    c.Cluster = &clusterv1.Cluster{
        TypeMeta: metav1.TypeMeta{
            APIVersion: clusterv1.GroupVersion.String(),
            Kind:       "Cluster",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      clusterID.Name,
            Namespace: "default",
        },
        Spec: clusterv1.ClusterSpec{
            ControlPlaneEndpoint: clusterv1.APIEndpoint{
                Host: installConfig.Config.Spec.ControlPlaneEndpoint,
                Port: 6443,
            },
            InfrastructureRef: &corev1.ObjectReference{
                APIVersion: "infrastructure.cluster.x-k8s.io/v1alpha1",
                Kind:       "BareMetalCluster",
                Name:       clusterID.Name,
            },
            ControlPlaneRef: &corev1.ObjectReference{
                APIVersion: "controlplane.cluster.x-k8s.io/v1beta1",
                Kind:       "KubeadmControlPlane",
                Name:       clusterID.Name + "-control-plane",
            },
        },
    }
    
    data, _ := yaml.Marshal(c.Cluster)
    c.Files = []*asset.File{
        {
            Filename: "cluster-api/cluster.yaml",
            Data:     data,
        },
    }
    
    return nil
}

// BareMetalClusterAsset 生成BareMetalCluster资源
type BareMetalClusterAsset struct {
    BareMetalCluster *v1alpha1.BareMetalCluster
    Files            []*asset.File
}

func (b *BareMetalClusterAsset) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &installconfig.ClusterID{},
    }
}

func (b *BareMetalClusterAsset) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    clusterID := &installconfig.ClusterID{}
    parents.Get(installConfig, clusterID)
    
    b.BareMetalCluster = &v1alpha1.BareMetalCluster{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "infrastructure.cluster.x-k8s.io/v1alpha1",
            Kind:       "BareMetalCluster",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      clusterID.Name,
            Namespace: "default",
        },
        Spec: v1alpha1.BareMetalClusterSpec{
            ControlPlaneEndpoint: clusterv1.APIEndpoint{
                Host: installConfig.Config.Spec.ControlPlaneEndpoint,
                Port: 6443,
            },
            BaseDomain: installConfig.Config.Spec.ClusterConfig.BaseDomain,
            NetworkSpec: v1alpha1.NetworkSpec{
                DNSDomain:     installConfig.Config.Spec.Networking.DNSDomain,
                PodSubnet:     installConfig.Config.Spec.Networking.PodSubnet,
                ServiceSubnet: installConfig.Config.Spec.Networking.ServiceSubnet,
                DNSServers:    installConfig.Config.Spec.Platform.DNSServers,
            },
            SSHSpec: v1alpha1.SSHSpec{
                User:           "root",
                PrivateKeyFile: "/root/.ssh/id_rsa",
            },
            ImageRegistry: v1alpha1.ImageRegistrySpec{
                Domain:    installConfig.Config.Spec.ImageRepo.Domain,
                Namespace: installConfig.Config.Spec.ImageRepo.Namespace,
            },
            UserProvidedDNS: installConfig.Config.Spec.Platform.UserProvidedDNS,
            UserProvidedLB:  installConfig.Config.Spec.Platform.UserProvidedLB,
        },
    }
    
    data, _ := yaml.Marshal(b.BareMetalCluster)
    b.Files = []*asset.File{
        {
            Filename: "cluster-api/baremetal-cluster.yaml",
            Data:     data,
        },
    }
    
    return nil
}

// KubeadmControlPlaneAsset 生成KubeadmControlPlane资源
type KubeadmControlPlaneAsset struct {
    ControlPlane *controlplanev1.KubeadmControlPlane
    Files        []*asset.File
}

func (k *KubeadmControlPlaneAsset) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &installconfig.ClusterID{},
        &tls.RootCA{},
        &tls.EtcdCA{},
        &tls.ServiceAccountKey{},
        &KubeadmConfigTemplateAsset{},
    }
}

func (k *KubeadmControlPlaneAsset) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    clusterID := &installconfig.ClusterID{}
    rootCA := &tls.RootCA{}
    etcdCA := &tls.EtcdCA{}
    saKey := &tls.ServiceAccountKey{}
    parents.Get(installConfig, clusterID, rootCA, etcdCA, saKey)
    
    masterNodes := getMasterNodes(installConfig)
    
    k.ControlPlane = &controlplanev1.KubeadmControlPlane{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "controlplane.cluster.x-k8s.io/v1beta1",
            Kind:       "KubeadmControlPlane",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      clusterID.Name + "-control-plane",
            Namespace: "default",
        },
        Spec: controlplanev1.KubeadmControlPlaneSpec{
            Replicas: pointer.Int32Ptr(int32(len(masterNodes))),
            Version:  installConfig.Config.Spec.KubernetesVersion,
            MachineTemplate: controlplanev1.KubeadmControlPlaneMachineTemplate{
                InfrastructureRef: corev1.ObjectReference{
                    APIVersion: "infrastructure.cluster.x-k8s.io/v1alpha1",
                    Kind:       "BareMetalMachineTemplate",
                    Name:       clusterID.Name + "-control-plane",
                },
            },
            KubeadmConfigSpec: bootstrapv1.KubeadmConfigSpec{
                InitConfiguration: &bootstrapv1.InitConfiguration{
                    LocalAPIEndpoint: bootstrapv1.APIEndpoint{
                        AdvertiseAddress: "$(hostname -I | awk '{print $1}')",
                        BindPort:         6443,
                    },
                    NodeRegistration: bootstrapv1.NodeRegistrationOptions{
                        CRISocket: "unix:///var/run/containerd/containerd.sock",
                        KubeletExtraArgs: map[string]string{
                            "cgroup-driver": "systemd",
                        },
                    },
                },
                ClusterConfiguration: &bootstrapv1.ClusterConfiguration{
                    KubernetesVersion: installConfig.Config.Spec.KubernetesVersion,
                    ControlPlaneEndpoint: fmt.Sprintf("%s:6443", 
                        installConfig.Config.Spec.ControlPlaneEndpoint),
                    Networking: bootstrapv1.Networking{
                        DNSDomain:     installConfig.Config.Spec.Networking.DNSDomain,
                        PodSubnet:     installConfig.Config.Spec.Networking.PodSubnet,
                        ServiceSubnet: installConfig.Config.Spec.Networking.ServiceSubnet,
                    },
                    Etcd: bootstrapv1.Etcd{
                        Local: &bootstrapv1.LocalEtcd{
                            DataDir: "/var/lib/etcd",
                            ServerCertSANs: []string{
                                installConfig.Config.Spec.ControlPlaneEndpoint,
                            },
                            PeerCertSANs: []string{
                                installConfig.Config.Spec.ControlPlaneEndpoint,
                            },
                        },
                    },
                    ImageRepository: installConfig.Config.Spec.ImageRepo.Domain + "/kubernetes",
                    CertificatesDir: "/etc/kubernetes/pki",
                },
                Files: k.generateCertificateFiles(rootCA, etcdCA, saKey),
                PreKubeadmCommands: []string{
                    "systemctl enable containerd",
                    "systemctl start containerd",
                    "systemctl enable kubelet",
                },
                PostKubeadmCommands: []string{
                    "mkdir -p $HOME/.kube",
                    "cp -i /etc/kubernetes/admin.conf $HOME/.kube/config",
                    "chown $(id -u):$(id -g) $HOME/.kube/config",
                },
            },
        },
    }
    
    data, _ := yaml.Marshal(k.ControlPlane)
    k.Files = []*asset.File{
        {
            Filename: "cluster-api/kubeadm-control-plane.yaml",
            Data:     data,
        },
    }
    
    return nil
}

func (k *KubeadmControlPlaneAsset) generateCertificateFiles(
    rootCA *tls.RootCA,
    etcdCA *tls.EtcdCA,
    saKey *tls.ServiceAccountKey,
) []bootstrapv1.File {
    return []bootstrapv1.File{
        {
            Path:        "/etc/kubernetes/pki/ca.crt",
            Owner:       "root:root",
            Permissions: "0644",
            Content:     string(rootCA.Cert()),
        },
        {
            Path:        "/etc/kubernetes/pki/ca.key",
            Owner:       "root:root",
            Permissions: "0600",
            Content:     string(rootCA.Key()),
        },
        {
            Path:        "/etc/kubernetes/pki/etcd/ca.crt",
            Owner:       "root:root",
            Permissions: "0644",
            Content:     string(etcdCA.Cert()),
        },
        {
            Path:        "/etc/kubernetes/pki/etcd/ca.key",
            Owner:       "root:root",
            Permissions: "0600",
            Content:     string(etcdCA.Key()),
        },
        {
            Path:        "/etc/kubernetes/pki/sa.pub",
            Owner:       "root:root",
            Permissions: "0644",
            Content:     string(saKey.PublicKey()),
        },
        {
            Path:        "/etc/kubernetes/pki/sa.key",
            Owner:       "root:root",
            Permissions: "0600",
            Content:     string(saKey.Key()),
        },
    }
}

// MachineDeploymentAsset 生成MachineDeployment资源
type MachineDeploymentAsset struct {
    MachineDeployment *clusterv1.MachineDeployment
    Files             []*asset.File
}

func (m *MachineDeploymentAsset) Dependencies() []asset.Asset {
    return []asset.Asset{
        &installconfig.InstallConfig{},
        &installconfig.ClusterID{},
        &BareMetalMachineTemplateAsset{},
        &KubeadmConfigTemplateAsset{},
    }
}

func (m *MachineDeploymentAsset) Generate(ctx context.Context, parents asset.Parents) error {
    installConfig := &installconfig.InstallConfig{}
    clusterID := &installconfig.ClusterID{}
    parents.Get(installConfig, clusterID)
    
    workerNodes := getWorkerNodes(installConfig)
    
    m.MachineDeployment = &clusterv1.MachineDeployment{
        TypeMeta: metav1.TypeMeta{
            APIVersion: clusterv1.GroupVersion.String(),
            Kind:       "MachineDeployment",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      clusterID.Name + "-worker",
            Namespace: "default",
        },
        Spec: clusterv1.MachineDeploymentSpec{
            Replicas: pointer.Int32Ptr(int32(len(workerNodes))),
            Selector: metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "cluster.x-k8s.io/cluster-name": clusterID.Name,
                    "node-role":                     "worker",
                },
            },
            ClusterName: clusterID.Name,
            Template: clusterv1.MachineTemplateSpec{
                Spec: clusterv1.MachineSpec{
                    ClusterName: clusterID.Name,
                    Bootstrap: clusterv1.Bootstrap{
                        ConfigRef: &corev1.ObjectReference{
                            APIVersion: "bootstrap.cluster.x-k8s.io/v1beta1",
                            Kind:       "KubeadmConfigTemplate",
                            Name:       clusterID.Name + "-worker",
                        },
                    },
                    InfrastructureRef: corev1.ObjectReference{
                        APIVersion: "infrastructure.cluster.x-k8s.io/v1alpha1",
                        Kind:       "BareMetalMachineTemplate",
                        Name:       clusterID.Name + "-worker",
                    },
                    Version: pointer.StringPtr(installConfig.Config.Spec.KubernetesVersion),
                },
            },
        },
    }
    
    data, _ := yaml.Marshal(m.MachineDeployment)
    m.Files = []*asset.File{
        {
            Filename: "cluster-api/machine-deployment.yaml",
            Data:     data,
        },
    }
    
    return nil
}
```
### 四、Infrastructure Provider实现
#### 4.1 BareMetal Infrastructure Provider
```go
// controllers/baremetalcluster_controller.go
package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/record"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
    "sigs.k8s.io/cluster-api/util"
    "sigs.k8s.io/cluster-api/util/conditions"
    "sigs.k8s.io/cluster-api/util/patch"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    infrav1alpha1 "github.com/bocloud/bke/infrastructure/baremetal/api/v1alpha1"
)

// BareMetalClusterReconciler reconciles a BareMetalCluster object
type BareMetalClusterReconciler struct {
    client.Client
    Log      logr.Logger
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

func (r *BareMetalClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("baremetalcluster", req.NamespacedName)
    
    // 1. 获取BareMetalCluster
    bareMetalCluster := &infrav1alpha1.BareMetalCluster{}
    if err := r.Get(ctx, req.NamespacedName, bareMetalCluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 获取关联的Cluster
    cluster, err := util.GetOwnerCluster(ctx, r.Client, bareMetalCluster.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if cluster == nil {
        log.Info("Waiting for Cluster Controller to set OwnerRef on BareMetalCluster")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 3. 创建patch helper
    patchHelper, err := patch.NewHelper(bareMetalCluster, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 执行reconcile
    result, err := r.reconcile(ctx, log, cluster, bareMetalCluster)
    if err != nil {
        return result, err
    }
    
    // 5. 更新状态
    if err := patchHelper.Patch(ctx, bareMetalCluster); err != nil {
        return ctrl.Result{}, err
    }
    
    return result, nil
}

func (r *BareMetalClusterReconciler) reconcile(
    ctx context.Context,
    log logr.Logger,
    cluster *clusterv1.Cluster,
    bareMetalCluster *infrav1alpha1.BareMetalCluster,
) (ctrl.Result, error) {
    // 1. 初始化状态
    conditions.MarkFalse(bareMetalCluster, infrav1alpha1.ClusterReady, "Initializing", clusterv1.ConditionSeverityInfo, "")
    
    // 2. 验证配置
    if err := r.validateCluster(bareMetalCluster); err != nil {
        conditions.MarkFalse(bareMetalCluster, infrav1alpha1.ClusterReady, "InvalidConfiguration", clusterv1.ConditionSeverityError, err.Error())
        return ctrl.Result{}, err
    }
    
    // 3. 检查基础设施是否就绪
    // UPI场景：用户已经提供了基础设施，直接标记为就绪
    bareMetalCluster.Status.Ready = true
    conditions.MarkTrue(bareMetalCluster, infrav1alpha1.ClusterReady)
    
    // 4. 设置控制平面端点
    bareMetalCluster.Spec.ControlPlaneEndpoint = cluster.Spec.ControlPlaneEndpoint
    
    log.Info("BareMetalCluster is ready")
    return ctrl.Result{}, nil
}

func (r *BareMetalClusterReconciler) validateCluster(bareMetalCluster *infrav1alpha1.BareMetalCluster) error {
    if bareMetalCluster.Spec.ControlPlaneEndpoint.Host == "" {
        return fmt.Errorf("control plane endpoint is required")
    }
    
    if bareMetalCluster.Spec.BaseDomain == "" {
        return fmt.Errorf("base domain is required")
    }
    
    return nil
}

func (r *BareMetalClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&infrav1alpha1.BareMetalCluster{}).
        Complete(r)
}

// BareMetalMachineReconciler reconciles a BareMetalMachine object
type BareMetalMachineReconciler struct {
    client.Client
    Log           logr.Logger
    Scheme        *runtime.Scheme
    Recorder      record.EventRecorder
    SSHExecutor   *executor.SSHExecutor
}

func (r *BareMetalMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("baremetalmachine", req.NamespacedName)
    
    // 1. 获取BareMetalMachine
    bareMetalMachine := &infrav1alpha1.BareMetalMachine{}
    if err := r.Get(ctx, req.NamespacedName, bareMetalMachine); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 获取关联的Machine
    machine, err := util.GetOwnerMachine(ctx, r.Client, bareMetalMachine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if machine == nil {
        log.Info("Waiting for Machine Controller to set OwnerRef on BareMetalMachine")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 3. 获取关联的Cluster
    cluster, err := util.GetClusterFromMetadata(ctx, r.Client, bareMetalMachine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if cluster == nil {
        log.Info("Waiting for Cluster to be ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 4. 创建patch helper
    patchHelper, err := patch.NewHelper(bareMetalMachine, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. 执行reconcile
    result, err := r.reconcile(ctx, log, cluster, machine, bareMetalMachine)
    if err != nil {
        return result, err
    }
    
    // 6. 更新状态
    if err := patchHelper.Patch(ctx, bareMetalMachine); err != nil {
        return ctrl.Result{}, err
    }
    
    return result, nil
}

func (r *BareMetalMachineReconciler) reconcile(
    ctx context.Context,
    log logr.Logger,
    cluster *clusterv1.Cluster,
    machine *clusterv1.Machine,
    bareMetalMachine *infrav1alpha1.BareMetalMachine,
) (ctrl.Result, error) {
    // 1. 检查是否已删除
    if !bareMetalMachine.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, log, bareMetalMachine)
    }
    
    // 2. 添加finalizer
    if !controllerutil.ContainsFinalizer(bareMetalMachine, infrav1alpha1.MachineFinalizer) {
        controllerutil.AddFinalizer(bareMetalMachine, infrav1alpha1.MachineFinalizer)
    }
    
    // 3. 初始化状态
    conditions.MarkFalse(bareMetalMachine, infrav1alpha1.MachineReady, "Initializing", clusterv1.ConditionSeverityInfo, "")
    
    // 4. 验证机器配置
    if err := r.validateMachine(bareMetalMachine); err != nil {
        conditions.MarkFalse(bareMetalMachine, infrav1alpha1.MachineReady, "InvalidConfiguration", clusterv1.ConditionSeverityError, err.Error())
        return ctrl.Result{}, err
    }
    
    // 5. 连接到机器
    executor, err := r.connectToMachine(bareMetalMachine)
    if err != nil {
        conditions.MarkFalse(bareMetalMachine, infrav1alpha1.MachineReady, "SSHConnectionFailed", clusterv1.ConditionSeverityError, err.Error())
        return ctrl.Result{RequeueAfter: 10 * time.Second}, err
    }
    defer executor.Close()
    
    // 6. 检查节点状态
    nodeReady, err := r.checkNodeStatus(ctx, executor, bareMetalMachine)
    if err != nil {
        log.Error(err, "Failed to check node status")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    if nodeReady {
        // 7. 节点已就绪
        bareMetalMachine.Status.Ready = true
        conditions.MarkTrue(bareMetalMachine, infrav1alpha1.MachineReady)
        
        // 8. 设置ProviderID
        bareMetalMachine.Spec.ProviderID = fmt.Sprintf("baremetal://%s", bareMetalMachine.Spec.IPAddress)
        
        log.Info("BareMetalMachine is ready")
        return ctrl.Result{}, nil
    }
    
    // 9. 等待节点就绪
    conditions.MarkFalse(bareMetalMachine, infrav1alpha1.MachineReady, "WaitingForNode", clusterv1.ConditionSeverityInfo, "")
    return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}

func (r *BareMetalMachineReconciler) connectToMachine(bareMetalMachine *infrav1alpha1.BareMetalMachine) (*executor.SSHExecutor, error) {
    sshSpec := bareMetalMachine.Spec.SSHSpec
    
    var privateKey []byte
    var err error
    
    if sshSpec.PrivateKey != "" {
        privateKey = []byte(sshSpec.PrivateKey)
    } else if sshSpec.PrivateKeyFile != "" {
        privateKey, err = os.ReadFile(sshSpec.PrivateKeyFile)
        if err != nil {
            return nil, fmt.Errorf("failed to read SSH private key: %v", err)
        }
    } else {
        return nil, fmt.Errorf("SSH private key is required")
    }
    
    return executor.NewSSHExecutor(
        bareMetalMachine.Spec.IPAddress,
        22,
        sshSpec.User,
        privateKey,
    )
}

func (r *BareMetalMachineReconciler) checkNodeStatus(ctx context.Context, executor *executor.SSHExecutor, bareMetalMachine *infrav1alpha1.BareMetalMachine) (bool, error) {
    // 检查kubelet状态
    output, err := executor.Execute("systemctl is-active kubelet")
    if err != nil || strings.TrimSpace(output) != "active" {
        return false, nil
    }
    
    // 检查containerd状态
    output, err = executor.Execute("systemctl is-active containerd")
    if err != nil || strings.TrimSpace(output) != "active" {
        return false, nil
    }
    
    // 检查节点是否已加入集群
    output, err = executor.Execute("kubectl get nodes $(hostname) -o jsonpath='{.status.conditions[?(@.type==\"Ready\")].status}'")
    if err != nil || strings.TrimSpace(output) != "True" {
        return false, nil
    }
    
    return true, nil
}

func (r *BareMetalMachineReconciler) reconcileDelete(ctx context.Context, log logr.Logger, bareMetalMachine *infrav1alpha1.BareMetalMachine) (ctrl.Result, error) {
    log.Info("Reconciling BareMetalMachine delete")
    
    // 1. 连接到机器
    executor, err := r.connectToMachine(bareMetalMachine)
    if err != nil {
        log.Error(err, "Failed to connect to machine for deletion")
        controllerutil.RemoveFinalizer(bareMetalMachine, infrav1alpha1.MachineFinalizer)
        return ctrl.Result{}, nil
    }
    defer executor.Close()
    
    // 2. 执行kubeadm reset
    _, err = executor.Execute("kubeadm reset -f")
    if err != nil {
        log.Error(err, "Failed to execute kubeadm reset")
    }
    
    // 3. 清理配置文件
    _, err = executor.Execute("rm -rf /etc/kubernetes /var/lib/etcd /var/lib/kubelet")
    if err != nil {
        log.Error(err, "Failed to cleanup configuration files")
    }
    
    // 4. 移除finalizer
    controllerutil.RemoveFinalizer(bareMetalMachine, infrav1alpha1.MachineFinalizer)
    
    log.Info("BareMetalMachine deleted successfully")
    return ctrl.Result{}, nil
}

func (r *BareMetalMachineReconciler) validateMachine(bareMetalMachine *infrav1alpha1.BareMetalMachine) error {
    if bareMetalMachine.Spec.Hostname == "" {
        return fmt.Errorf("hostname is required")
    }
    
    if bareMetalMachine.Spec.IPAddress == "" {
        return fmt.Errorf("IP address is required")
    }
    
    return nil
}

func (r *BareMetalMachineReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&infrav1alpha1.BareMetalMachine{}).
        Complete(r)
}
```
### 五、Bootstrap Provider实现
#### 5.1 自定义Kubeadm Bootstrap Provider
```go
// controllers/bkekubeadmconfig_controller.go
package controllers

import (
    "context"
    "fmt"
    "time"
    
    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/record"
    clusterv1 "sigs.k8s.io/cluster-api/api/v1beta1"
    "sigs.k8s.io/cluster-api/util"
    "sigs.k8s.io/cluster-api/util/conditions"
    "sigs.k8s.io/cluster-api/util/patch"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    bootstrapv1alpha1 "github.com/bocloud/bke/bootstrap/kubeadm/api/v1alpha1"
)

// BKEKubeadmConfigReconciler reconciles a BKEKubeadmConfig object
type BKEKubeadmConfigReconciler struct {
    client.Client
    Log      logr.Logger
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

func (r *BKEKubeadmConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("bkekubeadmconfig", req.NamespacedName)
    
    // 1. 获取BKEKubeadmConfig
    config := &bootstrapv1alpha1.BKEKubeadmConfig{}
    if err := r.Get(ctx, req.NamespacedName, config); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 获取关联的Machine
    machine, err := util.GetOwnerMachine(ctx, r.Client, config.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if machine == nil {
        log.Info("Waiting for Machine Controller to set OwnerRef on BKEKubeadmConfig")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 3. 获取关联的Cluster
    cluster, err := util.GetClusterFromMetadata(ctx, r.Client, config.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    if cluster == nil {
        log.Info("Waiting for Cluster to be ready")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 4. 创建patch helper
    patchHelper, err := patch.NewHelper(config, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. 执行reconcile
    result, err := r.reconcile(ctx, log, cluster, machine, config)
    if err != nil {
        return result, err
    }
    
    // 6. 更新状态
    if err := patchHelper.Patch(ctx, config); err != nil {
        return ctrl.Result{}, err
    }
    
    return result, nil
}

func (r *BKEKubeadmConfigReconciler) reconcile(
    ctx context.Context,
    log logr.Logger,
    cluster *clusterv1.Cluster,
    machine *clusterv1.Machine,
    config *bootstrapv1alpha1.BKEKubeadmConfig,
) (ctrl.Result, error) {
    // 1. 检查是否已生成配置
    if config.Status.Ready {
        log.Info("BKEKubeadmConfig is already ready")
        return ctrl.Result{}, nil
    }
    
    // 2. 初始化状态
    conditions.MarkFalse(config, bootstrapv1alpha1.ConfigReady, "Initializing", clusterv1.ConditionSeverityInfo, "")
    
    // 3. 生成bootstrap配置
    bootstrapData, err := r.generateBootstrapData(ctx, cluster, machine, config)
    if err != nil {
        conditions.MarkFalse(config, bootstrapv1alpha1.ConfigReady, "GenerationFailed", clusterv1.ConditionSeverityError, err.Error())
        return ctrl.Result{}, err
    }
    
    // 4. 创建Secret
    secret := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      config.Name,
            Namespace: config.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: bootstrapv1alpha1.GroupVersion.String(),
                    Kind:       "BKEKubeadmConfig",
                    Name:       config.Name,
                    UID:        config.UID,
                },
            },
        },
        Data: map[string][]byte{
            "value": bootstrapData,
        },
    }
    
    if err := r.Client.Create(ctx, secret); err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. 更新状态
    config.Status.DataSecretName = pointer.StringPtr(secret.Name)
    config.Status.Ready = true
    conditions.MarkTrue(config, bootstrapv1alpha1.ConfigReady)
    
    log.Info("BKEKubeadmConfig is ready")
    return ctrl.Result{}, nil
}

func (r *BKEKubeadmConfigReconciler) generateBootstrapData(
    ctx context.Context,
    cluster *clusterv1.Cluster,
    machine *clusterv1.Machine,
    config *bootstrapv1alpha1.BKEKubeadmConfig,
) ([]byte, error) {
    // 生成Shell脚本
    script := r.generateSetupScript(cluster, machine, config)
    
    return []byte(script), nil
}

func (r *BKEKubeadmConfigReconciler) generateSetupScript(
    cluster *clusterv1.Cluster,
    machine *clusterv1.Machine,
    config *bootstrapv1alpha1.BKEKubeadmConfig,
) string {
    script := `#!/bin/bash
set -euo pipefail

BKE_BOOTSTRAP_SCRIPT

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# 1. 系统配置
log "Configuring system..."
cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 2. 加载内核模块
log "Loading kernel modules..."
modprobe overlay
modprobe br_netfilter

# 3. 禁用Swap
log "Disabling swap..."
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 4. 安装依赖
log "Installing dependencies..."
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release containerd

# 5. 配置Containerd
log "Configuring containerd..."
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# 6. 安装Kubernetes
log "Installing Kubernetes..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# 7. 写入证书和配置文件
log "Writing certificates and configurations..."
mkdir -p /etc/kubernetes/pki
mkdir -p /etc/kubernetes/pki/etcd
`

    // 添加文件写入
    for _, file := range config.Spec.Files {
        script += fmt.Sprintf(`
cat > %s <<'EOF'
%s
EOF
chmod %s %s
`, file.Path, file.Content, file.Permissions, file.Path)
    }
    
    // 添加PreKubeadmCommands
    script += `
# 8. 执行PreKubeadmCommands
log "Executing pre-kubeadm commands..."
`
    for _, cmd := range config.Spec.PreKubeadmCommands {
        script += fmt.Sprintf("%s\n", cmd)
    }
    
    // 根据角色执行不同的kubeadm命令
    if config.Spec.InitConfiguration != nil {
        script += `
# 9. 初始化控制平面
log "Initializing control plane..."
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --upload-certs
`
    } else if config.Spec.JoinConfiguration != nil {
        script += `
# 9. 加入集群
log "Joining cluster..."
kubeadm join --config /etc/kubernetes/kubeadm-join-config.yaml
`
    }
    
    // 添加PostKubeadmCommands
    script += `
# 10. 执行PostKubeadmCommands
log "Executing post-kubeadm commands..."
`
    for _, cmd := range config.Spec.PostKubeadmCommands {
        script += fmt.Sprintf("%s\n", cmd)
    }
    
    script += `
log "Bootstrap completed successfully"
`
    
    return script
}

func (r *BKEKubeadmConfigReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&bootstrapv1alpha1.BKEKubeadmConfig{}).
        Complete(r)
}
```
### 六、安装流程实现
#### 6.1 Cluster API安装器
```go
package installer

import (
    "context"
    "fmt"
    "time"
    
    "github.com/bocloud/bke/pkg/asset"
    "github.com/bocloud/bke/pkg/asset/clusterapi"
    "github.com/bocloud/bke/pkg/asset/installconfig"
    "github.com/bocloud/bke/pkg/asset/tls"
    "github.com/bocloud/bke/pkg/state"
    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/kubernetes/scheme"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/client/config"
)

type ClusterAPIInstaller struct {
    assetStore   *asset.Store
    stateManager *state.Manager
    k8sClient    client.Client
    dynamicClient dynamic.Interface
    installDir   string
}

func NewClusterAPIInstaller(installDir string) (*ClusterAPIInstaller, error) {
    // 创建Kubernetes客户端
    cfg, err := config.GetConfig()
    if err != nil {
        return nil, fmt.Errorf("failed to get kubernetes config: %v", err)
    }
    
    k8sClient, err := client.New(cfg, client.Options{Scheme: scheme.Scheme})
    if err != nil {
        return nil, fmt.Errorf("failed to create kubernetes client: %v", err)
    }
    
    dynamicClient, err := dynamic.NewForConfig(cfg)
    if err != nil {
        return nil, fmt.Errorf("failed to create dynamic client: %v", err)
    }
    
    return &ClusterAPIInstaller{
        assetStore:     asset.NewStore(installDir),
        stateManager:   state.NewManager(installDir),
        k8sClient:      k8sClient,
        dynamicClient:  dynamicClient,
        installDir:     installDir,
    }, nil
}

func (c *ClusterAPIInstaller) Install(ctx context.Context) error {
    // 阶段1: 生成配置资产
    log.Info("Phase 1: Generating configuration assets...")
    if err := c.generateConfigAssets(ctx); err != nil {
        return fmt.Errorf("phase 1 failed: %v", err)
    }
    c.saveProgress("config", 100)
    
    // 阶段2: 生成证书资产
    log.Info("Phase 2: Generating certificate assets...")
    if err := c.generateCertificateAssets(ctx); err != nil {
        return fmt.Errorf("phase 2 failed: %v", err)
    }
    c.saveProgress("certificates", 100)
    
    // 阶段3: 生成Cluster API资源
    log.Info("Phase 3: Generating Cluster API resources...")
    if err := c.generateClusterAPIResources(ctx); err != nil {
        return fmt.Errorf("phase 3 failed: %v", err)
    }
    c.saveProgress("cluster-api", 100)
    
    // 阶段4: 应用Cluster API资源
    log.Info("Phase 4: Applying Cluster API resources...")
    if err := c.applyClusterAPIResources(ctx); err != nil {
        return fmt.Errorf("phase 4 failed: %v", err)
    }
    c.saveProgress("apply", 100)
    
    // 阶段5: 等待控制平面就绪
    log.Info("Phase 5: Waiting for control plane to be ready...")
    if err := c.waitForControlPlane(ctx); err != nil {
        return fmt.Errorf("phase 5 failed: %v", err)
    }
    c.saveProgress("control-plane", 100)
    
    // 阶段6: 等待Worker节点就绪
    log.Info("Phase 6: Waiting for worker nodes to be ready...")
    if err := c.waitForWorkerNodes(ctx); err != nil {
        return fmt.Errorf("phase 6 failed: %v", err)
    }
    c.saveProgress("workers", 100)
    
    // 阶段7: 安装网络插件
    log.Info("Phase 7: Installing network plugin...")
    if err := c.installNetworkPlugin(ctx); err != nil {
        return fmt.Errorf("phase 7 failed: %v", err)
    }
    c.saveProgress("network", 100)
    
    // 阶段8: 等待集群就绪
    log.Info("Phase 8: Waiting for cluster to be ready...")
    if err := c.waitForClusterReady(ctx); err != nil {
        return fmt.Errorf("phase 8 failed: %v", err)
    }
    c.saveProgress("cluster-ready", 100)
    
    log.Info("Installation completed successfully!")
    c.printSuccessMessage()
    
    return nil
}

func (c *ClusterAPIInstaller) generateClusterAPIResources(ctx context.Context) error {
    // 生成Cluster资源
    clusterAsset := &clusterapi.ClusterAsset{}
    if err := c.assetStore.Fetch(ctx, clusterAsset); err != nil {
        return err
    }
    
    // 生成BareMetalCluster资源
    bareMetalClusterAsset := &clusterapi.BareMetalClusterAsset{}
    if err := c.assetStore.Fetch(ctx, bareMetalClusterAsset); err != nil {
        return err
    }
    
    // 生成KubeadmControlPlane资源
    controlPlaneAsset := &clusterapi.KubeadmControlPlaneAsset{}
    if err := c.assetStore.Fetch(ctx, controlPlaneAsset); err != nil {
        return err
    }
    
    // 生成MachineDeployment资源
    machineDeploymentAsset := &clusterapi.MachineDeploymentAsset{}
    if err := c.assetStore.Fetch(ctx, machineDeploymentAsset); err != nil {
        return err
    }
    
    return nil
}

func (c *ClusterAPIInstaller) applyClusterAPIResources(ctx context.Context) error {
    // 1. 创建Namespace
    namespace := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{
            Name: "bke-system",
        },
    }
    if err := c.k8sClient.Create(ctx, namespace); err != nil {
        if !errors.IsAlreadyExists(err) {
            return err
        }
    }
    
    // 2. 应用BareMetalCluster
    bareMetalClusterGVR := schema.GroupVersionResource{
        Group:    "infrastructure.cluster.x-k8s.io",
        Version:  "v1alpha1",
        Resource: "baremetalclusters",
    }
    
    bareMetalClusterAsset := &clusterapi.BareMetalClusterAsset{}
    c.assetStore.Fetch(ctx, bareMetalClusterAsset)
    
    bareMetalClusterUnstructured := &unstructured.Unstructured{}
    if err := yaml.Unmarshal(bareMetalClusterAsset.Files[0].Data, bareMetalClusterUnstructured); err != nil {
        return err
    }
    
    _, err := c.dynamicClient.Resource(bareMetalClusterGVR).Namespace("default").Create(ctx, bareMetalClusterUnstructured, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }
    
    // 3. 应用Cluster
    clusterGVR := schema.GroupVersionResource{
        Group:    "cluster.x-k8s.io",
        Version:  "v1beta1",
        Resource: "clusters",
    }
    
    clusterAsset := &clusterapi.ClusterAsset{}
    c.assetStore.Fetch(ctx, clusterAsset)
    
    clusterUnstructured := &unstructured.Unstructured{}
    if err := yaml.Unmarshal(clusterAsset.Files[0].Data, clusterUnstructured); err != nil {
        return err
    }
    
    _, err = c.dynamicClient.Resource(clusterGVR).Namespace("default").Create(ctx, clusterUnstructured, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }
    
    // 4. 应用KubeadmControlPlane
    controlPlaneGVR := schema.GroupVersionResource{
        Group:    "controlplane.cluster.x-k8s.io",
        Version:  "v1beta1",
        Resource: "kubeadmcontrolplanes",
    }
    
    controlPlaneAsset := &clusterapi.KubeadmControlPlaneAsset{}
    c.assetStore.Fetch(ctx, controlPlaneAsset)
    
    controlPlaneUnstructured := &unstructured.Unstructured{}
    if err := yaml.Unmarshal(controlPlaneAsset.Files[0].Data, controlPlaneUnstructured); err != nil {
        return err
    }
    
    _, err = c.dynamicClient.Resource(controlPlaneGVR).Namespace("default").Create(ctx, controlPlaneUnstructured, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }
    
    // 5. 应用MachineDeployment
    machineDeploymentGVR := schema.GroupVersionResource{
        Group:    "cluster.x-k8s.io",
        Version:  "v1beta1",
        Resource: "machinedeployments",
    }
    
    machineDeploymentAsset := &clusterapi.MachineDeploymentAsset{}
    c.assetStore.Fetch(ctx, machineDeploymentAsset)
    
    machineDeploymentUnstructured := &unstructured.Unstructured{}
    if err := yaml.Unmarshal(machineDeploymentAsset.Files[0].Data, machineDeploymentUnstructured); err != nil {
        return err
    }
    
    _, err = c.dynamicClient.Resource(machineDeploymentGVR).Namespace("default").Create(ctx, machineDeploymentUnstructured, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }
    
    return nil
}

func (c *ClusterAPIInstaller) waitForControlPlane(ctx context.Context) error {
    clusterAsset := &clusterapi.ClusterAsset{}
    c.assetStore.Fetch(ctx, clusterAsset)
    
    clusterName := clusterAsset.Cluster.Name
    
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 检查KubeadmControlPlane状态
            controlPlaneGVR := schema.GroupVersionResource{
                Group:    "controlplane.cluster.x-k8s.io",
                Version:  "v1beta1",
                Resource: "kubeadmcontrolplanes",
            }
            
            controlPlane, err := c.dynamicClient.Resource(controlPlaneGVR).Namespace("default").Get(ctx, clusterName+"-control-plane", metav1.GetOptions{})
            if err != nil {
                log.Errorf("Failed to get control plane: %v", err)
                time.Sleep(10 * time.Second)
                continue
            }
            
            // 检查是否就绪
            status, ok := controlPlane.Object["status"].(map[string]interface{})
            if ok {
                ready, ok := status["ready"].(bool)
                if ok && ready {
                    log.Info("Control plane is ready")
                    return nil
                }
            }
            
            log.Info("Waiting for control plane to be ready...")
            time.Sleep(10 * time.Second)
        }
    }
}

func (c *ClusterAPIInstaller) waitForWorkerNodes(ctx context.Context) error {
    clusterAsset := &clusterapi.ClusterAsset{}
    c.assetStore.Fetch(ctx, clusterAsset)
    
    clusterName := clusterAsset.Cluster.Name
    
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 检查MachineDeployment状态
            machineDeploymentGVR := schema.GroupVersionResource{
                Group:    "cluster.x-k8s.io",
                Version:  "v1beta1",
                Resource: "machinedeployments",
            }
            
            machineDeployment, err := c.dynamicClient.Resource(machineDeploymentGVR).Namespace("default").Get(ctx, clusterName+"-worker", metav1.GetOptions{})
            if err != nil {
                log.Errorf("Failed to get machine deployment: %v", err)
                time.Sleep(10 * time.Second)
                continue
            }
            
            // 检查是否就绪
            status, ok := machineDeployment.Object["status"].(map[string]interface{})
            if ok {
                readyReplicas, ok := status["readyReplicas"].(int64)
                replicas, ok2 := status["replicas"].(int64)
                if ok && ok2 && readyReplicas == replicas {
                    log.Info("All worker nodes are ready")
                    return nil
                }
            }
            
            log.Info("Waiting for worker nodes to be ready...")
            time.Sleep(10 * time.Second)
        }
    }
}
```
### 七、配置文件示例
#### 7.1 install-config.yaml
```yaml
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: my-cluster
spec:
  clusterConfig:
    baseDomain: example.com
    platform:
      type: baremetal
      userProvidedDNS: true
      userProvidedLB: true
      dnsServers:
        - 192.168.1.10
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    controlPlaneEndpoint: 192.168.1.100:6443
    kubernetesVersion: v1.28.8
    imageRepo:
      domain: registry.example.com
      namespace: kubernetes
    
  nodes:
    - hostname: master-0
      ip: 192.168.1.100
      roles:
        - master
        - etcd
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
      
    - hostname: master-1
      ip: 192.168.1.101
      roles:
        - master
        - etcd
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
      
    - hostname: worker-0
      ip: 192.168.1.110
      roles:
        - worker
      sshUser: root
      sshKeyFile: /root/.ssh/id_rsa
```
### 八、命令行接口
```bash
# 完整安装
bkeadm install --config install-config.yaml

# 分步安装
bkeadm install config --config install-config.yaml
bkeadm install certs
bkeadm install cluster-api
bkeadm install apply

# 查看安装状态
bkeadm install status

# 查看集群状态
bkeadm cluster status

# 删除集群
bkeadm cluster delete
```
### 九、输出目录结构
```
my-cluster/
├── .bke_install_state.json      # 安装状态
├── install-config.yaml           # 安装配置
├── certs/                        # 证书
│   ├── ca.crt
│   ├── ca.key
│   └── ...
├── cluster-api/                  # Cluster API资源
│   ├── cluster.yaml
│   ├── baremetal-cluster.yaml
│   ├── kubeadm-control-plane.yaml
│   ├── machine-deployment.yaml
│   ├── baremetal-machine-template.yaml
│   └── kubeadm-config-template.yaml
└── kubeconfig                    # Kubeconfig
    └── admin.conf
```
### 十、关键优势
1. **声明式管理**：使用Cluster API的声明式方式管理集群生命周期
2. **标准化接口**：遵循Cluster API标准，易于与其他工具集成
3. **模块化设计**：Infrastructure Provider和Bootstrap Provider分离，易于扩展
4. **自动化运维**：自动处理节点扩缩容、升级、故障恢复
5. **状态管理**：Cluster API自动管理集群状态，无需额外状态持久化
6. **生态系统**：可以利用Cluster API生态系统的工具和最佳实践

这个基于Cluster API的方案将Asset-Based架构与Cluster API完美结合，既保留了Asset系统的优势，又获得了Cluster API的声明式管理和自动化能力。



        
