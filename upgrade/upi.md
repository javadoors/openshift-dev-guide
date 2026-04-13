    
# 节点组件安装与升级方案设计
## 一、方案概述
针对用户提供的操作系统场景，设计一个**声明式的节点组件管理系统**，支持容器运行时/kubelet/cni/csi的安装和灵活升级。
### 核心设计理念
1. **管理集群 + 目标节点分离架构**：控制器运行在管理集群，通过SSH/Agent方式管理目标节点
2. **声明式配置**：使用CRD定义期望状态，控制器负责状态协调
3. **版本化组件管理**：每个组件独立版本化，支持灵活升级策略
4. **幂等性操作**：所有操作可重复执行，不会产生副作用
## 二、架构设计
### 2.1 整体架构
```
┌─────────────────────────────────────────────────────────────┐
│                      管理集群                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           NodeComponentManager Controller             │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  │  │
│  │  │ NodeConfig  │  │ ComponentVer │  │ UpgradePolicy│  │  │
│  │  │ Controller  │  │  Controller  │  │  Controller  │  │  │
│  │  └─────────────┘  └──────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                           │                                  │
│                    Watch CRDs                               │
│                           │                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Component Registry (ConfigMap)            │  │
│  │   - containerd v1.7.0, v1.7.1, v1.7.2                 │  │
│  │   - kubelet v1.28.0, v1.28.1, v1.28.2                 │  │
│  │   - cni-plugins v1.3.0, v1.3.1                        │  │
│  │   - csi-driver v2.0.0, v2.1.0                         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                   SSH / Agent Communication
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
   │ Node-1  │        │ Node-2  │        │ Node-N  │
   │(未加入集群)│     │(未加入集群)│     │(未加入集群)│
   └─────────┘        └─────────┘        └─────────┘
```
### 2.2 核心组件
#### 2.2.1 CRD定义
**1. NodeConfig - 节点配置**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-192.168.1.10
spec:
  # 节点连接信息
  connection:
    host: 192.168.1.10
    port: 22
    sshKeySecret:
      name: node-ssh-key
      namespace: node-system
  
  # 操作系统信息
  os:
    type: linux
    distro: ubuntu
    version: "22.04"
    arch: amd64
  
  # 组件配置
  components:
    containerd:
      version: v1.7.2
      config:
        dataDir: /var/lib/containerd
        registry:
          mirrors:
            docker.io: https://registry.docker.io
        systemdCgroup: true
    
    kubelet:
      version: v1.28.2
      config:
        dataDir: /var/lib/kubelet
        configDir: /etc/kubernetes
        kubeletExtraArgs:
          max-pods: "110"
          pod-max-pids: "-1"
    
    cni:
      version: v1.3.0
      config:
        binDir: /opt/cni/bin
        confDir: /etc/cni/net.d
        pluginList:
          - bridge
          - loopback
          - portmap
    
    csi:
      version: v2.1.0
      config:
        driverName: csi-driver-hostpath
        dataDir: /var/lib/csi

status:
  # 节点状态
  phase: Ready  # Pending, Installing, Upgrading, Ready, Failed
  
  # 组件当前状态
  componentStatus:
    containerd:
      installedVersion: v1.7.2
      status: Ready
      lastUpdated: "2024-01-15T10:30:00Z"
    kubelet:
      installedVersion: v1.28.2
      status: Ready
      lastUpdated: "2024-01-15T10:35:00Z"
    cni:
      installedVersion: v1.3.0
      status: Ready
      lastUpdated: "2024-01-15T10:40:00Z"
    csi:
      installedVersion: v2.1.0
      status: Ready
      lastUpdated: "2024-01-15T10:45:00Z"
  
  # 操作系统实际信息
  osInfo:
    kernelVersion: 5.15.0-91-generic
    osImage: Ubuntu 22.04.3 LTS
  
  # 最后一次操作
  lastOperation:
    type: Install
    component: kubelet
    startTime: "2024-01-15T10:35:00Z"
    endTime: "2024-01-15T10:35:30Z"
    result: Success
```
**2. ComponentVersion - 组件版本定义**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.2
spec:
  componentName: containerd
  version: v1.7.2
  
  # 组件来源
  source:
    type: HTTP  # HTTP, OCI, Local
    url: https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz
    checksum: "sha256:xxxxx"
  
  # 安装脚本
  installScript: |
    #!/bin/bash
    set -e
    
    # 解压二进制文件
    tar -xzf /tmp/containerd.tar.gz -C /usr/local/bin
    
    # 创建配置目录
    mkdir -p /etc/containerd
    
    # 生成默认配置
    containerd config default > /etc/containerd/config.toml
    
    # 安装systemd服务
    cat > /etc/systemd/system/containerd.service <<EOF
    [Unit]
    Description=containerd container runtime
    Documentation=https://containerd.io
    After=network.target local-fs.target
    
    [Service]
    ExecStart=/usr/local/bin/containerd
    Type=notify
    Delegate=yes
    KillMode=process
    Restart=always
    RestartSec=5
    LimitNPROC=infinity
    LimitCORE=infinity
    LimitNOFILE=infinity
    TasksMax=infinity
    OOMScoreAdjust=-999
    
    [Install]
    WantedBy=multi-user.target
    EOF
    
    systemctl daemon-reload
    systemctl enable containerd
    systemctl start containerd
  
  # 升级脚本
  upgradeScript: |
    #!/bin/bash
    set -e
    
    # 停止旧版本
    systemctl stop containerd
    
    # 备份旧版本
    cp /usr/local/bin/containerd /usr/local/bin/containerd.bak
    
    # 安装新版本
    tar -xzf /tmp/containerd.tar.gz -C /usr/local/bin
    
    # 启动新版本
    systemctl start containerd
    
    # 验证
    if ! systemctl is-active containerd; then
      # 回滚
      mv /usr/local/bin/containerd.bak /usr/local/bin/containerd
      systemctl start containerd
      exit 1
    fi
  
  # 回滚脚本
  rollbackScript: |
    #!/bin/bash
    set -e
    
    systemctl stop containerd
    mv /usr/local/bin/containerd.bak /usr/local/bin/containerd
    systemctl start containerd
  
  # 健康检查
  healthCheck:
    command: "containerd --version"
    expectedOutput: "containerd github.com/containerd/containerd v1.7.2"
    timeout: 10s
  
  # 版本兼容性
  compatibility:
    os:
      - type: linux
        distro: [ubuntu, centos, debian]
        versions: ["20.04", "22.04", "7", "8", "9"]
        arch: [amd64, arm64]
    
    dependencies:
      - component: kubelet
        versionRange: ">=1.27.0"
      - component: cni
        versionRange: ">=1.2.0"
  
  # 升级路径
  upgradePath:
    fromVersions:
      - v1.7.0
      - v1.7.1
    preUpgradeCheck: |
      #!/bin/bash
      # 检查是否有运行中的容器
      running=$(ctr containers list -q | wc -l)
      if [ $running -gt 0 ]; then
        echo "Warning: $running containers are running"
      fi
```
**3. UpgradePolicy - 升级策略**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: UpgradePolicy
metadata:
  name: rolling-upgrade-policy
spec:
  # 升级策略类型
  type: RollingUpdate  # RollingUpdate, Batch, InPlace
  
  # 滚动更新配置
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0
    batchSize: 1
  
  # 升级顺序
  upgradeOrder:
    - containerd
    - cni
    - kubelet
    - csi
  
  # 前置检查
  preCheck:
    # 节点健康检查
    nodeHealthCheck:
      cpuThreshold: 80%
      memoryThreshold: 80%
      diskThreshold: 85%
    
    # 组件健康检查
    componentHealthCheck:
      enabled: true
      timeout: 30s
  
  # 升级超时
  timeout:
    perComponent: 5m
    perNode: 30m
    total: 2h
  
  # 失败处理
  failurePolicy:
    action: Pause  # Pause, Rollback, Continue
    maxRetries: 3
    retryInterval: 1m
  
  # 回滚配置
  rollback:
    enabled: true
    autoRollbackOnFailure: true
    timeout: 10m
```
## 三、详细设计
### 3.1 控制器设计
#### 3.1.1 NodeConfig Controller
```go
type NodeConfigReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    
    // SSH客户端管理器
    sshManager *SSHManager
    
    // 组件安装器
    componentInstaller *ComponentInstaller
    
    // 版本管理器
    versionManager *VersionManager
}

func (r *NodeConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    nodeConfig := &nodecomponentv1alpha1.NodeConfig{}
    if err := r.Get(ctx, req.NamespacedName, nodeConfig); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 1. 验证节点连接
    if err := r.validateNodeConnection(ctx, nodeConfig); err != nil {
        r.updateStatus(ctx, nodeConfig, PhaseConnectionFailed, err.Error())
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    // 2. 收集节点信息
    if err := r.collectNodeInfo(ctx, nodeConfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. 检测组件状态
    if err := r.detectComponentStatus(ctx, nodeConfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 协调组件状态
    if result, err := r.reconcileComponents(ctx, nodeConfig); err != nil {
        return ctrl.Result{}, err
    } else if result.Requeue {
        return result, nil
    }
    
    // 5. 更新最终状态
    r.updateStatus(ctx, nodeConfig, PhaseReady, "")
    
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

// 协调组件状态
func (r *NodeConfigReconciler) reconcileComponents(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) (ctrl.Result, error) {
    components := []string{"containerd", "cni", "kubelet", "csi"}
    
    for _, componentName := range components {
        desired := nodeConfig.Spec.Components[componentName]
        current := nodeConfig.Status.ComponentStatus[componentName]
        
        // 检查是否需要安装或升级
        if current == nil || current.InstalledVersion != desired.Version {
            // 获取组件版本定义
            componentVersion, err := r.versionManager.GetComponentVersion(ctx, componentName, desired.Version)
            if err != nil {
                return ctrl.Result{}, fmt.Errorf("get component version failed: %v", err)
            }
            
            // 执行安装或升级
            if current == nil {
                if err := r.installComponent(ctx, nodeConfig, componentVersion, desired); err != nil {
                    return ctrl.Result{}, err
                }
            } else {
                if err := r.upgradeComponent(ctx, nodeConfig, componentVersion, desired); err != nil {
                    return ctrl.Result{}, err
                }
            }
            
            // 更新状态
            r.updateComponentStatus(ctx, nodeConfig, componentName, desired.Version, PhaseInstalling)
            return ctrl.Result{Requeue: true}, nil
        }
        
        // 检查组件健康状态
        if err := r.healthCheckComponent(ctx, nodeConfig, componentName); err != nil {
            r.updateComponentStatus(ctx, nodeConfig, componentName, current.InstalledVersion, PhaseFailed)
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
    }
    
    return ctrl.Result{}, nil
}
```
#### 3.1.2 ComponentInstaller - 组件安装器
```go
type ComponentInstaller struct {
    sshManager *SSHManager
}

// 安装组件
func (i *ComponentInstaller) InstallComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig, 
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 1. 下载组件包
    if err := i.downloadComponent(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("download component failed: %v", err)
    }
    
    // 2. 验证校验和
    if err := i.verifyChecksum(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("verify checksum failed: %v", err)
    }
    
    // 3. 执行安装脚本
    if err := i.executeInstallScript(ctx, nodeConfig, componentVersion, config); err != nil {
        return fmt.Errorf("execute install script failed: %v", err)
    }
    
    // 4. 执行健康检查
    if err := i.healthCheck(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("health check failed: %v", err)
    }
    
    return nil
}

// 升级组件
func (i *ComponentInstaller) UpgradeComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 1. 执行前置检查
    if err := i.preUpgradeCheck(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("pre-upgrade check failed: %v", err)
    }
    
    // 2. 备份当前版本
    if err := i.backupCurrentVersion(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("backup failed: %v", err)
    }
    
    // 3. 下载新版本
    if err := i.downloadComponent(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("download component failed: %v", err)
    }
    
    // 4. 执行升级脚本
    if err := i.executeUpgradeScript(ctx, nodeConfig, componentVersion, config); err != nil {
        // 升级失败，执行回滚
        if rollbackErr := i.rollbackComponent(ctx, nodeConfig, componentVersion); rollbackErr != nil {
            return fmt.Errorf("upgrade failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("upgrade failed and rolled back: %v", err)
    }
    
    // 5. 执行健康检查
    if err := i.healthCheck(ctx, nodeConfig, componentVersion); err != nil {
        // 健康检查失败，执行回滚
        if rollbackErr := i.rollbackComponent(ctx, nodeConfig, componentVersion); rollbackErr != nil {
            return fmt.Errorf("health check failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("health check failed and rolled back: %v", err)
    }
    
    return nil
}

// 执行安装脚本
func (i *ComponentInstaller) executeInstallScript(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 获取SSH客户端
    sshClient, err := i.sshManager.GetClient(nodeConfig.Spec.Connection)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 创建临时目录
    tmpDir := fmt.Sprintf("/tmp/component-install-%s", componentVersion.Spec.ComponentName)
    if _, err := sshClient.ExecuteCommand(fmt.Sprintf("mkdir -p %s", tmpDir)); err != nil {
        return err
    }
    
    // 上传安装脚本
    scriptPath := fmt.Sprintf("%s/install.sh", tmpDir)
    if err := sshClient.UploadFile(scriptPath, []byte(componentVersion.Spec.InstallScript)); err != nil {
        return err
    }
    
    // 上传配置文件
    configPath := fmt.Sprintf("%s/config.yaml", tmpDir)
    configData, err := yaml.Marshal(config)
    if err != nil {
        return err
    }
    if err := sshClient.UploadFile(configPath, configData); err != nil {
        return err
    }
    
    // 执行安装脚本
    installCmd := fmt.Sprintf("cd %s && chmod +x install.sh && ./install.sh", tmpDir)
    output, err := sshClient.ExecuteCommand(installCmd)
    if err != nil {
        return fmt.Errorf("install script failed: %v, output: %s", err, output)
    }
    
    // 清理临时文件
    sshClient.ExecuteCommand(fmt.Sprintf("rm -rf %s", tmpDir))
    
    return nil
}
```
#### 3.1.3 SSHManager - SSH连接管理
```go
type SSHManager struct {
    client.Client
    cache map[string]*SSHClient
    mu    sync.RWMutex
}

type SSHClient struct {
    client *ssh.Client
    session *ssh.Session
}

func (m *SSHManager) GetClient(connection nodecomponentv1alpha1.Connection) (*SSHClient, error) {
    key := fmt.Sprintf("%s:%d", connection.Host, connection.Port)
    
    // 检查缓存
    m.mu.RLock()
    if client, ok := m.cache[key]; ok {
        m.mu.RUnlock()
        return client, nil
    }
    m.mu.RUnlock()
    
    // 获取SSH密钥
    sshKeySecret := &corev1.Secret{}
    if err := m.Get(context.Background(), types.NamespacedName{
        Name:      connection.SSHKeySecret.Name,
        Namespace: connection.SSHKeySecret.Namespace,
    }, sshKeySecret); err != nil {
        return nil, fmt.Errorf("get ssh key secret failed: %v", err)
    }
    
    // 创建SSH客户端
    config := &ssh.ClientConfig{
        User: connection.User,
        Auth: []ssh.AuthMethod{
            ssh.PublicKeysCallback(func() ([]ssh.Signer, error) {
                signer, err := ssh.ParsePrivateKey(sshKeySecret.Data["id_rsa"])
                if err != nil {
                    return nil, err
                }
                return []ssh.Signer{signer}, nil
            }),
        },
        HostKeyCallback: ssh.InsecureIgnoreHostKey(),
        Timeout:         10 * time.Second,
    }
    
    client, err := ssh.Dial("tcp", fmt.Sprintf("%s:%d", connection.Host, connection.Port), config)
    if err != nil {
        return nil, fmt.Errorf("ssh dial failed: %v", err)
    }
    
    sshClient := &SSHClient{client: client}
    
    // 缓存客户端
    m.mu.Lock()
    m.cache[key] = sshClient
    m.mu.Unlock()
    
    return sshClient, nil
}

func (c *SSHClient) ExecuteCommand(cmd string) (string, error) {
    session, err := c.client.NewSession()
    if err != nil {
        return "", err
    }
    defer session.Close()
    
    output, err := session.CombinedOutput(cmd)
    return string(output), err
}

func (c *SSHClient) UploadFile(remotePath string, data []byte) error {
    session, err := c.client.NewSession()
    if err != nil {
        return err
    }
    defer session.Close()
    
    // 使用scp上传文件
    go func() {
        w, _ := session.StdinPipe()
        defer w.Close()
        fmt.Fprintf(w, "C%04o %d %s\n", 0644, len(data), filepath.Base(remotePath))
        w.Write(data)
        fmt.Fprint(w, "\x00")
    }()
    
    return session.Run(fmt.Sprintf("scp -t %s", filepath.Dir(remotePath)))
}
```
### 3.2 升级流程设计
#### 3.2.1 升级检测
```go
type UpgradeDetector struct {
    client.Client
}

func (d *UpgradeDetector) DetectUpgrade(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) (*UpgradePlan, error) {
    plan := &UpgradePlan{
        NodeName: nodeConfig.Name,
    }
    
    // 检测每个组件是否需要升级
    for componentName, desired := range nodeConfig.Spec.Components {
        current := nodeConfig.Status.ComponentStatus[componentName]
        
        if current == nil {
            plan.AddComponentUpgrade(componentName, "", desired.Version, UpgradeTypeInstall)
        } else if current.InstalledVersion != desired.Version {
            // 检查升级路径
            if err := d.validateUpgradePath(ctx, componentName, current.InstalledVersion, desired.Version); err != nil {
                return nil, fmt.Errorf("invalid upgrade path for %s: %v", componentName, err)
            }
            
            plan.AddComponentUpgrade(componentName, current.InstalledVersion, desired.Version, UpgradeTypeUpgrade)
        }
    }
    
    return plan, nil
}

// 验证升级路径
func (d *UpgradeDetector) validateUpgradePath(ctx context.Context, componentName, fromVersion, toVersion string) error {
    // 获取目标版本定义
    targetVersion := &nodecomponentv1alpha1.ComponentVersion{}
    if err := d.Get(ctx, types.NamespacedName{
        Name: fmt.Sprintf("%s-%s", componentName, toVersion),
    }, targetVersion); err != nil {
        return err
    }
    
    // 检查升级路径
    for _, from := range targetVersion.Spec.UpgradePath.FromVersions {
        if from == fromVersion {
            return nil
        }
    }
    
    return fmt.Errorf("upgrade from %s to %s is not supported", fromVersion, toVersion)
}
```
#### 3.2.2 升级执行器
```go
type UpgradeExecutor struct {
    client.Client
    installer *ComponentInstaller
}

func (e *UpgradeExecutor) ExecuteUpgrade(ctx context.Context, plan *UpgradePlan, policy *nodecomponentv1alpha1.UpgradePolicy) error {
    // 1. 执行前置检查
    if err := e.preCheck(ctx, plan, policy); err != nil {
        return fmt.Errorf("pre-check failed: %v", err)
    }
    
    // 2. 按顺序执行升级
    for _, componentName := range policy.Spec.UpgradeOrder {
        if upgrade, ok := plan.ComponentUpgrades[componentName]; ok {
            // 获取节点配置
            nodeConfig := &nodecomponentv1alpha1.NodeConfig{}
            if err := e.Get(ctx, types.NamespacedName{Name: plan.NodeName}, nodeConfig); err != nil {
                return err
            }
            
            // 获取组件版本定义
            componentVersion := &nodecomponentv1alpha1.ComponentVersion{}
            if err := e.Get(ctx, types.NamespacedName{
                Name: fmt.Sprintf("%s-%s", componentName, upgrade.ToVersion),
            }, componentVersion); err != nil {
                return err
            }
            
            // 执行升级
            if upgrade.Type == UpgradeTypeInstall {
                if err := e.installer.InstallComponent(ctx, nodeConfig, componentVersion, 
                    nodeConfig.Spec.Components[componentName]); err != nil {
                    return fmt.Errorf("install %s failed: %v", componentName, err)
                }
            } else {
                if err := e.installer.UpgradeComponent(ctx, nodeConfig, componentVersion,
                    nodeConfig.Spec.Components[componentName]); err != nil {
                    return fmt.Errorf("upgrade %s failed: %v", componentName, err)
                }
            }
            
            // 更新状态
            if err := e.updateComponentStatus(ctx, nodeConfig, componentName, upgrade.ToVersion); err != nil {
                return err
            }
        }
    }
    
    return nil
}

// 前置检查
func (e *UpgradeExecutor) preCheck(ctx context.Context, plan *UpgradePlan, policy *nodecomponentv1alpha1.UpgradePolicy) error {
    // 获取节点配置
    nodeConfig := &nodecomponentv1alpha1.NodeConfig{}
    if err := e.Get(ctx, types.NamespacedName{Name: plan.NodeName}, nodeConfig); err != nil {
        return err
    }
    
    // 检查节点资源
    if policy.Spec.PreCheck.NodeHealthCheck.Enabled {
        if err := e.checkNodeResources(ctx, nodeConfig, policy.Spec.PreCheck.NodeHealthCheck); err != nil {
            return err
        }
    }
    
    // 检查组件健康
    if policy.Spec.PreCheck.ComponentHealthCheck.Enabled {
        for componentName := range plan.ComponentUpgrades {
            if err := e.checkComponentHealth(ctx, nodeConfig, componentName); err != nil {
                return fmt.Errorf("component %s health check failed: %v", componentName, err)
            }
        }
    }
    
    return nil
}
```
### 3.3 版本兼容性管理
```go
type VersionManager struct {
    client.Client
}

// 检查版本兼容性
func (m *VersionManager) CheckCompatibility(ctx context.Context, components map[string]nodecomponentv1alpha1.ComponentConfig) error {
    // 构建版本映射
    versionMap := make(map[string]string)
    for name, config := range components {
        versionMap[name] = config.Version
    }
    
    // 检查每个组件的依赖
    for componentName, config := range components {
        componentVersion, err := m.GetComponentVersion(ctx, componentName, config.Version)
        if err != nil {
            return err
        }
        
        // 检查依赖版本
        for _, dep := range componentVersion.Spec.Compatibility.Dependencies {
            actualVersion, ok := versionMap[dep.Component]
            if !ok {
                return fmt.Errorf("component %s depends on %s, but not specified", componentName, dep.Component)
            }
            
            // 验证版本范围
            if !m.versionInRange(actualVersion, dep.VersionRange) {
                return fmt.Errorf("component %s requires %s version %s, but got %s", 
                    componentName, dep.Component, dep.VersionRange, actualVersion)
            }
        }
    }
    
    return nil
}

// 版本范围检查
func (m *VersionManager) versionInRange(version, versionRange string) bool {
    // 解析版本范围 (例如: ">=1.27.0", ">=1.27.0 <1.29.0")
    // 使用 semver 库进行版本比较
    
    v, err := semver.Parse(version)
    if err != nil {
        return false
    }
    
    // 解析范围表达式
    rangeExpr := strings.Split(versionRange, " ")
    for _, expr := range rangeExpr {
        if !m.checkVersionExpression(v, expr) {
            return false
        }
    }
    
    return true
}

func (m *VersionManager) checkVersionExpression(v semver.Version, expr string) bool {
    if strings.HasPrefix(expr, ">=") {
        target := strings.TrimPrefix(expr, ">=")
        targetV, _ := semver.Parse(target)
        return v.GE(targetV)
    } else if strings.HasPrefix(expr, "<") {
        target := strings.TrimPrefix(expr, "<")
        targetV, _ := semver.Parse(target)
        return v.LT(targetV)
    } else if strings.HasPrefix(expr, ">") {
        target := strings.TrimPrefix(expr, ">")
        targetV, _ := semver.Parse(target)
        return v.GT(targetV)
    } else if strings.HasPrefix(expr, "<=") {
        target := strings.TrimPrefix(expr, "<=")
        targetV, _ := semver.Parse(target)
        return v.LE(targetV)
    } else if strings.HasPrefix(expr, "=") {
        target := strings.TrimPrefix(expr, "=")
        targetV, _ := semver.Parse(target)
        return v.Equals(targetV)
    }
    
    return false
}
```
## 四、使用示例
### 4.1 初始化节点
```yaml
# 1. 创建SSH密钥Secret
apiVersion: v1
kind: Secret
metadata:
  name: node-ssh-key
  namespace: node-system
type: Opaque
data:
  id_rsa: <base64-encoded-private-key>
---
# 2. 创建节点配置
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-192.168.1.10
  namespace: node-system
spec:
  connection:
    host: 192.168.1.10
    port: 22
    user: root
    sshKeySecret:
      name: node-ssh-key
      namespace: node-system
  
  os:
    type: linux
    distro: ubuntu
    version: "22.04"
    arch: amd64
  
  components:
    containerd:
      version: v1.7.2
      config:
        systemdCgroup: true
    
    kubelet:
      version: v1.28.2
      config:
        kubeletExtraArgs:
          max-pods: "110"
    
    cni:
      version: v1.3.0
      config:
        pluginList:
          - bridge
          - loopback
          - portmap
    
    csi:
      version: v2.1.0
      config:
        driverName: csi-driver-hostpath
```
### 4.2 升级组件
```yaml
# 修改NodeConfig，升级kubelet版本
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-192.168.1.10
  namespace: node-system
spec:
  # ... 其他配置保持不变
  components:
    containerd:
      version: v1.7.2  # 保持不变
    kubelet:
      version: v1.29.0  # 从v1.28.2升级到v1.29.0
    cni:
      version: v1.3.0
    csi:
      version: v2.1.0
```
### 4.3 批量升级多个节点
```yaml
# 创建升级策略
apiVersion: nodecomponent.io/v1alpha1
kind: UpgradePolicy
metadata:
  name: cluster-upgrade-policy
  namespace: node-system
spec:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    batchSize: 2
  
  upgradeOrder:
    - containerd
    - cni
    - kubelet
    - csi
  
  timeout:
    perComponent: 5m
    perNode: 30m
    total: 2h
  
  failurePolicy:
    action: Pause
    maxRetries: 3
---
# 创建批量升级任务
apiVersion: nodecomponent.io/v1alpha1
kind: UpgradeJob
metadata:
  name: cluster-upgrade-20240115
  namespace: node-system
spec:
  policyRef:
    name: cluster-upgrade-policy
  
  targetNodes:
    selector:
      matchLabels:
        cluster: production
  
  targetVersions:
    containerd: v1.7.3
    kubelet: v1.29.0
    cni: v1.3.0
    csi: v2.1.0
```
## 五、方案优势
### 5.1 灵活性
1. **声明式配置**：用户只需定义期望状态，控制器自动协调
2. **组件独立版本化**：每个组件可独立升级，不受其他组件限制
3. **可扩展性**：支持添加新组件，无需修改核心逻辑
### 5.2 可靠性
1. **自动回滚**：升级失败自动回滚到上一个稳定版本
2. **健康检查**：升级前后执行健康检查，确保组件可用
3. **版本兼容性检查**：升级前检查组件间版本兼容性
### 5.3 可维护性
1. **状态可视化**：通过CRD status查看组件安装和升级状态
2. **操作审计**：所有操作记录在lastOperation字段
3. **错误追踪**：详细的错误信息帮助定位问题
### 5.4 安全性
1. **SSH密钥管理**：SSH密钥存储在Kubernetes Secret中
2. **校验和验证**：下载的组件包验证校验和，防止篡改
3. **最小权限原则**：每个组件只授予必要的权限
## 六、与OpenShift Installer对比
| 特性 | OpenShift Installer | 本方案 |
|------|---------------------|--------|
| 组件安装方式 | 打包在OS镜像中 | 通过SSH远程安装 |
| 升级方式 | 升级整个OS | 独立升级各组件 |
| 适用场景 | OpenShift集群 | 通用Kubernetes集群 |
| 灵活性 | 较低（OS绑定） | 高（组件独立） |
| 复杂度 | 高（需要专用OS） | 中（标准OS即可） |
| 升级粒度 | OS级别 | 组件级别 |
## 七、总结
本方案通过**声明式CRD + SSH远程执行**的方式，实现了对非Kubernetes节点的组件安装和升级管理。核心优势包括：
1. **无需Pod/CRD依赖**：控制器运行在管理集群，通过SSH管理目标节点
2. **灵活的升级策略**：支持滚动更新、批量更新等多种策略
3. **完善的错误处理**：自动回滚、健康检查、版本兼容性验证
4. **可扩展架构**：支持添加新组件和自定义安装脚本

该方案适用于用户自行提供操作系统的场景，提供了比OpenShift Installer更灵活的组件管理能力。

 
# 节点组件安装与升级方案设计（支持多种部署模式）
## 一、方案概述
设计一个**可扩展的节点组件管理系统**，支持两种部署模式：
1. **远程安装模式**：用户完全提供操作系统，通过SSH远程安装组件
2. **镜像内置模式**：组件打包在OS镜像中，通过配置激活和升级
### 核心设计理念
1. **Provider抽象架构**：定义统一的组件管理接口，支持多种实现
2. **声明式配置**：使用CRD定义期望状态，控制器负责状态协调
3. **版本化组件管理**：每个组件独立版本化，支持灵活升级策略
4. **幂等性操作**：所有操作可重复执行，不会产生副作用
## 二、架构设计
### 2.1 整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                      管理集群                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              NodeComponentManager Controller                  │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │  │
│  │  │ NodeConfig  │  │ ComponentVer │  │ UpgradePolicy        │  │  │
│  │  │ Controller  │  │  Controller  │  │  Controller          │  │  │
│  │  └─────────────┘  └──────────────┘  └──────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                      Provider Interface                             │
│                              │                                      │
│         ┌────────────────────┼──────────────────┐                   │
│         │                    │                  │                   │
│  ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐            │
│  │   Remote    │     │   Image     │     │   Hybrid    │            │
│  │   Provider  │     │   Provider  │     │   Provider  │            │
│  │  (SSH安装)  │     │  (镜像内置) │     │  (混合模式) │            │
│  └─────────────┘     └─────────────┘     └─────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
         │                    │                    │
    SSH/Agent            配置激活/升级          SSH + 镜像组合
         │                    │                    │
    ┌────▼────┐          ┌────▼─────┐          ┌────▼─────┐
    │ Node-1  │          │ Node-2   │          │ Node-3   │
    │(用户OS) │          │(内置组件)│          │(混合模式)│
    └─────────┘          └──────────┘          └──────────┘
```
### 2.2 核心组件
#### 2.2.1 Provider接口定义
```go
package provider

import (
    "context"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type ProviderType string

const (
    ProviderTypeRemote ProviderType = "Remote"
    ProviderTypeImage  ProviderType = "Image"
    ProviderTypeHybrid ProviderType = "Hybrid"
)

type ComponentPhase string

const (
    PhasePending    ComponentPhase = "Pending"
    PhaseInstalling ComponentPhase = "Installing"
    PhaseUpgrading  ComponentPhase = "Upgrading"
    PhaseReady      ComponentPhase = "Ready"
    PhaseFailed     ComponentPhase = "Failed"
)

type ComponentStatus struct {
    Name             string
    InstalledVersion string
    Phase            ComponentPhase
    Message          string
    LastUpdated      string
}

type Provider interface {
    // Provider类型
    Type() ProviderType
    
    // 初始化Provider
    Initialize(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error
    
    // 检测组件状态
    DetectComponentStatus(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig, 
        componentName string) (*ComponentStatus, error)
    
    // 安装组件
    InstallComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
        componentVersion *nodecomponentv1alpha1.ComponentVersion, 
        config interface{}) error
    
    // 升级组件
    UpgradeComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
        componentVersion *nodecomponentv1alpha1.ComponentVersion,
        config interface{}) error
    
    // 健康检查
    HealthCheck(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
        componentName string) error
    
    // 回滚组件
    RollbackComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
        componentName string) error
    
    // 清理资源
    Cleanup(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error
}

type ProviderFactory interface {
    CreateProvider(providerType ProviderType) (Provider, error)
}
```
#### 2.2.2 CRD定义
**1. NodeConfig - 节点配置**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-192.168.1.10
spec:
  # 部署模式
  deploymentMode:
    type: Remote  # Remote, Image, Hybrid
    
    # Remote模式配置
    remote:
      connection:
        host: 192.168.1.10
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
    
    # Image模式配置
    image:
      osImageRef:
        name: ubuntu-22.04-with-components
        version: v1.0.0
      componentActivationMode: Auto  # Auto, Manual
      upgradeStrategy: InPlace  # InPlace, RollingUpdate
    
    # Hybrid模式配置
    hybrid:
      connection:
        host: 192.168.1.10
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      builtinComponents:
        - containerd
        - cni
      remoteComponents:
        - kubelet
        - csi
  
  # 操作系统信息
  os:
    type: linux
    distro: ubuntu
    version: "22.04"
    arch: amd64
  
  # 组件配置
  components:
    containerd:
      version: v1.7.2
      config:
        dataDir: /var/lib/containerd
        registry:
          mirrors:
            docker.io: https://registry.docker.io
        systemdCgroup: true
    
    kubelet:
      version: v1.28.2
      config:
        dataDir: /var/lib/kubelet
        configDir: /etc/kubernetes
        kubeletExtraArgs:
          max-pods: "110"
    
    cni:
      version: v1.3.0
      config:
        binDir: /opt/cni/bin
        confDir: /etc/cni/net.d
        pluginList:
          - bridge
          - loopback
          - portmap
    
    csi:
      version: v2.1.0
      config:
        driverName: csi-driver-hostpath
        dataDir: /var/lib/csi

status:
  phase: Ready
  
  # 部署模式状态
  deploymentModeStatus:
    type: Remote
    remoteStatus:
      connectionStatus: Connected
      lastConnectionTime: "2024-01-15T10:00:00Z"
  
  # 组件状态
  componentStatus:
    containerd:
      installedVersion: v1.7.2
      phase: Ready
      message: ""
      lastUpdated: "2024-01-15T10:30:00Z"
      deploymentSource: Remote  # Remote, Builtin, Hybrid
    kubelet:
      installedVersion: v1.28.2
      phase: Ready
      message: ""
      lastUpdated: "2024-01-15T10:35:00Z"
      deploymentSource: Remote
```
**2. OSImage - 操作系统镜像定义（Image模式专用）**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: OSImage
metadata:
  name: ubuntu-22.04-with-components
spec:
  # 镜像信息
  image:
    name: ubuntu-22.04-k8s-v1.28
    version: v1.0.0
    url: https://mirror.example.com/images/ubuntu-22.04-k8s-v1.28.qcow2
    checksum: "sha256:xxxxx"
  
  # 操作系统基础信息
  os:
    type: linux
    distro: ubuntu
    version: "22.04"
    arch: amd64
    kernelVersion: 5.15.0-91-generic
  
  # 内置组件列表
  builtinComponents:
    - name: containerd
      version: v1.7.2
      path: /opt/components/containerd
      systemdUnit: containerd.service
      configPath: /etc/containerd/config.toml
    
    - name: kubelet
      version: v1.28.2
      path: /opt/components/kubelet
      systemdUnit: kubelet.service
      configPath: /etc/kubernetes/kubelet.conf
    
    - name: cni
      version: v1.3.0
      path: /opt/cni/bin
      configPath: /etc/cni/net.d
    
    - name: csi
      version: v2.1.0
      path: /opt/components/csi
      systemdUnit: csi-driver.service
      configPath: /etc/csi/config.yaml
  
  # 组件升级支持
  upgradeSupport:
    enabled: true
    methods:
      - InPlace  # 原地升级
      - RollingUpdate  # 滚动升级（创建新节点）
    
    # 原地升级配置
    inPlaceUpgrade:
      supported: true
      backupEnabled: true
      backupPath: /var/lib/component-backup
    
    # 滚动升级配置
    rollingUpgrade:
      supported: true
      newNodeImageRef: ubuntu-22.04-k8s-v1.29
  
  # 激活配置
  activation:
    mode: Auto  # Auto, Manual
    autoActivateOn:
      - FirstBoot
      - ConfigChange
    
    # 激活脚本
    activateScript: |
      #!/bin/bash
      set -e
      
      # 激活所有内置组件
      for component in /opt/components/*; do
        component_name=$(basename $component)
        if [ -f "$component/activate.sh" ]; then
          bash "$component/activate.sh"
        fi
      done
      
      # 启用并启动systemd服务
      systemctl daemon-reload
      for service in containerd kubelet csi-driver; do
        systemctl enable $service
        systemctl start $service
      done

status:
  phase: Active
  
  # 镜像统计
  statistics:
    totalNodes: 10
    activeNodes: 8
    upgradingNodes: 2
  
  # 组件使用统计
  componentUsage:
    containerd:
      version: v1.7.2
      nodeCount: 10
    kubelet:
      version: v1.28.2
      nodeCount: 8
      upgradingCount: 2
```
**3. ComponentVersion - 组件版本定义**
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.2
spec:
  componentName: containerd
  version: v1.7.2
  
  # 支持的部署模式
  supportedDeploymentModes:
    - Remote
    - Image
    - Hybrid
  
  # Remote模式配置
  remoteConfig:
    source:
      type: HTTP
      url: https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz
      checksum: "sha256:xxxxx"
    
    installScript: |
      #!/bin/bash
      set -e
      tar -xzf /tmp/containerd.tar.gz -C /usr/local/bin
      mkdir -p /etc/containerd
      containerd config default > /etc/containerd/config.toml
      # ... systemd服务安装
    
    upgradeScript: |
      #!/bin/bash
      set -e
      systemctl stop containerd
      cp /usr/local/bin/containerd /usr/local/bin/containerd.bak
      tar -xzf /tmp/containerd.tar.gz -C /usr/local/bin
      systemctl start containerd
      # ... 验证和回滚逻辑
    
    rollbackScript: |
      #!/bin/bash
      systemctl stop containerd
      mv /usr/local/bin/containerd.bak /usr/local/bin/containerd
      systemctl start containerd
  
  # Image模式配置
  imageConfig:
    # 组件在镜像中的路径
    componentPath: /opt/components/containerd
    
    # 激活脚本
    activateScript: |
      #!/bin/bash
      set -e
      
      # 创建符号链接
      ln -sf /opt/components/containerd/bin/containerd /usr/local/bin/containerd
      ln -sf /opt/components/containerd/bin/ctr /usr/local/bin/ctr
      
      # 生成配置
      mkdir -p /etc/containerd
      if [ ! -f /etc/containerd/config.toml ]; then
        containerd config default > /etc/containerd/config.toml
      fi
      
      # 启用服务
      systemctl enable containerd
    
    # 原地升级脚本
    inPlaceUpgradeScript: |
      #!/bin/bash
      set -e
      
      OLD_VERSION=$1
      NEW_VERSION=$2
      
      # 备份当前版本
      BACKUP_DIR="/var/lib/component-backup/containerd-$OLD_VERSION"
      mkdir -p $BACKUP_DIR
      cp -r /opt/components/containerd $BACKUP_DIR/
      
      # 下载新版本
      curl -L -o /tmp/containerd-new.tar.gz https://example.com/containerd-$NEW_VERSION.tar.gz
      tar -xzf /tmp/containerd-new.tar.gz -C /opt/components/containerd-new
      
      # 停止服务
      systemctl stop containerd
      
      # 替换文件
      rm -rf /opt/components/containerd
      mv /opt/components/containerd-new /opt/components/containerd
      
      # 更新符号链接
      ln -sf /opt/components/containerd/bin/containerd /usr/local/bin/containerd
      
      # 启动服务
      systemctl start containerd
      
      # 验证
      if ! systemctl is-active containerd; then
        # 回滚
        rm -rf /opt/components/containerd
        mv $BACKUP_DIR/containerd /opt/components/containerd
        ln -sf /opt/components/containerd/bin/containerd /usr/local/bin/containerd
        systemctl start containerd
        exit 1
      fi
    
    # 配置更新脚本
    configUpdateScript: |
      #!/bin/bash
      CONFIG_FILE=/etc/containerd/config.toml
      
      # 合并用户配置
      if [ -f /tmp/user-config.yaml ]; then
        # 使用工具合并配置
        merge-config --base /opt/components/containerd/default-config.toml \
                     --override /tmp/user-config.yaml \
                     --output $CONFIG_FILE
      fi
      
      # 重启服务
      systemctl restart containerd
  
  # 健康检查
  healthCheck:
    command: "containerd --version"
    expectedOutput: "containerd github.com/containerd/containerd v1.7.2"
    timeout: 10s
  
  # 版本兼容性
  compatibility:
    os:
      - type: linux
        distro: [ubuntu, centos, debian]
        versions: ["20.04", "22.04", "7", "8", "9"]
        arch: [amd64, arm64]
    
    dependencies:
      - component: kubelet
        versionRange: ">=1.27.0"
      - component: cni
        versionRange: ">=1.2.0"
```
## 三、Provider实现
### 3.1 Remote Provider（远程安装模式）
```go
package provider

import (
    "context"
    "fmt"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type RemoteProvider struct {
    sshManager *SSHManager
}

func NewRemoteProvider(sshManager *SSHManager) *RemoteProvider {
    return &RemoteProvider{
        sshManager: sshManager,
    }
}

func (p *RemoteProvider) Type() ProviderType {
    return ProviderTypeRemote
}

func (p *RemoteProvider) Initialize(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // 验证SSH连接
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return fmt.Errorf("ssh connection failed: %v", err)
    }
    defer sshClient.Close()
    
    // 检查操作系统信息
    output, err := sshClient.ExecuteCommand("cat /etc/os-release")
    if err != nil {
        return fmt.Errorf("get os info failed: %v", err)
    }
    
    // 验证操作系统兼容性
    // ...
    
    return nil
}

func (p *RemoteProvider) DetectComponentStatus(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) (*ComponentStatus, error) {
    
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return nil, err
    }
    defer sshClient.Close()
    
    status := &ComponentStatus{
        Name: componentName,
    }
    
    // 检测组件是否安装
    switch componentName {
    case "containerd":
        output, err := sshClient.ExecuteCommand("containerd --version 2>/dev/null || echo 'not installed'")
        if err != nil {
            return nil, err
        }
        
        if strings.Contains(output, "not installed") {
            status.Phase = PhasePending
            status.Message = "Component not installed"
        } else {
            // 解析版本
            version := parseContainerdVersion(output)
            status.InstalledVersion = version
            status.Phase = PhaseReady
        }
    
    case "kubelet":
        output, err := sshClient.ExecuteCommand("kubelet --version 2>/dev/null || echo 'not installed'")
        if err != nil {
            return nil, err
        }
        
        if strings.Contains(output, "not installed") {
            status.Phase = PhasePending
        } else {
            version := parseKubeletVersion(output)
            status.InstalledVersion = version
            status.Phase = PhaseReady
        }
    
    // ... 其他组件
    }
    
    return status, nil
}

func (p *RemoteProvider) InstallComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 1. 下载组件包
    if err := p.downloadComponent(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("download component failed: %v", err)
    }
    
    // 2. 验证校验和
    if err := p.verifyChecksum(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("verify checksum failed: %v", err)
    }
    
    // 3. 执行安装脚本
    if err := p.executeInstallScript(ctx, sshClient, componentVersion, config); err != nil {
        return fmt.Errorf("execute install script failed: %v", err)
    }
    
    // 4. 执行健康检查
    if err := p.healthCheck(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("health check failed: %v", err)
    }
    
    return nil
}

func (p *RemoteProvider) UpgradeComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 1. 执行前置检查
    if err := p.preUpgradeCheck(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("pre-upgrade check failed: %v", err)
    }
    
    // 2. 备份当前版本
    if err := p.backupCurrentVersion(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("backup failed: %v", err)
    }
    
    // 3. 下载新版本
    if err := p.downloadComponent(ctx, sshClient, componentVersion); err != nil {
        return fmt.Errorf("download component failed: %v", err)
    }
    
    // 4. 执行升级脚本
    if err := p.executeUpgradeScript(ctx, sshClient, componentVersion, config); err != nil {
        // 升级失败，执行回滚
        if rollbackErr := p.rollbackComponent(ctx, sshClient, componentVersion); rollbackErr != nil {
            return fmt.Errorf("upgrade failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("upgrade failed and rolled back: %v", err)
    }
    
    // 5. 执行健康检查
    if err := p.healthCheck(ctx, sshClient, componentVersion); err != nil {
        // 健康检查失败，执行回滚
        if rollbackErr := p.rollbackComponent(ctx, sshClient, componentVersion); rollbackErr != nil {
            return fmt.Errorf("health check failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("health check failed and rolled back: %v", err)
    }
    
    return nil
}

func (p *RemoteProvider) HealthCheck(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 执行健康检查命令
    cmd := fmt.Sprintf("systemctl is-active %s", componentName)
    output, err := sshClient.ExecuteCommand(cmd)
    if err != nil {
        return err
    }
    
    if strings.TrimSpace(output) != "active" {
        return fmt.Errorf("component %s is not active", componentName)
    }
    
    return nil
}

func (p *RemoteProvider) RollbackComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    sshClient, err := p.sshManager.GetClient(nodeConfig.Spec.DeploymentMode.Remote.Connection)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 执行回滚脚本
    rollbackScript := fmt.Sprintf("/var/lib/component-backup/%s/rollback.sh", componentName)
    cmd := fmt.Sprintf("bash %s", rollbackScript)
    
    _, err = sshClient.ExecuteCommand(cmd)
    return err
}

func (p *RemoteProvider) Cleanup(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // 清理SSH连接缓存
    key := fmt.Sprintf("%s:%d", 
        nodeConfig.Spec.DeploymentMode.Remote.Connection.Host,
        nodeConfig.Spec.DeploymentMode.Remote.Connection.Port)
    
    p.sshManager.RemoveClient(key)
    return nil
}
```
### 3.2 Image Provider（镜像内置模式）
```go
package provider

import (
    "context"
    "fmt"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type ImageProvider struct {
    client client.Client
}

func NewImageProvider(client client.Client) *ImageProvider {
    return &ImageProvider{
        client: client,
    }
}

func (p *ImageProvider) Type() ProviderType {
    return ProviderTypeImage
}

func (p *ImageProvider) Initialize(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // 获取OS镜像定义
    osImage := &nodecomponentv1alpha1.OSImage{}
    if err := p.client.Get(ctx, types.NamespacedName{
        Name: nodeConfig.Spec.DeploymentMode.Image.OSImageRef.Name,
    }, osImage); err != nil {
        return fmt.Errorf("get os image failed: %v", err)
    }
    
    // 验证镜像是否包含所需组件
    for componentName := range nodeConfig.Spec.Components {
        found := false
        for _, builtin := range osImage.Spec.BuiltinComponents {
            if builtin.Name == componentName {
                found = true
                break
            }
        }
        
        if !found {
            return fmt.Errorf("component %s not found in os image", componentName)
        }
    }
    
    return nil
}

func (p *ImageProvider) DetectComponentStatus(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) (*ComponentStatus, error) {
    
    // 获取OS镜像定义
    osImage := &nodecomponentv1alpha1.OSImage{}
    if err := p.client.Get(ctx, types.NamespacedName{
        Name: nodeConfig.Spec.DeploymentMode.Image.OSImageRef.Name,
    }, osImage); err != nil {
        return nil, err
    }
    
    // 查找组件定义
    var builtinComponent *nodecomponentv1alpha1.BuiltinComponent
    for i, builtin := range osImage.Spec.BuiltinComponents {
        if builtin.Name == componentName {
            builtinComponent = &osImage.Spec.BuiltinComponents[i]
            break
        }
    }
    
    if builtinComponent == nil {
        return nil, fmt.Errorf("component %s not found in os image", componentName)
    }
    
    status := &ComponentStatus{
        Name:             componentName,
        InstalledVersion: builtinComponent.Version,
        Phase:            PhasePending,
    }
    
    // 检查组件是否已激活
    // 通过节点状态API或SSH检查
    if p.isComponentActivated(ctx, nodeConfig, componentName) {
        status.Phase = PhaseReady
    }
    
    return status, nil
}

func (p *ImageProvider) InstallComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // Image模式下，安装实际上是激活内置组件
    return p.activateComponent(ctx, nodeConfig, componentVersion, config)
}

func (p *ImageProvider) activateComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 根据激活模式执行不同操作
    activationMode := nodeConfig.Spec.DeploymentMode.Image.ComponentActivationMode
    
    switch activationMode {
    case "Auto":
        // 自动激活：组件在节点首次启动时自动激活
        // 这里只需要更新配置
        return p.updateComponentConfig(ctx, nodeConfig, componentVersion, config)
    
    case "Manual":
        // 手动激活：需要执行激活脚本
        return p.executeActivateScript(ctx, nodeConfig, componentVersion, config)
    
    default:
        return fmt.Errorf("unsupported activation mode: %s", activationMode)
    }
}

func (p *ImageProvider) UpgradeComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    upgradeStrategy := nodeConfig.Spec.DeploymentMode.Image.UpgradeStrategy
    
    switch upgradeStrategy {
    case "InPlace":
        // 原地升级：在当前节点上升级组件
        return p.inPlaceUpgrade(ctx, nodeConfig, componentVersion, config)
    
    case "RollingUpdate":
        // 滚动升级：创建新节点替换旧节点
        return p.rollingUpgrade(ctx, nodeConfig, componentVersion, config)
    
    default:
        return fmt.Errorf("unsupported upgrade strategy: %s", upgradeStrategy)
    }
}

func (p *ImageProvider) inPlaceUpgrade(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 1. 执行前置检查
    if err := p.preUpgradeCheck(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("pre-upgrade check failed: %v", err)
    }
    
    // 2. 备份当前版本
    if err := p.backupCurrentVersion(ctx, nodeConfig, componentVersion); err != nil {
        return fmt.Errorf("backup failed: %v", err)
    }
    
    // 3. 执行原地升级脚本
    if err := p.executeInPlaceUpgradeScript(ctx, nodeConfig, componentVersion, config); err != nil {
        // 升级失败，执行回滚
        if rollbackErr := p.rollbackComponent(ctx, nodeConfig, componentVersion); rollbackErr != nil {
            return fmt.Errorf("upgrade failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("upgrade failed and rolled back: %v", err)
    }
    
    // 4. 执行健康检查
    if err := p.HealthCheck(ctx, nodeConfig, componentVersion.Spec.ComponentName); err != nil {
        // 健康检查失败，执行回滚
        if rollbackErr := p.rollbackComponent(ctx, nodeConfig, componentVersion); rollbackErr != nil {
            return fmt.Errorf("health check failed: %v, rollback also failed: %v", err, rollbackErr)
        }
        return fmt.Errorf("health check failed and rolled back: %v", err)
    }
    
    return nil
}

func (p *ImageProvider) rollingUpgrade(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    // 获取OS镜像定义
    osImage := &nodecomponentv1alpha1.OSImage{}
    if err := p.client.Get(ctx, types.NamespacedName{
        Name: nodeConfig.Spec.DeploymentMode.Image.OSImageRef.Name,
    }, osImage); err != nil {
        return err
    }
    
    // 检查镜像是否支持滚动升级
    if !osImage.Spec.UpgradeSupport.RollingUpgrade.Supported {
        return fmt.Errorf("os image does not support rolling upgrade")
    }
    
    // 创建新节点配置
    newNodeConfig := nodeConfig.DeepCopy()
    newNodeConfig.Name = fmt.Sprintf("%s-new", nodeConfig.Name)
    newNodeConfig.Spec.DeploymentMode.Image.OSImageRef.Name = osImage.Spec.UpgradeSupport.RollingUpgrade.NewNodeImageRef
    
    // 创建新节点
    if err := p.client.Create(ctx, newNodeConfig); err != nil {
        return fmt.Errorf("create new node failed: %v", err)
    }
    
    // 等待新节点就绪
    if err := p.waitForNodeReady(ctx, newNodeConfig); err != nil {
        return fmt.Errorf("wait for new node ready failed: %v", err)
    }
    
    // 将新节点加入集群
    if err := p.joinNodeToCluster(ctx, newNodeConfig); err != nil {
        return fmt.Errorf("join new node to cluster failed: %v", err)
    }
    
    // 驱逐旧节点上的Pod
    if err := p.drainNode(ctx, nodeConfig); err != nil {
        return fmt.Errorf("drain old node failed: %v", err)
    }
    
    // 删除旧节点
    if err := p.client.Delete(ctx, nodeConfig); err != nil {
        return fmt.Errorf("delete old node failed: %v", err)
    }
    
    return nil
}

func (p *ImageProvider) HealthCheck(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    // 通过节点状态API或SSH检查组件健康状态
    // 这里简化实现，实际需要根据部署模式选择检查方式
    
    // 方式1: 通过SSH检查（如果有SSH访问权限）
    if nodeConfig.Spec.DeploymentMode.Remote != nil {
        // 使用SSH检查
        // ...
    }
    
    // 方式2: 通过节点状态API检查（如果节点已加入集群）
    if nodeConfig.Status.Phase == "Ready" {
        // 通过Kubernetes API检查
        // ...
    }
    
    return nil
}

func (p *ImageProvider) RollbackComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    // 执行回滚脚本
    rollbackScript := fmt.Sprintf("/var/lib/component-backup/%s/rollback.sh", componentName)
    
    // 通过SSH或节点代理执行
    // ...
    
    return nil
}

func (p *ImageProvider) Cleanup(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // Image模式通常不需要清理
    return nil
}
```
### 3.3 Hybrid Provider（混合模式）
```go
package provider

import (
    "context"
    "fmt"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type HybridProvider struct {
    remoteProvider *RemoteProvider
    imageProvider  *ImageProvider
}

func NewHybridProvider(remoteProvider *RemoteProvider, imageProvider *ImageProvider) *HybridProvider {
    return &HybridProvider{
        remoteProvider: remoteProvider,
        imageProvider:  imageProvider,
    }
}

func (p *HybridProvider) Type() ProviderType {
    return ProviderTypeHybrid
}

func (p *HybridProvider) Initialize(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // 初始化Remote Provider（用于远程组件）
    if err := p.remoteProvider.Initialize(ctx, nodeConfig); err != nil {
        return fmt.Errorf("initialize remote provider failed: %v", err)
    }
    
    // 初始化Image Provider（用于内置组件）
    if err := p.imageProvider.Initialize(ctx, nodeConfig); err != nil {
        return fmt.Errorf("initialize image provider failed: %v", err)
    }
    
    return nil
}

func (p *HybridProvider) DetectComponentStatus(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) (*ComponentStatus, error) {
    
    // 根据组件类型选择Provider
    provider := p.getProviderForComponent(nodeConfig, componentName)
    
    return provider.DetectComponentStatus(ctx, nodeConfig, componentName)
}

func (p *HybridProvider) InstallComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    provider := p.getProviderForComponent(nodeConfig, componentVersion.Spec.ComponentName)
    
    return provider.InstallComponent(ctx, nodeConfig, componentVersion, config)
}

func (p *HybridProvider) UpgradeComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentVersion *nodecomponentv1alpha1.ComponentVersion, config interface{}) error {
    
    provider := p.getProviderForComponent(nodeConfig, componentVersion.Spec.ComponentName)
    
    return provider.UpgradeComponent(ctx, nodeConfig, componentVersion, config)
}

func (p *HybridProvider) HealthCheck(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    provider := p.getProviderForComponent(nodeConfig, componentName)
    
    return provider.HealthCheck(ctx, nodeConfig, componentName)
}

func (p *HybridProvider) RollbackComponent(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig,
    componentName string) error {
    
    provider := p.getProviderForComponent(nodeConfig, componentName)
    
    return provider.RollbackComponent(ctx, nodeConfig, componentName)
}

func (p *HybridProvider) Cleanup(ctx context.Context, nodeConfig *nodecomponentv1alpha1.NodeConfig) error {
    // 清理两个Provider
    if err := p.remoteProvider.Cleanup(ctx, nodeConfig); err != nil {
        return err
    }
    
    return p.imageProvider.Cleanup(ctx, nodeConfig)
}

func (p *HybridProvider) getProviderForComponent(nodeConfig *nodecomponentv1alpha1.NodeConfig, 
    componentName string) Provider {
    
    hybrid := nodeConfig.Spec.DeploymentMode.Hybrid
    
    // 检查组件是否在远程组件列表中
    for _, name := range hybrid.RemoteComponents {
        if name == componentName {
            return p.remoteProvider
        }
    }
    
    // 默认使用Image Provider
    return p.imageProvider
}
```
## 四、控制器设计
### 4.1 Provider工厂
```go
package controller

import (
    "fmt"
    
    "nodecomponent.io/manager/provider"
)

type ProviderFactoryImpl struct {
    sshManager *SSHManager
    client     client.Client
}

func NewProviderFactory(sshManager *SSHManager, client client.Client) *ProviderFactoryImpl {
    return &ProviderFactoryImpl{
        sshManager: sshManager,
        client:     client,
    }
}

func (f *ProviderFactoryImpl) CreateProvider(providerType provider.ProviderType) (provider.Provider, error) {
    switch providerType {
    case provider.ProviderTypeRemote:
        return provider.NewRemoteProvider(f.sshManager), nil
    
    case provider.ProviderTypeImage:
        return provider.NewImageProvider(f.client), nil
    
    case provider.ProviderTypeHybrid:
        remoteProvider := provider.NewRemoteProvider(f.sshManager)
        imageProvider := provider.NewImageProvider(f.client)
        return provider.NewHybridProvider(remoteProvider, imageProvider), nil
    
    default:
        return nil, fmt.Errorf("unsupported provider type: %s", providerType)
    }
}
```
### 4.2 NodeConfig控制器
```go
package controller

import (
    "context"
    "fmt"
    "time"
    
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
    "nodecomponent.io/manager/provider"
)

type NodeConfigReconciler struct {
    client.Client
    Scheme          *runtime.Scheme
    providerFactory provider.ProviderFactory
}

func (r *NodeConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    nodeConfig := &nodecomponentv1alpha1.NodeConfig{}
    if err := r.Get(ctx, req.NamespacedName, nodeConfig); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 1. 创建Provider
    providerType := provider.ProviderType(nodeConfig.Spec.DeploymentMode.Type)
    prov, err := r.providerFactory.CreateProvider(providerType)
    if err != nil {
        r.updateStatus(ctx, nodeConfig, "ProviderError", err.Error())
        return ctrl.Result{}, err
    }
    
    // 2. 初始化Provider
    if err := prov.Initialize(ctx, nodeConfig); err != nil {
        r.updateStatus(ctx, nodeConfig, "InitializationFailed", err.Error())
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    // 3. 收集节点信息
    if err := r.collectNodeInfo(ctx, nodeConfig, prov); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 检测组件状态
    if err := r.detectComponentStatus(ctx, nodeConfig, prov); err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. 协调组件状态
    if result, err := r.reconcileComponents(ctx, nodeConfig, prov); err != nil {
        return ctrl.Result{}, err
    } else if result.Requeue {
        return result, nil
    }
    
    // 6. 更新最终状态
    r.updateStatus(ctx, nodeConfig, "Ready", "")
    
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *NodeConfigReconciler) reconcileComponents(ctx context.Context, 
    nodeConfig *nodecomponentv1alpha1.NodeConfig, prov provider.Provider) (ctrl.Result, error) {
    
    components := []string{"containerd", "cni", "kubelet", "csi"}
    
    for _, componentName := range components {
        desired, ok := nodeConfig.Spec.Components[componentName]
        if !ok {
            continue
        }
        
        current := nodeConfig.Status.ComponentStatus[componentName]
        
        // 检查是否需要安装或升级
        if current == nil || current.InstalledVersion != desired.Version {
            // 获取组件版本定义
            componentVersion := &nodecomponentv1alpha1.ComponentVersion{}
            if err := r.Get(ctx, types.NamespacedName{
                Name: fmt.Sprintf("%s-%s", componentName, desired.Version),
            }, componentVersion); err != nil {
                return ctrl.Result{}, fmt.Errorf("get component version failed: %v", err)
            }
            
            // 执行安装或升级
            if current == nil {
                if err := prov.InstallComponent(ctx, nodeConfig, componentVersion, desired); err != nil {
                    r.updateComponentStatus(ctx, nodeConfig, componentName, desired.Version, "Failed", err.Error())
                    return ctrl.Result{}, err
                }
            } else {
                if err := prov.UpgradeComponent(ctx, nodeConfig, componentVersion, desired); err != nil {
                    r.updateComponentStatus(ctx, nodeConfig, componentName, desired.Version, "Failed", err.Error())
                    return ctrl.Result{}, err
                }
            }
            
            // 更新状态
            r.updateComponentStatus(ctx, nodeConfig, componentName, desired.Version, "Installing", "")
            return ctrl.Result{Requeue: true}, nil
        }
        
        // 检查组件健康状态
        if err := prov.HealthCheck(ctx, nodeConfig, componentName); err != nil {
            r.updateComponentStatus(ctx, nodeConfig, componentName, current.InstalledVersion, "Failed", err.Error())
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
    }
    
    return ctrl.Result{}, nil
}

func (r *NodeConfigReconciler) detectComponentStatus(ctx context.Context,
    nodeConfig *nodecomponentv1alpha1.NodeConfig, prov provider.Provider) error {
    
    for componentName := range nodeConfig.Spec.Components {
        status, err := prov.DetectComponentStatus(ctx, nodeConfig, componentName)
        if err != nil {
            return err
        }
        
        if nodeConfig.Status.ComponentStatus == nil {
            nodeConfig.Status.ComponentStatus = make(map[string]nodecomponentv1alpha1.ComponentStatusDetail)
        }
        
        nodeConfig.Status.ComponentStatus[componentName] = nodecomponentv1alpha1.ComponentStatusDetail{
            InstalledVersion: status.InstalledVersion,
            Phase:            string(status.Phase),
            Message:          status.Message,
            LastUpdated:      status.LastUpdated,
        }
    }
    
    return r.Status().Update(ctx, nodeConfig)
}
```
## 五、使用示例
### 5.1 Remote模式（用户完全提供OS）
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-192.168.1.10
  namespace: node-system
spec:
  deploymentMode:
    type: Remote
    remote:
      connection:
        host: 192.168.1.10
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
  
  components:
    containerd:
      version: v1.7.2
    kubelet:
      version: v1.28.2
    cni:
      version: v1.3.0
    csi:
      version: v2.1.0
```
### 5.2 Image模式（组件打包在OS镜像中）
```yaml
# 1. 定义OS镜像
apiVersion: nodecomponent.io/v1alpha1
kind: OSImage
metadata:
  name: ubuntu-22.04-k8s-v1.28
spec:
  image:
    name: ubuntu-22.04-k8s-v1.28
    version: v1.0.0
    url: https://mirror.example.com/images/ubuntu-22.04-k8s-v1.28.qcow2
  
  builtinComponents:
    - name: containerd
      version: v1.7.2
      path: /opt/components/containerd
      systemdUnit: containerd.service
    
    - name: kubelet
      version: v1.28.2
      path: /opt/components/kubelet
      systemdUnit: kubelet.service
    
    - name: cni
      version: v1.3.0
      path: /opt/cni/bin
    
    - name: csi
      version: v2.1.0
      path: /opt/components/csi
      systemdUnit: csi-driver.service
  
  upgradeSupport:
    enabled: true
    methods:
      - InPlace
      - RollingUpdate
---
# 2. 创建节点配置
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-image-001
  namespace: node-system
spec:
  deploymentMode:
    type: Image
    image:
      osImageRef:
        name: ubuntu-22.04-k8s-v1.28
        version: v1.0.0
      componentActivationMode: Auto
      upgradeStrategy: InPlace
  
  components:
    containerd:
      version: v1.7.2
    kubelet:
      version: v1.28.2
    cni:
      version: v1.3.0
    csi:
      version: v2.1.0
```
### 5.3 Hybrid模式（混合模式）
```yaml
apiVersion: nodecomponent.io/v1alpha1
kind: NodeConfig
metadata:
  name: node-hybrid-001
  namespace: node-system
spec:
  deploymentMode:
    type: Hybrid
    hybrid:
      connection:
        host: 192.168.1.20
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      
      # 内置组件：使用Image Provider
      builtinComponents:
        - containerd
        - cni
      
      # 远程组件：使用Remote Provider
      remoteComponents:
        - kubelet
        - csi
  
  components:
    containerd:
      version: v1.7.2  # 使用Image Provider
    kubelet:
      version: v1.28.2  # 使用Remote Provider
    cni:
      version: v1.3.0  # 使用Image Provider
    csi:
      version: v2.1.0  # 使用Remote Provider
```
## 六、方案对比
| 特性 | Remote模式 | Image模式 | Hybrid模式 |
|------|-----------|-----------|-----------|
| **适用场景** | 用户完全提供OS | 组件预装在OS镜像中 | 部分组件预装，部分远程安装 |
| **安装方式** | SSH远程安装 | 激活内置组件 | 根据组件类型选择 |
| **升级方式** | 远程升级 | 原地升级/滚动升级 | 混合升级 |
| **灵活性** | 高 | 中 | 高 |
| **部署速度** | 慢 | 快 | 中 |
| **OS依赖** | 无 | 需要专用镜像 | 部分依赖 |
| **适用环境** | 通用场景 | 生产环境标准化部署 | 定制化场景 |
## 七、总结
本方案通过**Provider抽象架构**，实现了对多种部署模式的统一支持：
1. **Remote模式**：适用于用户完全提供操作系统的场景，通过SSH远程安装和管理组件
2. **Image模式**：适用于组件预装在OS镜像中的场景，支持激活和升级内置组件
3. **Hybrid模式**：结合两种模式的优势，支持部分组件使用内置版本，部分组件远程安装

核心优势：
- **统一接口**：通过Provider接口抽象，控制器代码无需关心具体实现
- **灵活扩展**：可以轻松添加新的Provider实现，支持更多部署模式
- **平滑迁移**：支持从一种模式迁移到另一种模式，无需重新部署
- **版本管理**：所有模式共享统一的版本管理和兼容性检查机制

