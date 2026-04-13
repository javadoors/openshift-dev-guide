    
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

          
# 基于引导集群的节点组件管理完整方案
## 一、问题分析
### 1.1 核心矛盾
```
┌─────────────────────────────────────────────────────────────┐
│                    鸡生蛋问题                                │
│                                                             │
│  需要控制器管理节点组件──┐                                   │
│                          │                                   │
│                          ▼                                   │
│              需要运行在Kubernetes集群中                       │
│                          │                                   │
│                          ▼                                   │
│              需要先创建Kubernetes集群                         │
│                          │                                   │
│                          ▼                                   │
│              需要先安装节点组件              │
│                          │                                   │
│                          └── 回到起点                         │
└─────────────────────────────────────────────────────────────┘
```
### 1.2 解决思路
引入**引导集群**作为整个流程的起点，分阶段完成集群创建：
```
引导集群  ──创建──>  管理集群  ──管理──>  业务集群
     │                   │                       │
     │                   │                       │
  临时集群            持久化管理集群          业务工作负载集群
  生命周期短          生命周期长              生命周期长
  资源少              资源适中                资源按需
```
## 二、整体架构设计
### 2.1 三层架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                         整体架构                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: Bootstrap Cluster (引导集群)                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - 临时Kubernetes集群（k3s/kind/minikube）                    │  │
│  │  - 运行NodeComponentManager控制器                             │  │
│  │  - 执行初始节点组件安装                                       │  │
│  │  - 创建管理集群                                               │  │
│  │  - 迁移控制器到管理集群                                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 创建并迁移
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: Management Cluster (管理集群)                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - 持久化Kubernetes集群                                       │  │
│  │  - 运行NodeComponentManager控制器                             │  │
│  │  - 管理多个业务集群的节点组件                                 │  │
│  │  - 存储组件版本、配置、策略等CRD资源                          │  │
│  │  - 提供API供上层平台调用                                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 管理和创建
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 3: Workload Clusters (业务集群)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Cluster A    │  │ Cluster B    │  │ Cluster C    │             │
│  │ - 业务应用   │  │ - 业务应用   │  │ - 业务应用   │             │
│  │ - 节点组件   │  │ - 节点组件   │  │ - 节点组件   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└───────────────────────────────────────────────────────────────────┘
```
### 2.2 组件职责划分
| 组件 | 引导集群 | 管理集群 | 业务集群 |
|------|---------|---------|---------|
| **NodeComponentManager控制器** | ✅ 临时运行 | ✅ 持久运行 | ❌ 不运行 |
| **CRD资源** | ✅ 临时存储 | ✅ 持久存储 | ❌ 不存储 |
| **节点组件** | ❌ 不安装 | ✅ 安装管理组件 | ✅ 安装业务组件 |
| **生命周期** | 短（创建后销毁） | 长（持久化） | 长（业务驱动） |
## 三、详细设计
### 3.1 引导集群设计
#### 3.1.1 引导集群类型
```go
package bootstrap

type BootstrapClusterType string

const (
    BootstrapClusterTypeK3s    BootstrapClusterType = "k3s"
    BootstrapClusterTypeKind   BootstrapClusterType = "kind"
    BootstrapClusterTypeMinikube BootstrapClusterType = "minikube"
    BootstrapClusterTypeExisting BootstrapClusterType = "existing"  // 使用现有集群
)

type BootstrapClusterConfig struct {
    Type BootstrapClusterType
    
    // K3s配置
    K3s *K3sConfig
    
    // Kind配置
    Kind *KindConfig
    
    // Minikube配置
    Minikube *MinikubeConfig
    
    // 现有集群配置
    Existing *ExistingClusterConfig
}

type K3sConfig struct {
    Version string
    DataDir string
    TLSSan  []string
    ExtraArgs []string
}

type KindConfig struct {
    ClusterName string
    NodeImage   string
    ConfigPath  string
}

type MinikubeConfig struct {
    ClusterName string
    Driver      string
    Memory      string
    CPUs        int
}

type ExistingClusterConfig struct {
    KubeconfigPath string
    Context        string
}
```
#### 3.1.2 引导集群管理器
```go
package bootstrap

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "time"
    
    "k8s.io/client-go/tools/clientcmd"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

type BootstrapClusterManager struct {
    config *BootstrapClusterConfig
    client client.Client
    kubeconfig string
}

func NewBootstrapClusterManager(config *BootstrapClusterConfig) *BootstrapClusterManager {
    return &BootstrapClusterManager{
        config: config,
    }
}

// 创建引导集群
func (m *BootstrapClusterManager) Create(ctx context.Context) error {
    switch m.config.Type {
    case BootstrapClusterTypeK3s:
        return m.createK3sCluster(ctx)
    case BootstrapClusterTypeKind:
        return m.createKindCluster(ctx)
    case BootstrapClusterTypeMinikube:
        return m.createMinikubeCluster(ctx)
    case BootstrapClusterTypeExisting:
        return m.useExistingCluster(ctx)
    default:
        return fmt.Errorf("unsupported bootstrap cluster type: %s", m.config.Type)
    }
}

// 创建K3s引导集群
func (m *BootstrapClusterManager) createK3sCluster(ctx context.Context) error {
    k3sConfig := m.config.K3s
    
    // 下载k3s（如果不存在）
    if _, err := exec.LookPath("k3s"); err != nil {
        if err := m.downloadK3s(ctx); err != nil {
            return fmt.Errorf("download k3s failed: %v", err)
        }
    }
    
    // 构建启动参数
    args := []string{"server"}
    args = append(args, "--write-kubeconfig-mode", "0644")
    args = append(args, "--data-dir", k3sConfig.DataDir)
    
    for _, san := range k3sConfig.TLSSan {
        args = append(args, "--tls-san", san)
    }
    
    args = append(args, k3sConfig.ExtraArgs...)
    
    // 启动k3s
    cmd := exec.CommandContext(ctx, "k3s", args...)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Start(); err != nil {
        return fmt.Errorf("start k3s failed: %v", err)
    }
    
    // 等待k3s就绪
    m.kubeconfig = fmt.Sprintf("%s/server/cred/admin.kubeconfig", k3sConfig.DataDir)
    if err := m.waitForClusterReady(ctx, m.kubeconfig, 5*time.Minute); err != nil {
        return fmt.Errorf("wait for k3s ready failed: %v", err)
    }
    
    return nil
}

// 创建Kind引导集群
func (m *BootstrapClusterManager) createKindCluster(ctx context.Context) error {
    kindConfig := m.config.Kind
    
    // 检查kind是否安装
    if _, err := exec.LookPath("kind"); err != nil {
        return fmt.Errorf("kind not found, please install kind first")
    }
    
    // 创建kind集群
    args := []string{"create", "cluster", "--name", kindConfig.ClusterName}
    if kindConfig.NodeImage != "" {
        args = append(args, "--image", kindConfig.NodeImage)
    }
    if kindConfig.ConfigPath != "" {
        args = append(args, "--config", kindConfig.ConfigPath)
    }
    
    cmd := exec.CommandContext(ctx, "kind", args...)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("create kind cluster failed: %v", err)
    }
    
    // 获取kubeconfig
    m.kubeconfig = fmt.Sprintf("%s/.kube/config", os.Getenv("HOME"))
    
    // 等待集群就绪
    if err := m.waitForClusterReady(ctx, m.kubeconfig, 5*time.Minute); err != nil {
        return fmt.Errorf("wait for kind cluster ready failed: %v", err)
    }
    
    return nil
}

// 使用现有集群
func (m *BootstrapClusterManager) useExistingCluster(ctx context.Context) error {
    existingConfig := m.config.Existing
    
    m.kubeconfig = existingConfig.KubeconfigPath
    
    // 验证集群连接
    if err := m.waitForClusterReady(ctx, m.kubeconfig, 30*time.Second); err != nil {
        return fmt.Errorf("connect to existing cluster failed: %v", err)
    }
    
    return nil
}

// 等待集群就绪
func (m *BootstrapClusterManager) waitForClusterReady(ctx context.Context, kubeconfig string, timeout time.Duration) error {
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return err
    }
    
    k8sClient, err := client.New(config, client.Options{})
    if err != nil {
        return err
    }
    
    // 检查API Server健康状态
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        nodes := &corev1.NodeList{}
        if err := k8sClient.List(ctx, nodes); err == nil {
            // 检查节点是否就绪
            for _, node := range nodes.Items {
                for _, condition := range node.Status.Conditions {
                    if condition.Type == corev1.NodeReady && condition.Status == corev1.ConditionTrue {
                        return nil
                    }
                }
            }
        }
        
        time.Sleep(5 * time.Second)
    }
    
    return fmt.Errorf("cluster not ready within %v", timeout)
}

// 销毁引导集群
func (m *BootstrapClusterManager) Destroy(ctx context.Context) error {
    switch m.config.Type {
    case BootstrapClusterTypeK3s:
        return m.destroyK3sCluster(ctx)
    case BootstrapClusterTypeKind:
        return m.destroyKindCluster(ctx)
    case BootstrapClusterTypeMinikube:
        return m.destroyMinikubeCluster(ctx)
    case BootstrapClusterTypeExisting:
        // 现有集群不销毁
        return nil
    default:
        return fmt.Errorf("unsupported bootstrap cluster type: %s", m.config.Type)
    }
}

// 销毁K3s集群
func (m *BootstrapClusterManager) destroyK3sCluster(ctx context.Context) error {
    // 停止k3s服务
    cmd := exec.CommandContext(ctx, "k3s-killall.sh")
    cmd.Run()
    
    // 清理数据目录
    if m.config.K3s.DataDir != "" {
        os.RemoveAll(m.config.K3s.DataDir)
    }
    
    return nil
}

// 销毁Kind集群
func (m *BootstrapClusterManager) destroyKindCluster(ctx context.Context) error {
    cmd := exec.CommandContext(ctx, "kind", "delete", "cluster", "--name", m.config.Kind.ClusterName)
    return cmd.Run()
}

// 获取kubeconfig
func (m *BootstrapClusterManager) GetKubeconfig() string {
    return m.kubeconfig
}

// 获取客户端
func (m *BootstrapClusterManager) GetClient() (client.Client, error) {
    if m.client != nil {
        return m.client, nil
    }
    
    config, err := clientcmd.BuildConfigFromFlags("", m.kubeconfig)
    if err != nil {
        return nil, err
    }
    
    m.client, err = client.New(config, client.Options{})
    return m.client, err
}
```
### 3.2 引导流程设计
#### 3.2.1 引导流程控制器
```go
package bootstrap

import (
    "context"
    "fmt"
    "time"
    
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    bootstrapv1alpha1 "nodecomponent.io/api/bootstrap/v1alpha1"
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type BootstrapFlowReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    
    bootstrapManager *BootstrapClusterManager
    providerFactory  provider.ProviderFactory
}

// 引导流程CRD
// +kubebuilder:object:root=true
type BootstrapFlow struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   BootstrapFlowSpec   `json:"spec,omitempty"`
    Status BootstrapFlowStatus `json:"status,omitempty"`
}

type BootstrapFlowSpec struct {
    // 引导集群配置
    BootstrapCluster BootstrapClusterConfig `json:"bootstrapCluster"`
    
    // 管理集群配置
    ManagementCluster ManagementClusterConfig `json:"managementCluster"`
    
    // 初始节点列表
    InitialNodes []NodeBootstrapConfig `json:"initialNodes"`
    
    // 是否保留引导集群
    KeepBootstrapCluster bool `json:"keepBootstrapCluster,omitempty"`
}

type ManagementClusterConfig struct {
    // 管理集群节点配置
    Nodes []NodeConfig `json:"nodes"`
    
    // 管理集群版本
    KubernetesVersion string `json:"kubernetesVersion"`
    
    // 管理组件配置
    Components map[string]ComponentConfig `json:"components"`
}

type NodeBootstrapConfig struct {
    // 节点连接信息
    Connection ConnectionConfig `json:"connection"`
    
    // 节点角色
    Role string `json:"role"` // management, workload
    
    // 组件配置
    Components map[string]ComponentConfig `json:"components"`
    
    // 部署模式
    DeploymentMode DeploymentModeConfig `json:"deploymentMode"`
}

type BootstrapFlowStatus struct {
    // 当前阶段
    Phase BootstrapPhase `json:"phase"`
    
    // 引导集群状态
    BootstrapClusterStatus BootstrapClusterStatus `json:"bootstrapClusterStatus,omitempty"`
    
    // 管理集群状态
    ManagementClusterStatus ManagementClusterStatus `json:"managementClusterStatus,omitempty"`
    
    // 节点状态
    NodeStatus map[string]NodeStatus `json:"nodeStatus,omitempty"`
    
    // 错误信息
    ErrorMessage string `json:"errorMessage,omitempty"`
}

type BootstrapPhase string

const (
    BootstrapPhasePending          BootstrapPhase = "Pending"
    BootstrapPhaseCreatingBootstrap BootstrapPhase = "CreatingBootstrap"
    BootstrapPhaseInstallingCRDs    BootstrapPhase = "InstallingCRDs"
    BootstrapPhaseInstallingComponents BootstrapPhase = "InstallingComponents"
    BootstrapPhaseCreatingManagement BootstrapPhase = "CreatingManagement"
    BootstrapPhaseMigrating         BootstrapPhase = "Migrating"
    BootstrapPhaseCompleted         BootstrapPhase = "Completed"
    BootstrapPhaseFailed            BootstrapPhase = "Failed"
)

func (r *BootstrapFlowReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    flow := &bootstrapv1alpha1.BootstrapFlow{}
    if err := r.Get(ctx, req.NamespacedName, flow); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 根据当前阶段执行相应操作
    switch flow.Status.Phase {
    case "", BootstrapPhasePending:
        return r.createBootstrapCluster(ctx, flow)
    
    case BootstrapPhaseCreatingBootstrap:
        return r.installCRDs(ctx, flow)
    
    case BootstrapPhaseInstallingCRDs:
        return r.installComponents(ctx, flow)
    
    case BootstrapPhaseInstallingComponents:
        return r.createManagementCluster(ctx, flow)
    
    case BootstrapPhaseCreatingManagement:
        return r.migrateToManagement(ctx, flow)
    
    case BootstrapPhaseMigrating:
        return r.cleanup(ctx, flow)
    
    case BootstrapPhaseCompleted:
        return ctrl.Result{}, nil
    
    case BootstrapPhaseFailed:
        return ctrl.Result{}, nil
    
    default:
        return ctrl.Result{}, fmt.Errorf("unknown phase: %s", flow.Status.Phase)
    }
}

// 创建引导集群
func (r *BootstrapFlowReconciler) createBootstrapCluster(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Creating bootstrap cluster")
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseCreatingBootstrap
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    // 创建引导集群
    r.bootstrapManager = NewBootstrapClusterManager(&flow.Spec.BootstrapCluster)
    if err := r.bootstrapManager.Create(ctx); err != nil {
        flow.Status.Phase = BootstrapPhaseFailed
        flow.Status.ErrorMessage = fmt.Sprintf("Create bootstrap cluster failed: %v", err)
        r.Status().Update(ctx, flow)
        return ctrl.Result{}, err
    }
    
    // 更新引导集群状态
    flow.Status.BootstrapClusterStatus = BootstrapClusterStatus{
        Phase:       "Ready",
        Kubeconfig:  r.bootstrapManager.GetKubeconfig(),
        CreatedTime: metav1.Now(),
    }
    
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 安装CRDs
func (r *BootstrapFlowReconciler) installCRDs(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Installing CRDs to bootstrap cluster")
    
    // 获取引导集群客户端
    k8sClient, err := r.bootstrapManager.GetClient()
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 安装CRDs
    crds := []string{
        "nodeconfigs.nodecomponent.io",
        "componentversions.nodecomponent.io",
        "osimages.nodecomponent.io",
        "upgradepolicies.nodecomponent.io",
    }
    
    for _, crdName := range crds {
        if err := r.installCRD(ctx, k8sClient, crdName); err != nil {
            flow.Status.Phase = BootstrapPhaseFailed
            flow.Status.ErrorMessage = fmt.Sprintf("Install CRD %s failed: %v", crdName, err)
            r.Status().Update(ctx, flow)
            return ctrl.Result{}, err
        }
    }
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseInstallingCRDs
    flow.Status.BootstrapClusterStatus.CRDsInstalled = true
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 安装节点组件
func (r *BootstrapFlowReconciler) installComponents(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Installing components to initial nodes")
    
    // 获取引导集群客户端
    k8sClient, err := r.bootstrapManager.GetClient()
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 为每个初始节点创建NodeConfig
    for _, nodeConfig := range flow.Spec.InitialNodes {
        nodeConfigCRD := &nodecomponentv1alpha1.NodeConfig{
            ObjectMeta: metav1.ObjectMeta{
                Name: fmt.Sprintf("node-%s", nodeConfig.Connection.Host),
            },
            Spec: nodecomponentv1alpha1.NodeConfigSpec{
                DeploymentMode: nodeConfig.DeploymentMode,
                Components:     nodeConfig.Components,
            },
        }
        
        if err := k8sClient.Create(ctx, nodeConfigCRD); err != nil {
            flow.Status.Phase = BootstrapPhaseFailed
            flow.Status.ErrorMessage = fmt.Sprintf("Create NodeConfig for %s failed: %v", nodeConfig.Connection.Host, err)
            r.Status().Update(ctx, flow)
            return ctrl.Result{}, err
        }
    }
    
    // 等待所有节点组件安装完成
    if result, err := r.waitForNodesReady(ctx, k8sClient, flow); err != nil || result.Requeue {
        return result, err
    }
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseInstallingComponents
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 创建管理集群
func (r *BootstrapFlowReconciler) createManagementCluster(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Creating management cluster")
    
    // 使用kubeadm创建管理集群
    // 这里的节点已经通过NodeComponentManager安装了容器运行时和kubelet
    // 现在需要初始化管理集群
    
    // 1. 在第一个管理节点上执行kubeadm init
    managementNodes := flow.Spec.ManagementCluster.Nodes
    if len(managementNodes) == 0 {
        flow.Status.Phase = BootstrapPhaseFailed
        flow.Status.ErrorMessage = "No management nodes specified"
        r.Status().Update(ctx, flow)
        return ctrl.Result{}, fmt.Errorf("no management nodes specified")
    }
    
    firstNode := managementNodes[0]
    
    // 通过SSH执行kubeadm init
    if err := r.initManagementCluster(ctx, firstNode, flow.Spec.ManagementCluster); err != nil {
        flow.Status.Phase = BootstrapPhaseFailed
        flow.Status.ErrorMessage = fmt.Sprintf("Init management cluster failed: %v", err)
        r.Status().Update(ctx, flow)
        return ctrl.Result{}, err
    }
    
    // 2. 其他管理节点加入集群
    for i := 1; i < len(managementNodes); i++ {
        if err := r.joinManagementCluster(ctx, managementNodes[i], flow.Spec.ManagementCluster); err != nil {
            flow.Status.Phase = BootstrapPhaseFailed
            flow.Status.ErrorMessage = fmt.Sprintf("Join management cluster failed: %v", err)
            r.Status().Update(ctx, flow)
            return ctrl.Result{}, err
        }
    }
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseCreatingManagement
    flow.Status.ManagementClusterStatus = ManagementClusterStatus{
        Phase:       "Ready",
        CreatedTime: metav1.Now(),
    }
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 迁移到管理集群
func (r *BootstrapFlowReconciler) migrateToManagement(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Migrating to management cluster")
    
    // 1. 获取管理集群kubeconfig
    managementKubeconfig, err := r.getManagementKubeconfig(ctx, flow)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 2. 在管理集群上安装CRDs
    if err := r.installCRDsToManagement(ctx, managementKubeconfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. 部署NodeComponentManager控制器到管理集群
    if err := r.deployControllerToManagement(ctx, managementKubeconfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 迁移NodeConfig资源到管理集群
    if err := r.migrateNodeConfigs(ctx, managementKubeconfig); err != nil {
        return ctrl.Result{}, err
    }
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseMigrating
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 清理引导集群
func (r *BootstrapFlowReconciler) cleanup(ctx context.Context, flow *bootstrapv1alpha1.BootstrapFlow) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 如果不保留引导集群，则销毁
    if !flow.Spec.KeepBootstrapCluster {
        log.Info("Destroying bootstrap cluster")
        if err := r.bootstrapManager.Destroy(ctx); err != nil {
            log.Error(err, "Destroy bootstrap cluster failed")
        }
    }
    
    // 更新状态
    flow.Status.Phase = BootstrapPhaseCompleted
    if err := r.Status().Update(ctx, flow); err != nil {
        return ctrl.Result{}, err
    }
    
    log.Info("Bootstrap flow completed successfully")
    return ctrl.Result{}, nil
}
```
### 3.3 命令行工具设计
#### 3.3.1 CLI命令结构
```go
package cmd

import (
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "nodecomponent",
    Short: "Node Component Manager CLI",
    Long:  "Node Component Manager - Manage container runtime, kubelet, CNI, CSI on Kubernetes nodes",
}

var initCmd = &cobra.Command{
    Use:   "init",
    Short: "Initialize management cluster",
    Long:  "Initialize management cluster with bootstrap process",
    RunE:  runInit,
}

var addNodeCmd = &cobra.Command{
    Use:   "add-node",
    Short: "Add a new node to management or workload cluster",
    Long:  "Add a new node and install components",
    RunE:  runAddNode,
}

var upgradeCmd = &cobra.Command{
    Use:   "upgrade",
    Short: "Upgrade components on nodes",
    Long:  "Upgrade container runtime, kubelet, CNI, CSI on nodes",
    RunE:  runUpgrade,
}

var destroyCmd = &cobra.Command{
    Use:   "destroy",
    Short: "Destroy management cluster",
    Long:  "Destroy management cluster and cleanup resources",
    RunE:  runDestroy,
}

func init() {
    rootCmd.AddCommand(initCmd)
    rootCmd.AddCommand(addNodeCmd)
    rootCmd.AddCommand(upgradeCmd)
    rootCmd.AddCommand(destroyCmd)
    
    // init命令参数
    initCmd.Flags().String("bootstrap-type", "k3s", "Bootstrap cluster type (k3s, kind, minikube, existing)")
    initCmd.Flags().String("bootstrap-kubeconfig", "", "Kubeconfig for existing bootstrap cluster")
    initCmd.Flags().String("config", "", "Configuration file for initialization")
    initCmd.Flags().Bool("keep-bootstrap", false, "Keep bootstrap cluster after initialization")
    
    // add-node命令参数
    addNodeCmd.Flags().String("cluster", "", "Target cluster name")
    addNodeCmd.Flags().String("role", "worker", "Node role (master, worker)")
    addNodeCmd.Flags().String("host", "", "Node host IP")
    addNodeCmd.Flags().String("ssh-key", "", "SSH private key file")
    addNodeCmd.Flags().String("deployment-mode", "remote", "Deployment mode (remote, image, hybrid)")
    
    // upgrade命令参数
    upgradeCmd.Flags().String("cluster", "", "Target cluster name")
    upgradeCmd.Flags().String("component", "", "Component to upgrade (containerd, kubelet, cni, csi)")
    upgradeCmd.Flags().String("version", "", "Target version")
    upgradeCmd.Flags().String("policy", "rolling", "Upgrade policy (rolling, batch, inplace)")
}

func runInit(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    
    // 解析参数
    bootstrapType, _ := cmd.Flags().GetString("bootstrap-type")
    bootstrapKubeconfig, _ := cmd.Flags().GetString("bootstrap-kubeconfig")
    configFile, _ := cmd.Flags().GetString("config")
    keepBootstrap, _ := cmd.Flags().GetBool("keep-bootstrap")
    
    // 加载配置文件
    config, err := loadConfig(configFile)
    if err != nil {
        return fmt.Errorf("load config failed: %v", err)
    }
    
    // 设置引导集群配置
    config.BootstrapCluster.Type = bootstrap.ClusterType(bootstrapType)
    if bootstrapType == "existing" {
        config.BootstrapCluster.Existing = &bootstrap.ExistingClusterConfig{
            KubeconfigPath: bootstrapKubeconfig,
        }
    }
    config.KeepBootstrapCluster = keepBootstrap
    
    // 创建引导流程管理器
    flowManager := bootstrap.NewBootstrapFlowManager(config)
    
    // 执行引导流程
    if err := flowManager.Run(ctx); err != nil {
        return fmt.Errorf("bootstrap flow failed: %v", err)
    }
    
    fmt.Println("✅ Management cluster initialized successfully!")
    fmt.Printf("   Kubeconfig: %s\n", flowManager.GetManagementKubeconfig())
    
    return nil
}

func runAddNode(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    
    // 解析参数
    cluster, _ := cmd.Flags().GetString("cluster")
    role, _ := cmd.Flags().GetString("role")
    host, _ := cmd.Flags().GetString("host")
    sshKey, _ := cmd.Flags().GetString("ssh-key")
    deploymentMode, _ := cmd.Flags().GetString("deployment-mode")
    
    // 连接到管理集群
    kubeconfig := os.Getenv("KUBECONFIG")
    if kubeconfig == "" {
        kubeconfig = filepath.Join(os.Getenv("HOME"), ".kube", "config")
    }
    
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return fmt.Errorf("build config failed: %v", err)
    }
    
    k8sClient, err := client.New(config, client.Options{})
    if err != nil {
        return fmt.Errorf("create client failed: %v", err)
    }
    
    // 创建NodeConfig
    nodeConfig := &nodecomponentv1alpha1.NodeConfig{
        ObjectMeta: metav1.ObjectMeta{
            Name: fmt.Sprintf("node-%s", host),
            Labels: map[string]string{
                "cluster": cluster,
                "role":    role,
            },
        },
        Spec: nodecomponentv1alpha1.NodeConfigSpec{
            DeploymentMode: nodecomponentv1alpha1.DeploymentModeConfig{
                Type: nodecomponentv1alpha1.DeploymentModeType(deploymentMode),
                Remote: &nodecomponentv1alpha1.RemoteConfig{
                    Connection: nodecomponentv1alpha1.ConnectionConfig{
                        Host: host,
                        Port: 22,
                        User: "root",
                        SSHKeySecret: nodecomponentv1alpha1.SecretReference{
                            Name:      "node-ssh-key",
                            Namespace: "node-system",
                        },
                    },
                },
            },
            Components: getDefaultComponents(role),
        },
    }
    
    // 创建NodeConfig
    if err := k8sClient.Create(ctx, nodeConfig); err != nil {
        return fmt.Errorf("create NodeConfig failed: %v", err)
    }
    
    fmt.Printf("✅ Node %s added successfully!\n", host)
    fmt.Println("   Check status with: nodecomponent get node node-" + host)
    
    return nil
}

func runUpgrade(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    
    // 解析参数
    cluster, _ := cmd.Flags().GetString("cluster")
    component, _ := cmd.Flags().GetString("component")
    version, _ := cmd.Flags().GetString("version")
    policy, _ := cmd.Flags().GetString("policy")
    
    // 连接到管理集群
    kubeconfig := os.Getenv("KUBECONFIG")
    if kubeconfig == "" {
        kubeconfig = filepath.Join(os.Getenv("HOME"), ".kube", "config")
    }
    
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return fmt.Errorf("build config failed: %v", err)
    }
    
    k8sClient, err := client.New(config, client.Options{})
    if err != nil {
        return fmt.Errorf("create client failed: %v", err)
    }
    
    // 获取集群的所有节点
    nodeConfigs := &nodecomponentv1alpha1.NodeConfigList{}
    if err := k8sClient.List(ctx, nodeConfigs, client.MatchingLabels{"cluster": cluster}); err != nil {
        return fmt.Errorf("list nodes failed: %v", err)
    }
    
    // 创建升级策略
    upgradePolicy := &nodecomponentv1alpha1.UpgradePolicy{
        ObjectMeta: metav1.ObjectMeta{
            Name: fmt.Sprintf("upgrade-%s-%s", component, time.Now().Format("20060102-150405")),
        },
        Spec: nodecomponentv1alpha1.UpgradePolicySpec{
            Type: nodecomponentv1alpha1.UpgradePolicyType(policy),
            TargetNodes: nodecomponentv1alpha1.NodeSelector{
                MatchLabels: map[string]string{"cluster": cluster},
            },
            TargetVersions: map[string]string{
                component: version,
            },
        },
    }
    
    if err := k8sClient.Create(ctx, upgradePolicy); err != nil {
        return fmt.Errorf("create upgrade policy failed: %v", err)
    }
    
    fmt.Printf("✅ Upgrade policy created successfully!\n")
    fmt.Printf("   Policy: %s\n", upgradePolicy.Name)
    fmt.Println("   Check status with: nodecomponent get upgrade-policy " + upgradePolicy.Name)
    
    return nil
}
```
## 四、完整流程示例
### 4.1 初始化管理集群
#### 4.1.1 创建配置文件
```yaml
# init-config.yaml
apiVersion: nodecomponent.io/v1alpha1
kind: BootstrapFlow
metadata:
  name: init-management-cluster
spec:
  # 引导集群配置
  bootstrapCluster:
    type: k3s
    k3s:
      version: v1.27.0
      dataDir: /var/lib/rancher/k3s-bootstrap
      tlsSan:
        - 192.168.1.100
  
  # 管理集群配置
  managementCluster:
    kubernetesVersion: v1.28.2
    nodes:
      - host: 192.168.1.101
        role: master
      - host: 192.168.1.102
        role: master
      - host: 192.168.1.103
        role: master
    components:
      containerd:
        version: v1.7.2
      kubelet:
        version: v1.28.2
      cni:
        version: v1.3.0
  
  # 初始节点配置
  initialNodes:
    - connection:
        host: 192.168.1.101
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      role: management
      deploymentMode:
        type: Remote
        remote:
          connection:
            host: 192.168.1.101
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
    
    - connection:
        host: 192.168.1.102
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      role: management
      deploymentMode:
        type: Remote
      components:
        containerd:
          version: v1.7.2
        kubelet:
          version: v1.28.2
        cni:
          version: v1.3.0
    
    - connection:
        host: 192.168.1.103
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      role: management
      deploymentMode:
        type: Remote
      components:
        containerd:
          version: v1.7.2
        kubelet:
          version: v1.28.2
        cni:
          version: v1.3.0
  
  # 不保留引导集群
  keepBootstrapCluster: false
```
#### 4.1.2 执行初始化
```bash
# 1. 创建SSH密钥Secret（如果还没有）
kubectl create secret generic node-ssh-key \
  --from-file=id_rsa=/path/to/ssh/key \
  -n node-system

# 2. 执行初始化
nodecomponent init --config init-config.yaml

# 输出：
# ℹ️  Creating bootstrap cluster (k3s)...
# ✅ Bootstrap cluster created successfully
# ℹ️  Installing CRDs to bootstrap cluster...
# ✅ CRDs installed successfully
# ℹ️  Installing components to initial nodes...
#    - Installing containerd on 192.168.1.101...
#    - Installing kubelet on 192.168.1.101...
#    - Installing cni on 192.168.1.101...
#    - Installing containerd on 192.168.1.102...
#    ...
# ✅ Components installed successfully
# ℹ️  Creating management cluster...
#    - Initializing cluster on 192.168.1.101...
#    - Joining 192.168.1.102 to cluster...
#    - Joining 192.168.1.103 to cluster...
# ✅ Management cluster created successfully
# ℹ️  Migrating to management cluster...
#    - Installing CRDs to management cluster...
#    - Deploying controllers to management cluster...
#    - Migrating NodeConfig resources...
# ✅ Migration completed successfully
# ℹ️  Destroying bootstrap cluster...
# ✅ Bootstrap cluster destroyed
# 
# ✅ Management cluster initialized successfully!
#    Kubeconfig: /root/.kube/config
```
### 4.2 添加工作节点
```bash
# 添加工作节点到管理集群
nodecomponent add-node \
  --cluster management \
  --role worker \
  --host 192.168.1.201 \
  --ssh-key /path/to/ssh/key \
  --deployment-mode remote

# 输出：
# ℹ️  Creating NodeConfig for 192.168.1.201...
# ✅ Node 192.168.1.201 added successfully!
#    Check status with: nodecomponent get node node-192.168.1.201
```
### 4.3 创建业务集群
```bash
# 创建业务集群配置
cat <<EOF | kubectl apply -f -
apiVersion: nodecomponent.io/v1alpha1
kind: Cluster
metadata:
  name: workload-cluster-1
spec:
  controlPlane:
    nodes:
      - host: 192.168.2.101
        role: master
      - host: 192.168.2.102
        role: master
      - host: 192.168.2.103
        role: master
  
  workers:
    - host: 192.168.2.201
      role: worker
    - host: 192.168.2.202
      role: worker
  
  components:
    containerd:
      version: v1.7.2
    kubelet:
      version: v1.28.2
    cni:
      version: v1.3.0
    csi:
      version: v2.1.0
EOF

# 输出：
# cluster.nodecomponent.io/workload-cluster-1 created
```
### 4.4 升级组件
```bash
# 升级kubelet到v1.29.0
nodecomponent upgrade \
  --cluster workload-cluster-1 \
  --component kubelet \
  --version v1.29.0 \
  --policy rolling

# 输出：
# ℹ️  Creating upgrade policy...
# ✅ Upgrade policy created successfully!
#    Policy: upgrade-kubelet-20240115-143022
#    Check status with: nodecomponent get upgrade-policy upgrade-kubelet-20240115-143022

# 查看升级状态
nodecomponent get upgrade-policy upgrade-kubelet-20240115-143022

# 输出：
# NAME                      COMPONENT   TARGET VERSION   PHASE       PROGRESS   AGE
# upgrade-kubelet-20240115-143022   kubelet     v1.29.0          Upgrading   2/5        5m
```
## 五、架构优势
### 5.1 解决鸡生蛋问题
```
┌─────────────────────────────────────────────────────────────┐
│                    传统方式的问题                            │
│                                                             │
│  需要控制器 ──> 需要集群 ──> 需要节点 ──> 需要组件 ──> 死循环 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    引导集群方案                              │
│                                                             │
│  引导集群（临时） ──> 安装组件 ──> 创建管理集群 ──> 迁移      │
│        ↑                                                    │
│        └── 不需要预先存在                                    │
└─────────────────────────────────────────────────────────────┘
```
### 5.2 灵活的部署模式
| 阶段 | Remote模式 | Image模式 | Hybrid模式 |
|------|-----------|-----------|-----------|
| **引导集群创建** | 使用k3s/kind | 使用预装镜像 | 混合使用 |
| **管理集群创建** | SSH安装组件 | 激活内置组件 | 按需选择 |
| **业务集群创建** | SSH安装组件 | 激活内置组件 | 按需选择 |
### 5.3 完整的生命周期管理
```
┌─────────────────────────────────────────────────────────────┐
│                    集群生命周期                              │
│                                                             │
│  初始化 ──> 扩容 ──> 升级 ──> 维护 ──> 缩容 ──> 销毁        │
│    │         │        │        │        │        │         │
│    │         │        │        │        │        │         │
│  引导流程  添加节点  组件升级  故障修复  移除节点  集群删除  │
└─────────────────────────────────────────────────────────────┘
```
## 六、总结
本方案通过**引导集群**解决了"鸡生蛋"问题，实现了完整的节点组件管理流程：
### 6.1 核心创新点
1. **三层架构**：引导集群 → 管理集群 → 业务集群，清晰分层
2. **Provider抽象**：统一接口支持Remote/Image/Hybrid三种部署模式
3. **声明式管理**：使用CRD定义期望状态，控制器自动协调
4. **自动化流程**：从引导到迁移的全自动化流程
### 6.2 关键流程
1. **初始化流程**：创建引导集群 → 安装组件 → 创建管理集群 → 迁移 → 清理
2. **扩容流程**：创建NodeConfig → 控制器安装组件 → 节点加入集群
3. **升级流程**：创建升级策略 → 控制器执行升级 → 健康检查 → 完成
### 6.3 适用场景
- **Remote模式**：用户完全提供操作系统，需要灵活安装组件
- **Image模式**：组件预装在OS镜像中，需要快速部署
- **Hybrid模式**：部分组件预装，部分组件远程安装

该方案提供了从零开始创建Kubernetes集群的完整解决方案，支持多种部署模式和灵活的组件管理。

   
# 基于Cluster API的业务集群管理方案扩展
## 一、方案概述
扩展现有方案，将Cluster API（CAPI）集成到管理集群中，实现通过CAPI标准化管理业务集群的创建、扩容、升级和销毁。
### 核心设计理念
1. **分层架构**：引导集群 → 管理集群(CAPI) → 业务集群
2. **CAPI集成**：在管理集群中部署CAPI核心组件和Infrastructure Provider
3. **统一接口**：通过自定义CRD封装CAPI资源，提供简化的用户体验
4. **组件联动**：节点组件管理与CAPI机器生命周期协调
## 二、架构设计
### 2.1 扩展后的整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                         整体架构                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: Bootstrap Cluster (引导集群)                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - 临时Kubernetes集群（k3s/kind/minikube）                    │  │
│  │  - 运行NodeComponentManager控制器                             │  │
│  │  - 创建管理集群                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 创建并迁移
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: Management Cluster (管理集群 + CAPI)                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - 持久化Kubernetes集群                                       │  │
│  │  - NodeComponentManager控制器                                 │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                     │  │
│  │  │ Cluster API     │  │ Infrastructure  │                     │  │
│  │  │ Core Components │  │ Providers       │                     │  │
│  │  │   - cluster-api │  │   - capbke      │                     │  │
│  │  │   - kubeadm     │  │   - capd        │                     │  │
│  │  │   - bootstrap   │  │   - ...         │                     │  │
│  │  └─────────────────┘  └─────────────────┘                     │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                     │  │
│  │  │ Custom CRDs     │  │ Integration     │                     │  │
│  │  │   - Cluster     │  │ Controllers     │                     │  │
│  │  │   - NodeConfig  │  │   - Cluster     │                     │  │
│  │  │   - ComponentVer│  │   - Node        │                     │  │
│  │  └─────────────────┘  └─────────────────┘                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 管理和创建
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 3: Workload Clusters (业务集群)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Cluster A    │  │ Cluster B    │  │ Cluster C    │             │
│  │ - CAPI管理   │  │ - CAPI管理   │  │ - CAPI管理   │             │
│  │ - 节点组件   │  │ - 节点组件   │  │ - 节点组件   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└───────────────────────────────────────────────────────────────────┘
```
### 2.2 集成关键点
| 组件 | 职责 | 集成方式 |
|------|------|---------|
| **Cluster API Core**        | 提供集群生命周期管理API | 在管理集群中部署       |
| **Infrastructure Provider** | 提供基础设施资源管理    | 部署CAPBKE等提供者     |
| **NodeComponentManager**    | 管理节点组件            | 与CAPI机器生命周期联动 |
| **Custom Cluster CRD**      | 统一集群创建接口        | 转换为CAPI资源         |
| **Integration Controller**  | 协调CAPI与节点组件管理  | 实现资源转换和状态同步 |
## 三、Cluster API集成设计
### 3.1 CAPI组件部署
#### 3.1.1 在管理集群中部署CAPI核心组件
```go
package capi

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
    "time"
    
    "sigs.k8s.io/cluster-api/cmd/clusterctl/client/config"
    "sigs.k8s.io/cluster-api/cmd/clusterctl/client/repository"
    "sigs.k8s.io/cluster-api/cmd/clusterctl/client/update"
)

type CAPIInstaller struct {
    kubeconfig string
    namespace  string
}

func NewCAPIInstaller(kubeconfig string, namespace string) *CAPIInstaller {
    return &CAPIInstaller{
        kubeconfig: kubeconfig,
        namespace:  namespace,
    }
}

// 部署CAPI核心组件
func (i *CAPIInstaller) DeployCoreComponents(ctx context.Context, version string) error {
    // 使用clusterctl命令部署CAPI核心组件
    cmd := exec.CommandContext(ctx, "clusterctl", "init", 
        "--kubeconfig", i.kubeconfig,
        "--namespace", i.namespace,
        "--core", fmt.Sprintf("cluster-api:%s", version),
        "--bootstrap", "kubeadm:v1.28.2",
        "--control-plane", "kubeadm:v1.28.2",
    )
    
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    return cmd.Run()
}

// 部署Infrastructure Provider
func (i *CAPIInstaller) DeployInfrastructureProvider(ctx context.Context, providerName string, version string) error {
    // 使用clusterctl命令部署基础设施提供者
    cmd := exec.CommandContext(ctx, "clusterctl", "init", 
        "--kubeconfig", i.kubeconfig,
        "--namespace", i.namespace,
        "--infrastructure", fmt.Sprintf("%s:%s", providerName, version),
    )
    
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    return cmd.Run()
}

// 等待CAPI组件就绪
func (i *CAPIInstaller) WaitForCAPIRunning(ctx context.Context) error {
    config, err := clientcmd.BuildConfigFromFlags("", i.kubeconfig)
    if err != nil {
        return err
    }
    
    client, err := kubernetes.NewForConfig(config)
    if err != nil {
        return err
    }
    
    // 等待cluster-api控制器就绪
    deadline := time.Now().Add(5 * time.Minute)
    for time.Now().Before(deadline) {
        deployments, err := client.AppsV1().Deployments(i.namespace).List(ctx, metav1.ListOptions{
            LabelSelector: "cluster.x-k8s.io/provider=cluster-api",
        })
        
        if err != nil {
            time.Sleep(10 * time.Second)
            continue
        }
        
        allReady := true
        for _, deployment := range deployments.Items {
            if deployment.Status.ReadyReplicas != *deployment.Spec.Replicas {
                allReady = false
                break
            }
        }
        
        if allReady {
            return nil
        }
        
        time.Sleep(10 * time.Second)
    }
    
    return fmt.Errorf("CAPI components not ready within 5 minutes")
}
```
### 3.2 自定义Cluster CRD设计
#### 3.2.1 Cluster CRD定义
```yaml
apiVersion: cluster.nodecomponent.io/v1alpha1
kind: Cluster
metadata:
  name: workload-cluster-1
spec:
  # 集群基本信息
  clusterName: workload-cluster-1
  kubernetesVersion: v1.28.2
  
  # 控制平面配置
  controlPlane:
    replicas: 3
    infrastructureRef:
      kind: BKECluster
      name: workload-cluster-1-control-plane
    machineTemplate:
      infrastructureRef:
        kind: BKEMachineTemplate
        name: workload-cluster-1-control-plane
      bootstrap:
        configRef:
          kind: KubeadmConfigTemplate
          name: workload-cluster-1-control-plane
  
  # 工作节点配置
  workers:
    replicas: 3
    machineDeployments:
      - name: worker-pool-1
        replicas: 2
        infrastructureRef:
          kind: BKEMachineTemplate
          name: workload-cluster-1-worker
        bootstrap:
          configRef:
            kind: KubeadmConfigTemplate
            name: workload-cluster-1-worker
  
  # 节点组件配置
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
  
  # 基础设施配置
  infrastructure:
    provider: capbke
    version: v1.0.0
    config:
      bkeClusterConfig:
        network:
          podCIDR: 10.244.0.0/16
          serviceCIDR: 10.96.0.0/12
        etcd:
          storage: 20Gi
        master:
          nodes:
            - ip: 192.168.2.101
              role: master
            - ip: 192.168.2.102
              role: master
            - ip: 192.168.2.103
              role: master
        worker:
          nodes:
            - ip: 192.168.2.201
              role: worker
            - ip: 192.168.2.202
              role: worker
            - ip: 192.168.2.203
              role: worker

status:
  phase: Provisioning  # Pending, Provisioning, Ready, Failed, Upgrading
  controlPlaneStatus:
    phase: Provisioning
    readyReplicas: 0
    replicas: 3
  workerStatus:
    phase: Provisioning
    readyReplicas: 0
    replicas: 3
  componentStatus:
    containerd:
      version: v1.7.2
      status: Provisioning
    kubelet:
      version: v1.28.2
      status: Provisioning
  kubeconfig: |
    apiVersion: v1
    clusters:
    - cluster:
        server: https://192.168.2.101:6443
        certificate-authority-data: xxxxx
      name: workload-cluster-1
    # ...
```
### 3.3 Cluster集成控制器
#### 3.3.1 控制器实现
```go
package controller

import (
    "context"
    "fmt"
    "time"
    
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/cluster-api/api/v1beta1"
    capierrors "sigs.k8s.io/cluster-api/errors"
    
    clusterv1alpha1 "cluster.nodecomponent.io/api/v1alpha1"
    "cluster.nodecomponent.io/controller/capi"
)

type ClusterReconciler struct {
    client.Client
    Scheme        *runtime.Scheme
    capiManager   *capi.CAPIManager
    componentManager *component.ComponentManager
}

func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 获取Cluster资源
    cluster := &clusterv1alpha1.Cluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 根据当前阶段执行操作
    switch cluster.Status.Phase {
    case "", clusterv1alpha1.ClusterPhasePending:
        return r.initializeCluster(ctx, cluster)
    
    case clusterv1alpha1.ClusterPhaseProvisioning:
        return r.provisionCluster(ctx, cluster)
    
    case clusterv1alpha1.ClusterPhaseReady:
        return r.manageComponents(ctx, cluster)
    
    case clusterv1alpha1.ClusterPhaseFailed:
        return r.handleFailure(ctx, cluster)
    
    case clusterv1alpha1.ClusterPhaseUpgrading:
        return r.upgradeCluster(ctx, cluster)
    
    default:
        return ctrl.Result{}, fmt.Errorf("unknown cluster phase: %s", cluster.Status.Phase)
    }
}

// 初始化集群
func (r *ClusterReconciler) initializeCluster(ctx context.Context, cluster *clusterv1alpha1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Initializing cluster")
    
    // 1. 验证配置
    if err := r.validateClusterConfig(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 2. 创建节点配置
    if err := r.createNodeConfigs(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. 创建CAPI资源
    if err := r.createCAPIResources(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. 更新状态
    cluster.Status.Phase = clusterv1alpha1.ClusterPhaseProvisioning
    cluster.Status.ControlPlaneStatus.Phase = clusterv1alpha1.ControlPlanePhaseProvisioning
    cluster.Status.WorkerStatus.Phase = clusterv1alpha1.WorkerPhaseProvisioning
    
    if err := r.Status().Update(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{Requeue: true}, nil
}

// 创建节点配置
func (r *ClusterReconciler) createNodeConfigs(ctx context.Context, cluster *clusterv1alpha1.Cluster) error {
    log := ctrl.LoggerFrom(ctx)
    
    // 创建控制平面节点配置
    for _, node := range cluster.Spec.Infrastructure.Config.BKEClusterConfig.Master.Nodes {
        nodeConfig := r.buildNodeConfig(cluster, node, "control-plane")
        if err := r.Create(ctx, nodeConfig); err != nil {
            return fmt.Errorf("create control-plane node config failed: %v", err)
        }
    }
    
    // 创建工作节点配置
    for _, node := range cluster.Spec.Infrastructure.Config.BKEClusterConfig.Worker.Nodes {
        nodeConfig := r.buildNodeConfig(cluster, node, "worker")
        if err := r.Create(ctx, nodeConfig); err != nil {
            return fmt.Errorf("create worker node config failed: %v", err)
        }
    }
    
    return nil
}

// 构建节点配置
func (r *ClusterReconciler) buildNodeConfig(cluster *clusterv1alpha1.Cluster, nodeInfo clusterv1alpha1.NodeInfo, role string) *nodecomponentv1alpha1.NodeConfig {
    return &nodecomponentv1alpha1.NodeConfig{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-%s-%s", cluster.Name, role, nodeInfo.IP),
            Namespace: cluster.Namespace,
            Labels: map[string]string{
                "cluster": cluster.Name,
                "role":    role,
                "node":    nodeInfo.IP,
            },
        },
        Spec: nodecomponentv1alpha1.NodeConfigSpec{
            DeploymentMode: nodecomponentv1alpha1.DeploymentModeConfig{
                Type: "Remote",
                Remote: &nodecomponentv1alpha1.RemoteConfig{
                    Connection: nodecomponentv1alpha1.ConnectionConfig{
                        Host: nodeInfo.IP,
                        Port: 22,
                        User: "root",
                        SSHKeySecret: nodecomponentv1alpha1.SecretReference{
                            Name:      "node-ssh-key",
                            Namespace: "node-system",
                        },
                    },
                },
            },
            Components: cluster.Spec.Components,
        },
    }
}

// 创建CAPI资源
func (r *ClusterReconciler) createCAPIResources(ctx context.Context, cluster *clusterv1alpha1.Cluster) error {
    // 1. 创建CAPI Cluster资源
    capiCluster := r.buildCAPICluster(cluster)
    if err := r.Create(ctx, capiCluster); err != nil {
        return fmt.Errorf("create CAPI Cluster failed: %v", err)
    }
    
    // 2. 创建基础设施Cluster资源 (BKECluster)
    infraCluster := r.buildInfrastructureCluster(cluster)
    if err := r.Create(ctx, infraCluster); err != nil {
        return fmt.Errorf("create infrastructure cluster failed: %v", err)
    }
    
    // 3. 创建控制平面资源
    if err := r.createControlPlaneResources(ctx, cluster); err != nil {
        return fmt.Errorf("create control plane resources failed: %v", err)
    }
    
    // 4. 创建工作节点资源
    if err := r.createWorkerResources(ctx, cluster); err != nil {
        return fmt.Errorf("create worker resources failed: %v", err)
    }
    
    return nil
}

// 构建CAPI Cluster资源
func (r *ClusterReconciler) buildCAPICluster(cluster *clusterv1alpha1.Cluster) *v1beta1.Cluster {
    capiCluster := &v1beta1.Cluster{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cluster.Name,
            Namespace: cluster.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(cluster, clusterv1alpha1.GroupVersion.WithKind("Cluster")),
            },
        },
        Spec: v1beta1.ClusterSpec{
            ClusterNetwork: &v1beta1.ClusterNetwork{
                Pods:    &v1beta1.NetworkRanges{CIDRBlocks: []string{cluster.Spec.Infrastructure.Config.BKEClusterConfig.Network.PodCIDR}},
                Services: &v1beta1.NetworkRanges{CIDRBlocks: []string{cluster.Spec.Infrastructure.Config.BKEClusterConfig.Network.ServiceCIDR}},
            },
            ControlPlaneRef: &corev1.ObjectReference{
                Kind:       "KubeadmControlPlane",
                Name:       fmt.Sprintf("%s-control-plane", cluster.Name),
                Namespace:  cluster.Namespace,
                APIVersion: "controlplane.cluster.x-k8s.io/v1beta1",
            },
            InfrastructureRef: &corev1.ObjectReference{
                Kind:       "BKECluster",
                Name:       fmt.Sprintf("%s-control-plane", cluster.Name),
                Namespace:  cluster.Namespace,
                APIVersion: "infrastructure.cluster.x-k8s.io/v1beta1",
            },
        },
    }
    
    return capiCluster
}

// 创建控制平面资源
func (r *ClusterReconciler) createControlPlaneResources(ctx context.Context, cluster *clusterv1alpha1.Cluster) error {
    // 1. 创建KubeadmConfigTemplate
    kubeadmConfigTemplate := r.buildKubeadmConfigTemplate(cluster, "control-plane")
    if err := r.Create(ctx, kubeadmConfigTemplate); err != nil {
        return fmt.Errorf("create kubeadm config template failed: %v", err)
    }
    
    // 2. 创建BKEMachineTemplate
    bkeMachineTemplate := r.buildBKEMachineTemplate(cluster, "control-plane")
    if err := r.Create(ctx, bkeMachineTemplate); err != nil {
        return fmt.Errorf("create bke machine template failed: %v", err)
    }
    
    // 3. 创建KubeadmControlPlane
    kubeadmControlPlane := r.buildKubeadmControlPlane(cluster)
    if err := r.Create(ctx, kubeadmControlPlane); err != nil {
        return fmt.Errorf("create kubeadm control plane failed: %v", err)
    }
    
    return nil
}

// 创建工作节点资源
func (r *ClusterReconciler) createWorkerResources(ctx context.Context, cluster *clusterv1alpha1.Cluster) error {
    // 1. 创建KubeadmConfigTemplate
    kubeadmConfigTemplate := r.buildKubeadmConfigTemplate(cluster, "worker")
    if err := r.Create(ctx, kubeadmConfigTemplate); err != nil {
        return fmt.Errorf("create kubeadm config template failed: %v", err)
    }
    
    // 2. 创建BKEMachineTemplate
    bkeMachineTemplate := r.buildBKEMachineTemplate(cluster, "worker")
    if err := r.Create(ctx, bkeMachineTemplate); err != nil {
        return fmt.Errorf("create bke machine template failed: %v", err)
    }
    
    // 3. 创建MachineDeployment
    machineDeployment := r.buildMachineDeployment(cluster)
    if err := r.Create(ctx, machineDeployment); err != nil {
        return fmt.Errorf("create machine deployment failed: %v", err)
    }
    
    return nil
}

// 提供集群
func (r *ClusterReconciler) provisionCluster(ctx context.Context, cluster *clusterv1alpha1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 1. 检查CAPI Cluster状态
    capiCluster := &v1beta1.Cluster{}
    if err := r.Get(ctx, client.ObjectKey{Namespace: cluster.Namespace, Name: cluster.Name}, capiCluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 2. 检查控制平面状态
    kubeadmControlPlane := &controlplanev1beta1.KubeadmControlPlane{}
    if err := r.Get(ctx, client.ObjectKey{Namespace: cluster.Namespace, Name: fmt.Sprintf("%s-control-plane", cluster.Name)}, kubeadmControlPlane); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. 更新控制平面状态
    cluster.Status.ControlPlaneStatus.ReadyReplicas = kubeadmControlPlane.Status.ReadyReplicas
    cluster.Status.ControlPlaneStatus.Replicas = kubeadmControlPlane.Status.Replicas
    
    if kubeadmControlPlane.Status.ReadyReplicas == kubeadmControlPlane.Status.Replicas {
        cluster.Status.ControlPlaneStatus.Phase = clusterv1alpha1.ControlPlanePhaseReady
    }
    
    // 4. 检查工作节点状态
    machineDeployment := &v1beta1.MachineDeployment{}
    if err := r.Get(ctx, client.ObjectKey{Namespace: cluster.Namespace, Name: fmt.Sprintf("%s-worker", cluster.Name)}, machineDeployment); err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. 更新工作节点状态
    cluster.Status.WorkerStatus.ReadyReplicas = machineDeployment.Status.ReadyReplicas
    cluster.Status.WorkerStatus.Replicas = machineDeployment.Status.Replicas
    
    if machineDeployment.Status.ReadyReplicas == machineDeployment.Status.Replicas {
        cluster.Status.WorkerStatus.Phase = clusterv1alpha1.WorkerPhaseReady
    }
    
    // 6. 如果所有组件都就绪，更新集群状态
    if cluster.Status.ControlPlaneStatus.Phase == clusterv1alpha1.ControlPlanePhaseReady &&
       cluster.Status.WorkerStatus.Phase == clusterv1alpha1.WorkerPhaseReady {
        
        cluster.Status.Phase = clusterv1alpha1.ClusterPhaseReady
        
        // 获取集群kubeconfig
        kubeconfigSecret := &corev1.Secret{}
        if err := r.Get(ctx, client.ObjectKey{Namespace: cluster.Namespace, Name: fmt.Sprintf("%s-kubeconfig", cluster.Name)}, kubeconfigSecret); err == nil {
            cluster.Status.Kubeconfig = string(kubeconfigSecret.Data["value"])
        }
    }
    
    if err := r.Status().Update(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }
    
    if cluster.Status.Phase != clusterv1alpha1.ClusterPhaseReady {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    return ctrl.Result{Requeue: true}, nil
}
```
### 3.4 Cluster API与节点组件管理联动
#### 3.4.1 节点组件安装与CAPI机器生命周期协调
```go
package controller

import (
    "context"
    "fmt"
    
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/cluster-api/api/v1beta1"
    
    nodecomponentv1alpha1 "nodecomponent.io/api/v1alpha1"
)

type MachineReconciler struct {
    client.Client
    Scheme        *runtime.Scheme
    componentManager *ComponentManager
}

func (r *MachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 获取Machine资源
    machine := &v1beta1.Machine{}
    if err := r.Get(ctx, req.NamespacedName, machine); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 检查Machine是否有对应的NodeConfig
    nodeConfigName := fmt.Sprintf("%s-%s", machine.Namespace, machine.Name)
    nodeConfig := &nodecomponentv1alpha1.NodeConfig{}
    if err := r.Get(ctx, client.ObjectKey{Namespace: machine.Namespace, Name: nodeConfigName}, nodeConfig); err != nil {
        // 如果没有NodeConfig，创建一个
        log.Info("Creating NodeConfig for Machine", "machine", machine.Name)
        nodeConfig = r.buildNodeConfigFromMachine(ctx, machine)
        if err := r.Create(ctx, nodeConfig); err != nil {
            return ctrl.Result{}, fmt.Errorf("create NodeConfig failed: %v", err)
        }
    }
    
    // 检查节点组件是否安装完成
    if !r.isComponentInstallationComplete(nodeConfig) {
        log.Info("Waiting for components to be installed", "machine", machine.Name)
        // 更新Machine状态为Provisioning
        if machine.Status.Phase != v1beta1.MachinePhaseProvisioning {
            machine.Status.Phase = v1beta1.MachinePhaseProvisioning
            if err := r.Status().Update(ctx, machine); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 节点组件安装完成，更新Machine状态为Ready
    if machine.Status.Phase != v1beta1.MachinePhaseReady {
        log.Info("Machine components installed, updating status to Ready", "machine", machine.Name)
        machine.Status.Phase = v1beta1.MachinePhaseReady
        if err := r.Status().Update(ctx, machine); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

// 从Machine构建NodeConfig
func (r *MachineReconciler) buildNodeConfigFromMachine(ctx context.Context, machine *v1beta1.Machine) *nodecomponentv1alpha1.NodeConfig {
    // 从Machine标签获取集群信息
    clusterName := machine.Labels["cluster.x-k8s.io/cluster-name"]
    
    // 获取Cluster资源
    cluster := &clusterv1alpha1.Cluster{}
    r.Get(ctx, client.ObjectKey{Namespace: machine.Namespace, Name: clusterName}, cluster)
    
    // 从Machine获取节点IP
    // 这里假设IP在Machine的Annotation或Spec中
    nodeIP := machine.Annotations["nodecomponent.io/ip"]
    
    return &nodecomponentv1alpha1.NodeConfig{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-%s", clusterName, nodeIP),
            Namespace: machine.Namespace,
            Labels: map[string]string{
                "cluster": clusterName,
                "machine": machine.Name,
            },
        },
        Spec: nodecomponentv1alpha1.NodeConfigSpec{
            DeploymentMode: nodecomponentv1alpha1.DeploymentModeConfig{
                Type: "Remote",
                Remote: &nodecomponentv1alpha1.RemoteConfig{
                    Connection: nodecomponentv1alpha1.ConnectionConfig{
                        Host: nodeIP,
                        Port: 22,
                        User: "root",
                        SSHKeySecret: nodecomponentv1alpha1.SecretReference{
                            Name:      "node-ssh-key",
                            Namespace: "node-system",
                        },
                    },
                },
            },
            Components: cluster.Spec.Components,
        },
    }
}

// 检查组件安装是否完成
func (r *MachineReconciler) isComponentInstallationComplete(nodeConfig *nodecomponentv1alpha1.NodeConfig) bool {
    for _, componentStatus := range nodeConfig.Status.ComponentStatus {
        if componentStatus.Phase != "Ready" {
            return false
        }
    }
    return true
}
```
## 四、使用示例
### 4.1 初始化管理集群（包含CAPI）
#### 4.1.1 创建配置文件
```yaml
# init-management-cluster.yaml
apiVersion: nodecomponent.io/v1alpha1
kind: BootstrapFlow
metadata:
  name: init-management-cluster-with-capi
spec:
  bootstrapCluster:
    type: k3s
    k3s:
      version: v1.27.0
      dataDir: /var/lib/rancher/k3s-bootstrap
  
  managementCluster:
    kubernetesVersion: v1.28.2
    nodes:
      - host: 192.168.1.101
        role: master
      - host: 192.168.1.102
        role: master
      - host: 192.168.1.103
        role: master
    components:
      containerd:
        version: v1.7.2
      kubelet:
        version: v1.28.2
      cni:
        version: v1.3.0
  
  # CAPI配置
  capi:
    enabled: true
    version: v1.5.3
    providers:
      - name: kubeadm
        version: v1.5.3
      - name: capbke
        version: v1.0.0
  
  initialNodes:
    - connection:
        host: 192.168.1.101
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
          namespace: node-system
      role: management
      deploymentMode:
        type: Remote
      components:
        containerd:
          version: v1.7.2
        kubelet:
          version: v1.28.2
        cni:
          version: v1.3.0
  
  keepBootstrapCluster: false
```
#### 4.1.2 执行初始化
```bash
nodecomponent init --config init-management-cluster.yaml

# 输出：
# ℹ️  Creating bootstrap cluster (k3s)...
# ✅ Bootstrap cluster created successfully
# ℹ️  Installing CRDs to bootstrap cluster...
# ✅ CRDs installed successfully
# ℹ️  Installing components to initial nodes...
#    - Installing containerd on 192.168.1.101...
#    - Installing kubelet on 192.168.1.101...
#    - Installing cni on 192.168.1.101...
#    ...
# ✅ Components installed successfully
# ℹ️  Creating management cluster...
#    - Initializing cluster on 192.168.1.101...
#    - Joining 192.168.1.102 to cluster...
#    - Joining 192.168.1.103 to cluster...
# ✅ Management cluster created successfully
# ℹ️  Deploying CAPI components...
#    - Installing CAPI core components (v1.5.3)...
#    - Installing Kubeadm provider (v1.5.3)...
#    - Installing CAPBKE provider (v1.0.0)...
# ✅ CAPI components deployed successfully
# ℹ️  Migrating to management cluster...
#    - Installing CRDs to management cluster...
#    - Deploying controllers to management cluster...
#    - Migrating resources...
# ✅ Migration completed successfully
# ℹ️  Destroying bootstrap cluster...
# ✅ Bootstrap cluster destroyed
# 
# ✅ Management cluster initialized successfully with CAPI!
#    Kubeconfig: /root/.kube/config
#    CAPI Version: v1.5.3
#    Providers: kubeadm (v1.5.3), capbke (v1.0.0)
```
### 4.2 使用CAPI创建业务集群
#### 4.2.1 创建集群配置
```yaml
# workload-cluster.yaml
apiVersion: cluster.nodecomponent.io/v1alpha1
kind: Cluster
metadata:
  name: workload-cluster-1
  namespace: default
spec:
  clusterName: workload-cluster-1
  kubernetesVersion: v1.28.2
  
  controlPlane:
    replicas: 3
    infrastructureRef:
      kind: BKECluster
      name: workload-cluster-1-control-plane
    machineTemplate:
      infrastructureRef:
        kind: BKEMachineTemplate
        name: workload-cluster-1-control-plane
      bootstrap:
        configRef:
          kind: KubeadmConfigTemplate
          name: workload-cluster-1-control-plane
  
  workers:
    replicas: 3
    machineDeployments:
      - name: worker-pool-1
        replicas: 3
        infrastructureRef:
          kind: BKEMachineTemplate
          name: workload-cluster-1-worker
        bootstrap:
          configRef:
            kind: KubeadmConfigTemplate
            name: workload-cluster-1-worker
  
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
  
  infrastructure:
    provider: capbke
    version: v1.0.0
    config:
      bkeClusterConfig:
        network:
          podCIDR: 10.244.0.0/16
          serviceCIDR: 10.96.0.0/12
        etcd:
          storage: 20Gi
        master:
          nodes:
            - ip: 192.168.2.101
              role: master
            - ip: 192.168.2.102
              role: master
            - ip: 192.168.2.103
              role: master
        worker:
          nodes:
            - ip: 192.168.2.201
              role: worker
            - ip: 192.168.2.202
              role: worker
            - ip: 192.168.2.203
              role: worker
```
#### 4.2.2 创建集群
```bash
kubectl apply -f workload-cluster.yaml

# 输出：
# cluster.cluster.nodecomponent.io/workload-cluster-1 created

# 查看集群状态
kubectl get cluster workload-cluster-1

# 输出：
# NAME                  PHASE         CONTROL PLANE READY   WORKERS READY   AGE
# workload-cluster-1    Provisioning  0/3            0/3            5m

# 等待集群就绪
watch kubectl get cluster workload-cluster-1

# 最终输出：
# NAME                  PHASE   CONTROL PLANE READY   WORKERS READY   AGE
# workload-cluster-1    Ready   3/3                  3/3            20m

# 获取集群kubeconfig
kubectl get secret workload-cluster-1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > workload-cluster-1.kubeconfig

# 验证集群
kubectl --kubeconfig=workload-cluster-1.kubeconfig get nodes

# 输出：
# NAME                                STATUS   ROLES           AGE   VERSION
# workload-cluster-1-control-plane-0  Ready    control-plane   15m   v1.28.2
# workload-cluster-1-control-plane-1  Ready    control-plane   14m   v1.28.2
# workload-cluster-1-control-plane-2  Ready    control-plane   13m   v1.28.2
# workload-cluster-1-worker-0         Ready    <none>          10m   v1.28.2
# workload-cluster-1-worker-1         Ready    <none>          10m   v1.28.2
# workload-cluster-1-worker-2         Ready    <none>          10m   v1.28.2
```
### 4.3 扩展业务集群
```bash
# 增加工作节点数量
kubectl patch cluster workload-cluster-1 -p '{"spec":{"workers":{"machineDeployments":[{"name":"worker-pool-1","replicas":5}]}}}' --type=merge

# 查看状态
kubectl get cluster workload-cluster-1

# 输出：
# NAME                  PHASE   CONTROL PLANE READY   WORKERS READY   AGE
# workload-cluster-1    Ready   3/3                  5/5            1h
```
### 4.4 升级业务集群
```bash
# 升级Kubernetes版本
kubectl patch cluster workload-cluster-1 -p '{"spec":{"kubernetesVersion":"v1.29.0"}}' --type=merge

# 查看升级状态
kubectl get cluster workload-cluster-1

# 输出：
# NAME                  PHASE       CONTROL PLANE READY   WORKERS READY   AGE
# workload-cluster-1    Upgrading   3/3                  5/5            1h

# 等待升级完成
watch kubectl get cluster workload-cluster-1

# 最终输出：
# NAME                  PHASE   CONTROL PLANE READY   WORKERS READY   AGE
# workload-cluster-1    Ready   3/3                  5/5            2h
```
## 五、方案优势
### 5.1 标准化管理
| 特性 | 优势 |
|------|------|
| **Cluster API集成** | 使用标准的Kubernetes集群管理API |
| **统一接口** | 简化的自定义CRD封装复杂的CAPI资源 |
| **多Provider支持** | 支持CAPBKE、CAPD等多种Infrastructure Provider |
| **社区生态** | 兼容Cluster API社区工具和最佳实践 |
### 5.2 完整的生命周期管理
```
┌─────────────────────────────────────────────────────────────┐
│                    集群生命周期 (CAPI增强)                  │
│                                                             │
│  初始化 ──> 扩容 ──> 升级 ──> 维护 ──> 缩容 ──> 销毁        │
│    │         │        │        │        │        │          │
│    │         │        │        │        │        │          │
│  CAPI创建   Machine   Kubeadm   节点组件   Machine  CAPI删除│
│  Cluster    Scale    Upgrade   Upgrade   Scale    Cluster   │
└─────────────────────────────────────────────────────────────┘
```
### 5.3 与节点组件管理的无缝集成
| 场景 | 集成方式 | 优势 |
|------|---------|------|
| **节点创建** | 先安装节点组件，再标记Machine为Ready | 确保节点满足Kubernetes要求 |
| **节点升级** | 协调CAPI的Machine升级与组件升级 | 确保版本兼容性 |
| **节点维护** | 利用CAPI的Machine健康检查和修复 | 提高集群可靠性 |
| **故障处理** | 结合节点组件状态和CAPI事件 | 快速定位和解决问题 |
## 六、总结
本方案通过将Cluster API集成到管理集群中，实现了基于CAPI的标准化业务集群管理。核心创新点包括：
### 6.1 关键创新
1. **三层架构增强**：引导集群 → 管理集群(CAPI) → 业务集群
2. **统一Cluster CRD**：简化的用户接口封装复杂的CAPI资源
3. **组件生命周期联动**：节点组件管理与CAPI机器生命周期协调
4. **多Provider支持**：兼容多种基础设施提供者
### 6.2 核心价值
1. **标准化**：采用Cluster API标准，兼容社区生态
2. **简化使用**：通过自定义CRD降低用户使用复杂度
3. **灵活扩展**：支持多种基础设施和部署模式
4. **完整管理**：提供从初始化到销毁的完整集群生命周期管理

该方案解决了"鸡生蛋"问题，同时利用Cluster API提供的标准化集群管理能力，为用户提供了一个灵活、可靠的Kubernetes集群管理解决方案。


# Cluster API集成扩展方案：管理集群使用Cluster API安装业务集群
## 一、方案概述
基于之前的节点组件管理方案，扩展支持Cluster API作为业务集群的创建和管理框架，同时将节点组件安装与Infrastructure Machine紧密集成，实现从基础设施到节点组件的端到端管理。
### 核心设计理念
1. **Cluster API集成**：复用Cluster API的成熟集群生命周期管理能力
2. **Infrastructure Provider扩展**：开发自定义Infrastructure Provider对接节点组件管理系统
3. **Machine生命周期整合**：在Machine的创建、升级、删除等阶段无缝集成组件安装
4. **多部署模式支持**：兼容Remote/Image/Hybrid三种部署模式
5. **状态同步机制**：确保Cluster API状态与组件安装状态的一致性
## 二、架构设计
### 2.1 整体架构
```
┌───────────────────────────────────────────────────────────────────────────────┐
│                                  管理集群                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                     Cluster API Components                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐    │  │
│  │  │        CAPBKE Infrastructure Provider                           │    │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐    │    │  │
│  │  │  │ BKECluster      │  │ BKEMachine      │  │ BKEMachineSet │    │    │  │
│  │  │  │ Controller      │  │ Controller      │  │ Controller    │    │    │  │
│  │  │  └─────────────────┘  └─────────────────┘  └───────────────┘    │    │  │
│  │  │                     │                                           │    │  │
│  │  │                     ▼                                           │    │  │
│  │  │          ┌─────────────────────────┐                            │    │  │
│  │  │          │  NodeComponentManager   │                            │    │  │
│  │  │          │  (节点组件管理控制器)   │                            │    │  │
│  │  │          └─────────────────────────┘                            │    │  │
│  │  │                     │                                           │    │  │
│  │  │                     ▼                                           │    │  │
│  │  │          ┌─────────────────────────┐                            │    │  │
│  │  │          │   Provider Interface    │                            │    │  │
│  │  │          └─────────────────────────┘                            │    │  │
│  │  │                       │                                         │    │  │
│  │  │         ┌─────────────┼─────────────┐                           │    │  │
│  │  │         │             │             │                           │    │  │
│  │  │  ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────┐                    │    │  │
│  │  │  │ Remote      │ │ Image     │ │ Hybrid    │                    │    │  │
│  │  │  │ Provider    │ │ Provider  │ │ Provider  │                    │    │  │
│  │  │  └─────────────┘ └───────────┘ └───────────┘                    │    │  │
│  │  └─────────────────────────────────────────────────────────────────┘    │  │
│  │                                                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐    │  │
│  │  │              Cluster API Core Components                        │    │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐    │    │  │
│  │  │  │ Cluster         │  │ Machine         │  │ MachineSet    │    │    │  │
│  │  │  │ Controller      │  │ Controller      │  │ Controller    │    │    │  │
│  │  │  └─────────────────┘  └─────────────────┘  └───────────────┘    │    │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐    │    │  │
│  │  │  │ KubeadmControl  │  │ KubeadmBootstrap│  │ MachineDeploy-│    │    │  │
│  │  │  │ PlaneController │  │   Controller    │  │ mentController│    │    │  │
│  │  │  └─────────────────┘  └─────────────────┘  └───────────────┘    │    │  │
│  │  └─────────────────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 创建并管理
                                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                                  业务集群                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                     Kubernetes Components                               │  │
│  │  ┌─────────────────┐   ┌─────────────────┐  ┌───────────────┐           │  │
│  │  │  Control Plane  │   │    Workers      │  │    Nodes      │           │  │
│  │  │  (apiserver,    │   │   (kubelet,     │  │   (containerd,│           │  │
│  │  │  controller-manager,│  kube-proxy,    │  │   kubelet,    │           │  │
│  │  │  scheduler, etcd)│  │  cni)           │  │   cni, csi)   │           │  │
│  │  └─────────────────┘   └─────────────────┘  └───────────────┘           │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```
### 2.2 核心组件
#### 2.2.1 CAPBKE Infrastructure Provider
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: workload-cluster-1
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  
  # 引用基础设施Cluster
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: BKECluster
    name: workload-cluster-1
    namespace: clusters
```
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKECluster
metadata:
  name: workload-cluster-1
  namespace: clusters
spec:
  # 集群配置
  controlPlaneEndpoint:
    host: 192.168.1.100
    port: 6443
  
  # 组件配置
  components:
    containerd:
      version: v1.7.2
      config:
        dataDir: /var/lib/containerd
        systemdCgroup: true
    
    kubelet:
      version: v1.28.2
      config:
        dataDir: /var/lib/kubelet
    
    cni:
      version: v1.3.0
      config:
        pluginList:
          - bridge
          - loopback
          - portmap
    
    csi:
      version: v2.1.0

status:
  phase: Provisioned
  controlPlaneReady: true
  ready: true
```
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: workload-cluster-1-control-plane-0
  namespace: clusters
spec:
  clusterName: workload-cluster-1
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: workload-cluster-1-control-plane-0
  
  # 引用基础设施Machine
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: BKEMachine
    name: workload-cluster-1-control-plane-0
    namespace: clusters
  
  version: v1.28.2
```
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKEMachine
metadata:
  name: workload-cluster-1-control-plane-0
  namespace: clusters
spec:
  # 节点连接信息
  connection:
    host: 192.168.2.101
    port: 22
    user: root
    sshKeySecret:
      name: node-ssh-key
      namespace: clusters
  
  # 节点配置
  nodeConfig:
    os:
      type: linux
      distro: ubuntu
      version: "22.04"
      arch: amd64
    
    # 部署模式
    deploymentMode:
      type: Remote
    
    # 组件配置
    components:
      containerd:
        version: v1.7.2
        config:
          dataDir: /var/lib/containerd
          systemdCgroup: true
      
      kubelet:
        version: v1.28.2
        config:
          dataDir: /var/lib/kubelet
      
      cni:
        version: v1.3.0
        config:
          pluginList:
            - bridge
            - loopback
            - portmap
      
      csi:
        version: v2.1.0

status:
  phase: Running
  providerID: bke://192.168.2.101
  ready: true
```
#### 2.2.2 BKEMachine Controller实现
```go
package controllers

import (
    "context"
    "fmt"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/clientcmd"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    
    clusterapisdk "sigs.k8s.io/cluster-api/util/sdk"
    clusterapiv1beta1 "sigs.k8s.io/cluster-api/api/v1beta1"
    
    bkeapi "infrastructure.cluster.x-k8s.io/bke/api/v1alpha1"
    "infrastructure.cluster.x-k8s.io/bke/pkg/provider"
    "infrastructure.cluster.x-k8s.io/bke/pkg/ssh"
)

const (
    // BKEMachineFinalizer is the finalizer for BKEMachine resources.
    BKEMachineFinalizer = "bkemachine.infrastructure.cluster.x-k8s.io"
)

// BKEMachineReconciler reconciles a BKEMachine object.
type BKEMachineReconciler struct {
    client.Client
    Scheme          *runtime.Scheme
    SSHManager      *ssh.SSHManager
    ProviderFactory provider.ProviderFactory
}

//+kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=bkemachines/finalizers,verbs=update
//+kubebuilder:rbac:groups=cluster.x-k8s.io,resources=machines;machines/status,verbs=get;list;watch
//+kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch

func (r *BKEMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Reconciling BKEMachine")
    
    // 获取BKEMachine资源
    bkeMachine := &bkeapi.BKEMachine{}
    if err := r.Get(ctx, req.NamespacedName, bkeMachine); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 获取对应的Machine资源
    machine, err := clusterapisdk.MachineFromMetadata(ctx, r.Client, bkeMachine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 检查Machine是否标记为删除
    if !machine.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, bkeMachine, machine)
    }
    
    // 检查BKEMachine是否标记为删除
    if !bkeMachine.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, bkeMachine, machine)
    }
    
    // 确保finalizer存在
    if !controllerutil.ContainsFinalizer(bkeMachine, BKEMachineFinalizer) {
        controllerutil.AddFinalizer(bkeMachine, BKEMachineFinalizer)
        if err := r.Update(ctx, bkeMachine); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 检查Machine是否暂停
    if clusterapisdk.IsPaused(machine, bkeMachine) {
        log.Info("Machine is paused, skipping reconciliation")
        return ctrl.Result{}, nil
    }
    
    // 协调Machine状态
    return r.reconcileNormal(ctx, bkeMachine, machine)
}

// 协调Machine正常状态
func (r *BKEMachineReconciler) reconcileNormal(ctx context.Context, bkeMachine *bkeapi.BKEMachine, machine *clusterapiv1beta1.Machine) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 检查Machine是否已经准备好
    if bkeMachine.Status.Ready {
        return ctrl.Result{}, nil
    }
    
    // 获取对应的BKECluster资源
    bkeCluster := &bkeapi.BKECluster{}
    clusterName := types.NamespacedName{
        Name:      machine.Spec.ClusterName,
        Namespace: bkeMachine.Namespace,
    }
    if err := r.Get(ctx, clusterName, bkeCluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 创建Provider
    providerType := provider.ProviderType(bkeMachine.Spec.NodeConfig.DeploymentMode.Type)
    prov, err := r.ProviderFactory.CreateProvider(providerType)
    if err != nil {
        return ctrl.Result{}, fmt.Errorf("create provider failed: %v", err)
    }
    
    // 更新状态为Provisioning
    if bkeMachine.Status.Phase != bkeapi.MachinePhaseProvisioning {
        bkeMachine.Status.Phase = bkeapi.MachinePhaseProvisioning
        if err := r.Status().Update(ctx, bkeMachine); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 1. 初始化Provider
    if err := prov.Initialize(ctx, bkeMachine.Spec.NodeConfig); err != nil {
        return ctrl.Result{}, fmt.Errorf("initialize provider failed: %v", err)
    }
    
    // 2. 检测组件状态
    componentStatus, err := prov.DetectComponentStatus(ctx, bkeMachine.Spec.NodeConfig, "containerd")
    if err != nil {
        return ctrl.Result{}, fmt.Errorf("detect component status failed: %v", err)
    }
    
    // 3. 安装或升级组件
    if componentStatus.Phase == provider.PhasePending || componentStatus.InstalledVersion != bkeMachine.Spec.NodeConfig.Components["containerd"].Version {
        // 获取组件版本定义
        componentVersion := &bkeapi.ComponentVersion{}
        componentVersionName := fmt.Sprintf("containerd-%s", bkeMachine.Spec.NodeConfig.Components["containerd"].Version)
        if err := r.Get(ctx, types.NamespacedName{
            Name: componentVersionName,
        }, componentVersion); err != nil {
            return ctrl.Result{}, fmt.Errorf("get component version failed: %v", err)
        }
        
        // 执行安装或升级
        if componentStatus.Phase == provider.PhasePending {
            if err := prov.InstallComponent(ctx, bkeMachine.Spec.NodeConfig, componentVersion, bkeMachine.Spec.NodeConfig.Components["containerd"]); err != nil {
                return ctrl.Result{}, fmt.Errorf("install component failed: %v", err)
            }
        } else {
            if err := prov.UpgradeComponent(ctx, bkeMachine.Spec.NodeConfig, componentVersion, bkeMachine.Spec.NodeConfig.Components["containerd"]); err != nil {
                return ctrl.Result{}, fmt.Errorf("upgrade component failed: %v", err)
            }
        }
    }
    
    // 4. 安装其他组件（kubelet, cni, csi）
    for componentName := range bkeMachine.Spec.NodeConfig.Components {
        if componentName == "containerd" {
            continue // 已经安装过
        }
        
        // 检测组件状态
        componentStatus, err := prov.DetectComponentStatus(ctx, bkeMachine.Spec.NodeConfig, componentName)
        if err != nil {
            return ctrl.Result{}, fmt.Errorf("detect component status failed: %v", err)
        }
        
        // 安装组件
        if componentStatus.Phase == provider.PhasePending || componentStatus.InstalledVersion != bkeMachine.Spec.NodeConfig.Components[componentName].Version {
            // 获取组件版本定义
            componentVersion := &bkeapi.ComponentVersion{}
            componentVersionName := fmt.Sprintf("%s-%s", componentName, bkeMachine.Spec.NodeConfig.Components[componentName].Version)
            if err := r.Get(ctx, types.NamespacedName{
                Name: componentVersionName,
            }, componentVersion); err != nil {
                return ctrl.Result{}, fmt.Errorf("get component version failed: %v", err)
            }
            
            // 执行安装或升级
            if componentStatus.Phase == provider.PhasePending {
                if err := prov.InstallComponent(ctx, bkeMachine.Spec.NodeConfig, componentVersion, bkeMachine.Spec.NodeConfig.Components[componentName]); err != nil {
                    return ctrl.Result{}, fmt.Errorf("install component failed: %v", err)
                }
            } else {
                if err := prov.UpgradeComponent(ctx, bkeMachine.Spec.NodeConfig, componentVersion, bkeMachine.Spec.NodeConfig.Components[componentName]); err != nil {
                    return ctrl.Result{}, fmt.Errorf("upgrade component failed: %v", err)
                }
            }
        }
    }
    
    // 5. 验证所有组件健康状态
    for componentName := range bkeMachine.Spec.NodeConfig.Components {
        if err := prov.HealthCheck(ctx, bkeMachine.Spec.NodeConfig, componentName); err != nil {
            return ctrl.Result{}, fmt.Errorf("component health check failed: %v", err)
        }
    }
    
    // 6. 设置ProviderID
    if bkeMachine.Status.ProviderID == "" {
        bkeMachine.Status.ProviderID = fmt.Sprintf("bke://%s", bkeMachine.Spec.Connection.Host)
    }
    
    // 7. 更新状态为Running和Ready
    bkeMachine.Status.Phase = bkeapi.MachinePhaseRunning
    bkeMachine.Status.Ready = true
    bkeMachine.Status.Addresses = []corev1.NodeAddress{
        {
            Type:    corev1.NodeInternalIP,
            Address: bkeMachine.Spec.Connection.Host,
        },
    }
    
    if err := r.Status().Update(ctx, bkeMachine); err != nil {
        return ctrl.Result{}, err
    }
    
    log.Info("BKEMachine provisioned successfully")
    return ctrl.Result{}, nil
}

// 协调Machine删除状态
func (r *BKEMachineReconciler) reconcileDelete(ctx context.Context, bkeMachine *bkeapi.BKEMachine, machine *clusterapiv1beta1.Machine) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Deleting BKEMachine")
    
    // 清理资源
    if controllerutil.ContainsFinalizer(bkeMachine, BKEMachineFinalizer) {
        // 执行清理操作，如卸载组件、清理数据等
        if err := r.cleanupMachine(ctx, bkeMachine); err != nil {
            return ctrl.Result{}, err
        }
        
        // 移除finalizer
        controllerutil.RemoveFinalizer(bkeMachine, BKEMachineFinalizer)
        if err := r.Update(ctx, bkeMachine); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

// 清理Machine资源
func (r *BKEMachineReconciler) cleanupMachine(ctx context.Context, bkeMachine *bkeapi.BKEMachine) error {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Cleaning up BKEMachine resources")
    
    // 创建Provider
    providerType := provider.ProviderType(bkeMachine.Spec.NodeConfig.DeploymentMode.Type)
    prov, err := r.ProviderFactory.CreateProvider(providerType)
    if err != nil {
        return fmt.Errorf("create provider failed: %v", err)
    }
    
    // 清理组件
    if err := prov.Cleanup(ctx, bkeMachine.Spec.NodeConfig); err != nil {
        log.Error(err, "Cleanup provider resources failed, ignoring")
    }
    
    // 清理SSH连接
    key := fmt.Sprintf("%s:%d", bkeMachine.Spec.Connection.Host, bkeMachine.Spec.Connection.Port)
    r.SSHManager.RemoveClient(key)
    
    return nil
}

func (r *BKEMachineReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&bkeapi.BKEMachine{}).
        Owns(&clusterapiv1beta1.Machine{}).
        Complete(r)
}
```
### 2.3 组件安装与Machine生命周期的集成
#### 2.3.1 Machine创建流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Machine创建流程                                  │
└─────────────────────────────────────────────────────────────────────┘

┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ Cluster API       │    │ CAPBKE Provider   │    │ NodeComponent     │
│ Machine Controller│    │ BKEMachine        │    │ Manager           │
│                   │    │ Controller        │    │                   │
└─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
          │                        │                        │
          │ Create BKEMachine      │                        │
          └───────────────────────>│                        │
                                    │                        │
                                    │ Create Provider        │
                                    └───────────────────────>│
                                    │                        │
                                    │ Initialize Provider    │
                                    <───────────────────────┤
                                    │                        │
                                    │ Detect Component Status│
                                    └───────────────────────>│
                                    │                        │
                                    │ Install Components      │
                                    <───────────────────────┤
                                    │                        │
                                    │ Health Check            │
                                    └───────────────────────>│
                                    │                        │
                                    │ Update BKEMachine Status│
                                    <───────────────────────┤
                                    │                        │
                                    │ Set ProviderID          │
                                    └───────────────────────>│
                                    │                        │
                                    │ Update Machine Status   │
                                    <───────────────────────┤
                                    │                        │
                                    │ Machine Ready           │
                                    └───────────────────────>│
```
#### 2.3.2 Machine升级流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Machine升级流程                                  │
└─────────────────────────────────────────────────────────────────────┘

┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ Cluster API       │    │ CAPBKE Provider   │    │ NodeComponent     │
│ Machine Controller│    │ BKEMachine        │    │ Manager           │
│                   │    │ Controller        │    │                   │
└─────────┬─────────┘    └─────────┬─────────┘    └───────────────────┘
          │                        │                        │
          │ Update Machine Spec    │                        │
          └───────────────────────>│                        │
                                    │                        │
                                    │ Detect Version Change  │
                                    └───────────────────────>│
                                    │                        │
                                    │ Check Upgrade Path     │
                                    <───────────────────────┤
                                    │                        │
                                    │ Backup Current Version │
                                    └───────────────────────>│
                                    │                        │
                                    │ Upgrade Components     │
                                    <───────────────────────┤
                                    │                        │
                                    │ Health Check            │
                                    └───────────────────────>│
                                    │                        │
                                    │ Update Status          │
                                    <───────────────────────┤
                                    │                        │
                                    │ Upgrade Complete        │
                                    └───────────────────────>│
```
### 2.4 ClusterTemplate与MachineTemplate设计
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterTemplate
metadata:
  name: bke-cluster-template
  namespace: clusters
spec:
  clusterClassRef:
    name: bke-cluster-class
  
  template:
    spec:
      clusterNetwork:
        pods:
          cidrBlocks: ["192.168.0.0/16"]
        services:
          cidrBlocks: ["10.96.0.0/12"]
      
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: BKEClusterTemplate
        name: bke-cluster-template
        namespace: clusters
```

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKEClusterTemplate
metadata:
  name: bke-cluster-template
  namespace: clusters
spec:
  template:
    spec:
      controlPlaneEndpoint:
        host: ${CONTROL_PLANE_ENDPOINT}
        port: 6443
      
      components:
        containerd:
          version: ${CONTAINERD_VERSION}
          config:
            systemdCgroup: true
        
        kubelet:
          version: ${KUBERNETES_VERSION}
        
        cni:
          version: ${CNI_VERSION}
        
        csi:
          version: ${CSI_VERSION}
```

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineTemplate
metadata:
  name: bke-control-plane-template
  namespace: clusters
spec:
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: bke-control-plane-template
      
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: BKEMachineTemplate
        name: bke-control-plane-template
      
      version: ${KUBERNETES_VERSION}
```

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKEMachineTemplate
metadata:
  name: bke-control-plane-template
  namespace: clusters
spec:
  template:
    spec:
      connection:
        host: ${NODE_IP}
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
      
      nodeConfig:
        os:
          type: linux
          distro: ubuntu
          version: "22.04"
          arch: amd64
        
        deploymentMode:
          type: Remote
        
        components:
          containerd:
            version: ${CONTAINERD_VERSION}
            config:
              systemdCgroup: true
          
          kubelet:
            version: ${KUBERNETES_VERSION}
          
          cni:
            version: ${CNI_VERSION}
          
          csi:
            version: ${CSI_VERSION}
```
## 三、详细实现
### 3.1 组件安装与Machine的集成
#### 3.1.1 Provider接口扩展
```go
package provider

import (
    "context"
    
    bkeapi "infrastructure.cluster.x-k8s.io/bke/api/v1alpha1"
)

// MachineProvider扩展了Provider接口，支持与Cluster API Machine的集成
type MachineProvider interface {
    Provider
    
    // 从BKEMachine中获取节点配置
    GetNodeConfig(bkeMachine *bkeapi.BKEMachine) *NodeConfig
    
    // 设置Machine状态
    SetMachineStatus(ctx context.Context, bkeMachine *bkeapi.BKEMachine) error
    
    // 验证Machine配置
    ValidateMachineConfig(ctx context.Context, bkeMachine *bkeapi.BKEMachine) error
}
```
#### 3.1.2 RemoteMachineProvider实现
```go
package provider

import (
    "context"
    "fmt"
    
    bkeapi "infrastructure.cluster.x-k8s.io/bke/api/v1alpha1"
)

type RemoteMachineProvider struct {
    *RemoteProvider
}

func NewRemoteMachineProvider(sshManager *SSHManager) *RemoteMachineProvider {
    return &RemoteMachineProvider{
        RemoteProvider: NewRemoteProvider(sshManager),
    }
}

func (p *RemoteMachineProvider) GetNodeConfig(bkeMachine *bkeapi.BKEMachine) *NodeConfig {
    return &bkeMachine.Spec.NodeConfig
}

func (p *RemoteMachineProvider) SetMachineStatus(ctx context.Context, bkeMachine *bkeapi.BKEMachine) error {
    // 设置ProviderID
    bkeMachine.Status.ProviderID = fmt.Sprintf("bke://%s", bkeMachine.Spec.Connection.Host)
    
    // 设置节点地址
    bkeMachine.Status.Addresses = []corev1.NodeAddress{
        {
            Type:    corev1.NodeInternalIP,
            Address: bkeMachine.Spec.Connection.Host,
        },
    }
    
    return nil
}

func (p *RemoteMachineProvider) ValidateMachineConfig(ctx context.Context, bkeMachine *bkeapi.BKEMachine) error {
    // 验证连接信息
    if bkeMachine.Spec.Connection.Host == "" {
        return fmt.Errorf("connection host is required")
    }
    
    if bkeMachine.Spec.Connection.Port == 0 {
        return fmt.Errorf("connection port is required")
    }
    
    // 验证组件配置
    for componentName, config := range bkeMachine.Spec.NodeConfig.Components {
        if config.Version == "" {
            return fmt.Errorf("version for component %s is required", componentName)
        }
    }
    
    return nil
}
```
### 3.2 Cluster API集成点
#### 3.2.1 ClusterReconciler扩展
```go
package controllers

import (
    "context"
    "fmt"
    
    clusterapiv1beta1 "sigs.k8s.io/cluster-api/api/v1beta1"
    clusterapisdk "sigs.k8s.io/cluster-api/util/sdk"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    bkeapi "infrastructure.cluster.x-k8s.io/bke/api/v1alpha1"
    "infrastructure.cluster.x-k8s.io/bke/pkg/manager"
)

// BKEClusterReconciler reconciles a BKECluster object.
type BKEClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    
    // 组件版本管理器
    VersionManager *manager.VersionManager
}

func (r *BKEClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Reconciling BKECluster")
    
    // 获取BKECluster资源
    bkeCluster := &bkeapi.BKECluster{}
    if err := r.Get(ctx, req.NamespacedName, bkeCluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 获取对应的Cluster资源
    cluster, err := clusterapisdk.ClusterFromMetadata(ctx, r.Client, bkeCluster.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 检查Cluster是否标记为删除
    if !cluster.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, bkeCluster, cluster)
    }
    
    // 检查BKECluster是否标记为删除
    if !bkeCluster.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, bkeCluster, cluster)
    }
    
    // 协调Cluster正常状态
    return r.reconcileNormal(ctx, bkeCluster, cluster)
}

func (r *BKEClusterReconciler) reconcileNormal(ctx context.Context, bkeCluster *bkeapi.BKECluster, cluster *clusterapiv1beta1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 检查组件版本兼容性
    if err := r.VersionManager.CheckCompatibility(ctx, bkeCluster.Spec.Components); err != nil {
        return ctrl.Result{}, fmt.Errorf("check component compatibility failed: %v", err)
    }
    
    // 更新状态为Provisioned
    if bkeCluster.Status.Phase != bkeapi.ClusterPhaseProvisioned {
        bkeCluster.Status.Phase = bkeapi.ClusterPhaseProvisioned
        bkeCluster.Status.ControlPlaneReady = true
        bkeCluster.Status.Ready = true
        
        if err := r.Status().Update(ctx, bkeCluster); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    log.Info("BKECluster provisioned successfully")
    return ctrl.Result{}, nil
}

func (r *BKEClusterReconciler) reconcileDelete(ctx context.Context, bkeCluster *bkeapi.BKECluster, cluster *clusterapiv1beta1.Cluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Deleting BKECluster")
    
    // 更新状态为Deleting
    bkeCluster.Status.Phase = bkeapi.ClusterPhaseDeleting
    if err := r.Status().Update(ctx, bkeCluster); err != nil {
        return ctrl.Result{}, err
    }
    
    // 执行清理操作
    // ...
    
    log.Info("BKECluster deleted successfully")
    return ctrl.Result{}, nil
}
```
### 3.3 业务集群创建示例
#### 3.3.1 创建Cluster资源
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: workload-cluster-1
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: BKECluster
    name: workload-cluster-1
    namespace: clusters
```
#### 3.3.2 创建BKECluster资源
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKECluster
metadata:
  name: workload-cluster-1
  namespace: clusters
spec:
  controlPlaneEndpoint:
    host: 192.168.1.100
    port: 6443
  
  components:
    containerd:
      version: v1.7.2
      config:
        systemdCgroup: true
    
    kubelet:
      version: v1.28.2
    
    cni:
      version: v1.3.0
    
    csi:
      version: v2.1.0
```
#### 3.3.3 创建Control Plane Machine
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: workload-cluster-1-control-plane-0
  namespace: clusters
spec:
  clusterName: workload-cluster-1
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: workload-cluster-1-control-plane-0
  
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: BKEMachine
    name: workload-cluster-1-control-plane-0
    namespace: clusters
  
  version: v1.28.2
```
#### 3.3.4 创建BKEMachine资源（Control Plane）
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKEMachine
metadata:
  name: workload-cluster-1-control-plane-0
  namespace: clusters
spec:
  connection:
    host: 192.168.2.101
    port: 22
    user: root
    sshKeySecret:
      name: node-ssh-key
      namespace: clusters
  
  nodeConfig:
    os:
      type: linux
      distro: ubuntu
      version: "22.04"
      arch: amd64
    
    deploymentMode:
      type: Remote
    
    components:
      containerd:
        version: v1.7.2
        config:
          systemdCgroup: true
      
      kubelet:
        version: v1.28.2
      
      cni:
        version: v1.3.0
      
      csi:
        version: v2.1.0
```
#### 3.3.5 创建Worker MachineDeployment
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: workload-cluster-1-md-0
  namespace: clusters
spec:
  clusterName: workload-cluster-1
  replicas: 2
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: workload-cluster-1
      cluster.x-k8s.io/deployment-name: workload-cluster-1-md-0
  
  template:
    spec:
      clusterName: workload-cluster-1
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: workload-cluster-1-md-0
      
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: BKEMachineTemplate
        name: workload-cluster-1-md-0
      
      version: v1.28.2
```
#### 3.3.6 创建BKEMachineTemplate资源（Worker）
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: BKEMachineTemplate
metadata:
  name: workload-cluster-1-md-0
  namespace: clusters
spec:
  template:
    spec:
      connection:
        host: ${NODE_IP}
        port: 22
        user: root
        sshKeySecret:
          name: node-ssh-key
      
      nodeConfig:
        os:
          type: linux
          distro: ubuntu
          version: "22.04"
          arch: amd64
        
        deploymentMode:
          type: Remote
        
        components:
          containerd:
            version: v1.7.2
            config:
              systemdCgroup: true
          
          kubelet:
            version: v1.28.2
          
          cni:
            version: v1.3.0
          
          csi:
            version: v2.1.0
```
## 四、方案优势
### 4.1 Cluster API集成的优势
1. **成熟的生命周期管理**：复用Cluster API的集群、机器生命周期管理能力
2. **标准化接口**：遵循Cluster API的标准接口，便于与其他Provider集成
3. **模板化部署**：支持ClusterTemplate和MachineTemplate，实现快速部署
4. **升级策略**：支持滚动升级、批量升级等多种升级策略
5. **状态同步**：自动同步Cluster API状态与节点组件安装状态
### 4.2 组件安装与Infrastructure Machine集成的优势
1. **端到端管理**：从基础设施到节点组件的完整管理流程
2. **状态一致性**：确保Machine状态与组件安装状态一致
3. **多部署模式**：支持Remote/Image/Hybrid三种部署模式
4. **灵活的组件管理**：每个Machine可以独立配置组件版本和部署模式
5. **统一的监控和管理**：通过Cluster API统一监控和管理所有集群
### 4.3 与之前方案的兼容性
1. **向后兼容**：保持与之前节点组件管理方案的兼容性
2. **平滑迁移**：支持从之前的方案迁移到Cluster API集成方案
3. **复用代码**：复用之前的Provider实现和组件管理逻辑
4. **统一的用户体验**：保持一致的配置方式和管理界面
## 五、总结
本方案通过开发自定义的Cluster API Infrastructure Provider（CAPBKE），将节点组件管理系统与Cluster API无缝集成，实现了：
1. **管理集群使用Cluster API安装业务集群**：利用Cluster API的成熟能力管理业务集群生命周期
2. **组件安装与Infrastructure Machine的紧密集成**：在Machine的创建、升级、删除等阶段自动管理节点组件
3. **支持多种部署模式**：兼容Remote/Image/Hybrid三种部署模式
4. **统一的状态管理**：确保Cluster API状态与组件安装状态的一致性
5. **模板化部署**：支持ClusterTemplate和MachineTemplate，实现快速部署

该方案既充分利用了Cluster API的成熟特性，又保留了节点组件管理系统的灵活性，为用户提供了从基础设施到节点组件的端到端管理解决方案。

    
# Cluster API集成的节点组件管理方案
## 一、方案概述
将节点组件管理与Cluster API深度集成，支持管理集群通过Cluster API安装业务集群，并实现组件安装与Infrastructure Machine的无缝衔接。
### 核心设计理念
1. **Cluster API原生集成**：扩展Cluster API资源，而非独立并行
2. **Infrastructure Machine驱动**：将组件安装作为Infrastructure Machine的生命周期一部分
3. **声明式组件管理**：使用CRD定义组件配置和版本
4. **统一生命周期管理**：组件安装、升级与Machine生命周期同步
## 二、架构设计
### 2.1 整体架构
```
┌───────────────────────────────────────────────────────────────────────┐
│                         管理集群                                      │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                     Cluster API栈                               │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │  │
│  │  │ Cluster         │  │ KubeadmControl  │  │ Machine         │  │  │
│  │  │ Controller      │  │ Plane Controller│  │ Controller      │  │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘  │  │
│  │                         │                     │                 │  │
│  │                         ▼                     ▼                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │            Custom Infrastructure Provider               │    │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐               │    │  │
│  │  │  │ Machine Driver  │  │ Component       │               │    │  │
│  │  │  │ (如: Metal3)    │  │ Installer       │               │    │  │
│  │  │  └─────────────────┘  └─────────────────┘               │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                         │                     │                 │  │
│  │                         ▼                     ▼                 │  │
│  │  ┌─────────────────┐  ┌───────────────────────────────────┐     │  │
│  │  │ ComponentConfig │  │ ComponentVersion                  │     │  │
│  │  │ CRD             │  │ CRD                               │     │  │
│  │  └─────────────────┘  └───────────────────────────────────┘     │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                        │
│                           API调用                                     │
│                              │                                        │
└───────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 创建
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         业务集群                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     Kubernetes集群                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │  │
│  │  │ Control Plane   │  │ Worker Nodes    │  │ Addons          │  │  │
│  │  │ (kube-apiserver,│  │                 │  │ (CNI, CSI, ...) │  │  │
│  │  │ kube-scheduler, │  │                 │  │                 │  │  │
│  │  │ kube-controller)│  │                 │  │                 │  │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```
### 2.2 核心资源定义
#### 2.2.1 ComponentConfig CRD
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentConfig
metadata:
  name: container-runtime-config
spec:
  # 组件类型
  componentType: container-runtime  # container-runtime, kubelet, cni, csi
  
  # 组件配置
  config:
    # containerd配置示例
    containerd:
      version: 1.7.11
      config:
        dataDir: /var/lib/containerd
        registry:
          mirrors:
            docker.io:
              endpoint:
                - https://registry-1.docker.io
        plugins:
          io.containerd.grpc.v1.cri:
            systemd_cgroup: true
            containerd:
              default_runtime_name: runc
  
  # 适用的基础设施类型
  infrastructureRef:
    kind: Metal3MachineTemplate
    name: control-plane-template
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1

status:
  conditions:
    - type: Ready
      status: "True"
      reason: ConfigValid
      message: Component config is valid
  observedGeneration: 1
```
#### 2.2.2 ComponentVersion CRD
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.11
spec:
  componentType: container-runtime
  name: containerd
  version: 1.7.11
  
  # 组件来源
  source:
    type: HTTP
    url: https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-amd64.tar.gz
    checksum: sha256:abc123...
  
  # 安装脚本
  installScript: |
    #!/bin/bash
    set -e
    
    # 下载并解压
    curl -L -o /tmp/containerd.tar.gz {{.Source.URL}}
    sha256sum -c <<< "{{.Source.Checksum}} /tmp/containerd.tar.gz"
    
    tar -C /usr/local -xzf /tmp/containerd.tar.gz
    
    # 安装systemd服务
    mkdir -p /etc/systemd/system
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
    
    # 配置
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    
    # 启用并启动
    systemctl daemon-reload
    systemctl enable --now containerd
  
  # 升级脚本
  upgradeScript: |
    #!/bin/bash
    set -e
    
    # 备份当前版本
    cp /usr/local/bin/containerd /usr/local/bin/containerd.bak
    
    # 下载并解压新版本
    curl -L -o /tmp/containerd.tar.gz {{.Source.URL}}
    sha256sum -c <<< "{{.Source.Checksum}} /tmp/containerd.tar.gz"
    
    tar -C /usr/local -xzf /tmp/containerd.tar.gz
    
    # 重启服务
    systemctl restart containerd
    
    # 验证
    if ! systemctl is-active containerd; then
      # 回滚
      cp /usr/local/bin/containerd.bak /usr/local/bin/containerd
      systemctl restart containerd
      exit 1
    fi
  
  # 兼容性信息
  compatibility:
    kubernetesVersions:
      min: "1.26.0"
      max: "1.30.0"
    os:
      - name: ubuntu
        versions: ["20.04", "22.04"]
        architectures: ["amd64", "arm64"]
      - name: centos
        versions: ["7", "8", "9"]
        architectures: ["amd64"]

status:
  conditions:
    - type: Ready
      status: "True"
      reason: VersionValid
      message: Component version is valid
  downloaded: true
  downloadTime: "2024-01-15T10:00:00Z"
```
#### 2.2.3 扩展Machine CRD
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: control-plane-0
spec:
  clusterName: my-cluster
  version: v1.28.2
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: KubeadmConfig
      name: control-plane-0
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: Metal3Machine
    name: control-plane-0
  # 新增：节点组件配置
  components:
    - name: container-runtime
      configRef:
        apiVersion: cluster.x-k8s.io/v1beta1
        kind: ComponentConfig
        name: container-runtime-config
    - name: kubelet
      configRef:
        apiVersion: cluster.x-k8s.io/v1beta1
        kind: ComponentConfig
        name: kubelet-config
    - name: cni
      configRef:
        apiVersion: cluster.x-k8s.io/v1beta1
        kind: ComponentConfig
        name: cni-config
```
#### 2.2.4 扩展MachineTemplate CRD
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineTemplate
metadata:
  name: control-plane-template
spec:
  template:
    spec:
      clusterName: my-cluster
      version: v1.28.2
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: control-plane-template
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: Metal3MachineTemplate
        name: control-plane-template
      # 新增：节点组件配置模板
      components:
        - name: container-runtime
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: container-runtime-config
        - name: kubelet
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: kubelet-config
        - name: cni
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: cni-config
```
## 三、Infrastructure Provider集成
### 3.1 Custom Infrastructure Provider设计
```go
package metal3

import (
    "context"
    "fmt"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/klog/v2"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    clusterapiv1beta1 "cluster.x-k8s.io/api/v1beta1"
    infrastructurev1beta1 "cluster.x-k8s.io/api/infrastructure/v1beta1"
    "cluster.x-k8s.io/cluster-api/controllers/noderefutil"
    "cluster.x-k8s.io/cluster-api/util"
    "cluster.x-k8s.io/cluster-api/util/annotations"
    "cluster.x-k8s.io/cluster-api/util/patch"
    
    "myprovider.io/api/v1beta1"
    "myprovider.io/provider/component"
)

// Metal3MachineReconciler reconciles a Metal3Machine object
type Metal3MachineReconciler struct {
    client.Client
    Scheme                 *runtime.Scheme
    ComponentInstaller     component.Installer
    SSHManager             *SSHManager
}

// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=metal3machines,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=metal3machines/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=infrastructure.cluster.x-k8s.io,resources=metal3machines/finalizers,verbs=update
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=machines,verbs=get;list;watch
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=componentconfigs,verbs=get;list;watch
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=componentversions,verbs=get;list;watch

func (r *Metal3MachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Reconciling Metal3Machine")
    
    // 获取Metal3Machine
    metal3Machine := &v1beta1.Metal3Machine{}
    if err := r.Get(ctx, req.NamespacedName, metal3Machine); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 获取对应的Machine
    machine, err := util.GetMachineFromMetadata(ctx, r.Client, metal3Machine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 获取Cluster
    cluster, err := util.GetClusterFromMetadata(ctx, r.Client, metal3Machine.ObjectMeta)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 创建patch
    helper, err := patch.NewHelper(metal3Machine, r.Client)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // 处理删除
    if !metal3Machine.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, metal3Machine, machine, cluster)
    }
    
    // 处理创建/更新
    return r.reconcileNormal(ctx, helper, metal3Machine, machine, cluster)
}

func (r *Metal3MachineReconciler) reconcileNormal(ctx context.Context, helper *patch.Helper,
    metal3Machine *v1beta1.Metal3Machine, machine *clusterapiv1beta1.Machine, cluster *clusterapiv1beta1.Cluster) (ctrl.Result, error) {
    
    log := ctrl.LoggerFrom(ctx)
    
    // 检查是否已经完成
    if metal3Machine.Status.Ready {
        return ctrl.Result{}, nil
    }
    
    // 检查基础设施是否就绪
    if !metal3Machine.Status.InfrastructureReady {
        log.Info("Infrastructure not ready, waiting")
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // 获取节点IP
    nodeIP := metal3Machine.Status.Addresses[0].Address
    
    // 安装节点组件
    if err := r.installComponents(ctx, metal3Machine, machine, cluster, nodeIP); err != nil {
        log.Error(err, "Failed to install components")
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    // 更新状态
    metal3Machine.Status.Ready = true
    if err := helper.Patch(ctx, metal3Machine); err != nil {
        return ctrl.Result{}, err
    }
    
    log.Info("Metal3Machine ready")
    return ctrl.Result{}, nil
}

func (r *Metal3MachineReconciler) installComponents(ctx context.Context,
    metal3Machine *v1beta1.Metal3Machine, machine *clusterapiv1beta1.Machine,
    cluster *clusterapiv1beta1.Cluster, nodeIP string) error {
    
    log := ctrl.LoggerFrom(ctx)
    
    // 检查Machine是否有组件配置
    if machine.Spec.Components == nil || len(machine.Spec.Components) == 0 {
        log.Info("No components to install")
        return nil
    }
    
    // 获取SSH密钥
    sshKey, err := r.getSSHKey(ctx, metal3Machine)
    if err != nil {
        return fmt.Errorf("failed to get SSH key: %v", err)
    }
    
    // 创建SSH客户端
    sshClient, err := r.SSHManager.NewClient(nodeIP, 22, "root", sshKey)
    if err != nil {
        return fmt.Errorf("failed to create SSH client: %v", err)
    }
    defer sshClient.Close()
    
    // 安装每个组件
    for _, componentSpec := range machine.Spec.Components {
        // 获取ComponentConfig
        componentConfig := &clusterapiv1beta1.ComponentConfig{}
        configRef := componentSpec.ConfigRef
        configKey := types.NamespacedName{
            Namespace: configRef.Namespace,
            Name:      configRef.Name,
        }
        
        if err := r.Get(ctx, configKey, componentConfig); err != nil {
            return fmt.Errorf("failed to get ComponentConfig %s: %v", configKey, err)
        }
        
        // 获取ComponentVersion
        componentType := componentConfig.Spec.ComponentType
        componentVersionName := fmt.Sprintf("%s-v%s", componentType, componentConfig.Spec.Config[componentType]["version"])
        
        componentVersion := &clusterapiv1beta1.ComponentVersion{}
        versionKey := types.NamespacedName{
            Namespace: metal3Machine.Namespace,
            Name:      componentVersionName,
        }
        
        if err := r.Get(ctx, versionKey, componentVersion); err != nil {
            return fmt.Errorf("failed to get ComponentVersion %s: %v", versionKey, err)
        }
        
        // 执行组件安装
        log.Info("Installing component", "component", componentType, "version", componentVersion.Spec.Version)
        
        if err := r.ComponentInstaller.Install(ctx, componentVersion, componentConfig, sshClient); err != nil {
            return fmt.Errorf("failed to install component %s: %v", componentType, err)
        }
        
        log.Info("Component installed successfully", "component", componentType)
    }
    
    return nil
}

func (r *Metal3MachineReconciler) getSSHKey(ctx context.Context, metal3Machine *v1beta1.Metal3Machine) ([]byte, error) {
    // 从Secret获取SSH密钥
    sshKeySecret := &corev1.Secret{}
    secretKey := types.NamespacedName{
        Namespace: metal3Machine.Namespace,
        Name:      metal3Machine.Spec.SSHKeySecret.Name,
    }
    
    if err := r.Get(ctx, secretKey, sshKeySecret); err != nil {
        return nil, err
    }
    
    return sshKeySecret.Data["privateKey"], nil
}
```
### 3.2 组件安装器
```go
package component

import (
    "context"
    "fmt"
    "strings"
    "text/template"
    
    clusterapiv1beta1 "cluster.x-k8s.io/api/v1beta1"
)

// Installer defines the interface for installing components
type Installer interface {
    Install(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
        config *clusterapiv1beta1.ComponentConfig, sshClient SSHClient) error
    
    Upgrade(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
        config *clusterapiv1beta1.ComponentConfig, sshClient SSHClient) error
    
    HealthCheck(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
        sshClient SSHClient) error
}

// SSHClient defines the interface for SSH operations
type SSHClient interface {
    ExecuteCommand(cmd string) (string, error)
    UploadFile(remotePath string, data []byte) error
    DownloadFile(remotePath string) ([]byte, error)
    Close() error
}

// DefaultInstaller implements Installer interface
type DefaultInstaller struct{}

func NewDefaultInstaller() *DefaultInstaller {
    return &DefaultInstaller{}
}

func (i *DefaultInstaller) Install(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
    config *clusterapiv1beta1.ComponentConfig, sshClient SSHClient) error {
    
    // 渲染安装脚本
    installScript, err := i.renderScript(version.Spec.InstallScript, version, config)
    if err != nil {
        return fmt.Errorf("failed to render install script: %v", err)
    }
    
    // 上传脚本
    if err := sshClient.UploadFile("/tmp/install-component.sh", []byte(installScript)); err != nil {
        return fmt.Errorf("failed to upload install script: %v", err)
    }
    
    // 执行脚本
    cmd := "chmod +x /tmp/install-component.sh && /tmp/install-component.sh"
    output, err := sshClient.ExecuteCommand(cmd)
    if err != nil {
        return fmt.Errorf("failed to execute install script: %v, output: %s", err, output)
    }
    
    // 清理脚本
    sshClient.ExecuteCommand("rm /tmp/install-component.sh")
    
    return nil
}

func (i *DefaultInstaller) Upgrade(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
    config *clusterapiv1beta1.ComponentConfig, sshClient SSHClient) error {
    
    // 渲染升级脚本
    upgradeScript, err := i.renderScript(version.Spec.UpgradeScript, version, config)
    if err != nil {
        return fmt.Errorf("failed to render upgrade script: %v", err)
    }
    
    // 上传脚本
    if err := sshClient.UploadFile("/tmp/upgrade-component.sh", []byte(upgradeScript)); err != nil {
        return fmt.Errorf("failed to upload upgrade script: %v", err)
    }
    
    // 执行脚本
    cmd := "chmod +x /tmp/upgrade-component.sh && /tmp/upgrade-component.sh"
    output, err := sshClient.ExecuteCommand(cmd)
    if err != nil {
        return fmt.Errorf("failed to execute upgrade script: %v, output: %s", err, output)
    }
    
    // 清理脚本
    sshClient.ExecuteCommand("rm /tmp/upgrade-component.sh")
    
    return nil
}

func (i *DefaultInstaller) HealthCheck(ctx context.Context, version *clusterapiv1beta1.ComponentVersion, 
    sshClient SSHClient) error {
    
    // 执行健康检查命令
    if version.Spec.HealthCheck != nil && version.Spec.HealthCheck.Command != "" {
        output, err := sshClient.ExecuteCommand(version.Spec.HealthCheck.Command)
        if err != nil {
            return fmt.Errorf("health check failed: %v", err)
        }
        
        // 验证输出
        if version.Spec.HealthCheck.ExpectedOutput != "" && 
           !strings.Contains(output, version.Spec.HealthCheck.ExpectedOutput) {
            return fmt.Errorf("health check output mismatch. Expected: %s, Got: %s", 
                version.Spec.HealthCheck.ExpectedOutput, output)
        }
    }
    
    return nil
}

// 渲染脚本模板
func (i *DefaultInstaller) renderScript(script string, version *clusterapiv1beta1.ComponentVersion, 
    config *clusterapiv1beta1.ComponentConfig) (string, error) {
    
    // 创建模板
    tmpl, err := template.New("component-script").Parse(script)
    if err != nil {
        return "", err
    }
    
    // 准备模板数据
    data := map[string]interface{}{
        "Version": version.Spec,
        "Config":  config.Spec.Config,
        "Source":  version.Spec.Source,
    }
    
    // 渲染模板
    var rendered strings.Builder
    if err := tmpl.Execute(&rendered, data); err != nil {
        return "", err
    }
    
    return rendered.String(), nil
}
```
## 四、Cluster API集成流程
### 4.1 集群创建流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                         集群创建流程                               │
└─────────────────────────────────────────────────────────────────────┘

1. 用户创建Cluster资源
   └──> 2. Cluster Controller创建基础设施资源
          └──> 3. 用户创建Machine/MachineDeployment资源
                 └──> 4. Machine Controller创建BootstrapConfig
                        └──> 5. Machine Controller创建Infrastructure Machine
                               └──> 6. Infrastructure Provider创建基础设施资源
                                      └──> 7. Infrastructure Provider安装节点组件
                                             └──> 8. Infrastructure Provider更新Machine状态
                                                    └──> 9. Bootstrap Provider执行引导
                                                           └──> 10. Machine加入集群
                                                                  └──> 11. Cluster就绪
```
### 4.2 组件升级流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                         组件升级流程                               │
└─────────────────────────────────────────────────────────────────────┘

1. 用户更新ComponentConfig资源（新版本）
   └──> 2. 用户更新Machine/MachineDeployment的组件配置引用
          └──> 3. Machine Controller触发Infrastructure Machine更新
                 └──> 4. Infrastructure Provider检测组件版本变更
                        └──> 5. Infrastructure Provider执行组件升级
                               └──> 6. Infrastructure Provider验证升级结果
                                      └──> 7. Infrastructure Provider更新Machine状态
                                             └──> 8. 升级完成
```
### 4.3 Machine扩容流程
```
┌─────────────────────────────────────────────────────────────────────┐
│                         Machine扩容流程                           │
└─────────────────────────────────────────────────────────────────────┘

1. 用户更新MachineDeployment的replicas字段
   └──> 2. Machine Deployment Controller创建新的Machine资源
          └──> 3. Machine Controller创建BootstrapConfig
                 └──> 4. Machine Controller创建Infrastructure Machine
                        └──> 5. Infrastructure Provider创建基础设施资源
                               └──> 6. Infrastructure Provider安装节点组件
                                      └──> 7. Infrastructure Provider更新Machine状态
                                             └──> 8. Bootstrap Provider执行引导
                                                    └──> 9. 新Machine加入集群
                                                           └──> 10. 扩容完成
```
## 五、组件版本管理
### 5.1 版本兼容性检查
```go
package component

import (
    "context"
    "fmt"
    "regexp"
    "strings"
    
    clusterapiv1beta1 "cluster.x-k8s.io/api/v1beta1"
    semver "github.com/Masterminds/semver/v3"
)

type VersionCompatibilityChecker struct{}

func NewVersionCompatibilityChecker() *VersionCompatibilityChecker {
    return &VersionCompatibilityChecker{}
}

func (c *VersionCompatibilityChecker) CheckCompatibility(ctx context.Context, 
    componentVersion *clusterapiv1beta1.ComponentVersion, 
    kubernetesVersion string, osInfo OSInfo) error {
    
    // 检查Kubernetes版本兼容性
    if err := c.checkKubernetesCompatibility(componentVersion, kubernetesVersion); err != nil {
        return err
    }
    
    // 检查OS兼容性
    if err := c.checkOSCompatibility(componentVersion, osInfo); err != nil {
        return err
    }
    
    return nil
}

func (c *VersionCompatibilityChecker) checkKubernetesCompatibility(
    componentVersion *clusterapiv1beta1.ComponentVersion, 
    kubernetesVersion string) error {
    
    // 解析Kubernetes版本
    k8sVerStr := extractSemver(kubernetesVersion)
    k8sVer, err := semver.NewVersion(k8sVerStr)
    if err != nil {
        return fmt.Errorf("failed to parse Kubernetes version %s: %v", kubernetesVersion, err)
    }
    
    // 检查最小版本
    if componentVersion.Spec.Compatibility.KubernetesVersions.Min != "" {
        minVer, err := semver.NewVersion(componentVersion.Spec.Compatibility.KubernetesVersions.Min)
        if err != nil {
            return fmt.Errorf("failed to parse min Kubernetes version %s: %v", 
                componentVersion.Spec.Compatibility.KubernetesVersions.Min, err)
        }
        
        if k8sVer.LessThan(minVer) {
            return fmt.Errorf("Kubernetes version %s is less than minimum required version %s", 
                k8sVerStr, componentVersion.Spec.Compatibility.KubernetesVersions.Min)
        }
    }
    
    // 检查最大版本
    if componentVersion.Spec.Compatibility.KubernetesVersions.Max != "" {
        maxVer, err := semver.NewVersion(componentVersion.Spec.Compatibility.KubernetesVersions.Max)
        if err != nil {
            return fmt.Errorf("failed to parse max Kubernetes version %s: %v", 
                componentVersion.Spec.Compatibility.KubernetesVersions.Max, err)
        }
        
        if k8sVer.GreaterThan(maxVer) {
            return fmt.Errorf("Kubernetes version %s is greater than maximum supported version %s", 
                k8sVerStr, componentVersion.Spec.Compatibility.KubernetesVersions.Max)
        }
    }
    
    return nil
}

func (c *VersionCompatibilityChecker) checkOSCompatibility(
    componentVersion *clusterapiv1beta1.ComponentVersion, 
    osInfo OSInfo) error {
    
    for _, osCompatibility := range componentVersion.Spec.Compatibility.OS {
        // 检查OS名称
        if osCompatibility.Name != osInfo.Name {
            continue
        }
        
        // 检查OS版本
        versionMatch := false
        for _, supportedVersion := range osCompatibility.Versions {
            if supportedVersion == osInfo.Version {
                versionMatch = true
                break
            }
        }
        
        if !versionMatch {
            continue
        }
        
        // 检查架构
        archMatch := false
        for _, supportedArch := range osCompatibility.Architectures {
            if supportedArch == osInfo.Architecture {
                archMatch = true
                break
            }
        }
        
        if archMatch {
            return nil
        }
    }
    
    return fmt.Errorf("OS %s %s %s is not compatible with component version %s", 
        osInfo.Name, osInfo.Version, osInfo.Architecture, componentVersion.Spec.Version)
}

// 从字符串中提取语义化版本
func extractSemver(version string) string {
    re := regexp.MustCompile(`v?(\d+\.\d+\.\d+)`)
    matches := re.FindStringSubmatch(version)
    if len(matches) > 1 {
        return matches[1]
    }
    return strings.TrimPrefix(version, "v")
}

type OSInfo struct {
    Name         string
    Version      string
    Architecture string
}
```
### 5.2 升级策略控制器
```go
package controller

import (
    "context"
    "fmt"
    "time"
    
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    clusterapiv1beta1 "cluster.x-k8s.io/api/v1beta1"
    "cluster.x-k8s.io/cluster-api/util"
)

type ComponentUpgradeReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=componentupgrades,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=componentupgrades/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=machines,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=cluster.x-k8s.io,resources=componentconfigs,verbs=get;list;watch

// ComponentUpgrade CRD
type ComponentUpgrade struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   ComponentUpgradeSpec   `json:"spec,omitempty"`
    Status ComponentUpgradeStatus `json:"status,omitempty"`
}

type ComponentUpgradeSpec struct {
    ClusterName string `json:"clusterName"`
    
    Components []ComponentUpgradeItem `json:"components"`
    
    // 升级策略
    Strategy UpgradeStrategy `json:"strategy"`
}

type ComponentUpgradeItem struct {
    ComponentType string `json:"componentType"`
    ConfigRef     corev1.ObjectReference `json:"configRef"`
}

type UpgradeStrategy struct {
    Type string `json:"type"` // rolling, batch, inplace
    
    // 滚动升级配置
    RollingUpdate *RollingUpdateStrategy `json:"rollingUpdate,omitempty"`
    
    // 批处理升级配置
    BatchUpdate *BatchUpdateStrategy `json:"batchUpdate,omitempty"`
    
    // 超时配置
    Timeout *metav1.Duration `json:"timeout,omitempty"`
}

type RollingUpdateStrategy struct {
    MaxUnavailable int32 `json:"maxUnavailable"`
    MaxSurge       int32 `json:"maxSurge"`
}

type BatchUpdateStrategy struct {
    BatchSize int32 `json:"batchSize"`
    Interval  metav1.Duration `json:"interval"`
}

type ComponentUpgradeStatus struct {
    Phase string `json:"phase"` // Pending, InProgress, Completed, Failed
    
    Progress ComponentUpgradeProgress `json:"progress"`
    
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

type ComponentUpgradeProgress struct {
    TotalNodes    int32 `json:"totalNodes"`
    UpgradedNodes int32 `json:"upgradedNodes"`
    FailedNodes   int32 `json:"failedNodes"`
}

func (r *ComponentUpgradeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    log.Info("Reconciling ComponentUpgrade")
    
    // 获取ComponentUpgrade
    upgrade := &ComponentUpgrade{}
    if err := r.Get(ctx, req.NamespacedName, upgrade); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 根据阶段执行操作
    switch upgrade.Status.Phase {
    case "", "Pending":
        return r.initiateUpgrade(ctx, upgrade)
    case "InProgress":
        return r.continueUpgrade(ctx, upgrade)
    case "Completed", "Failed":
        return ctrl.Result{}, nil
    default:
        return ctrl.Result{}, fmt.Errorf("unknown phase: %s", upgrade.Status.Phase)
    }
}

func (r *ComponentUpgradeReconciler) initiateUpgrade(ctx context.Context, upgrade *ComponentUpgrade) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 更新状态为InProgress
    upgrade.Status.Phase = "InProgress"
    upgrade.Status.Progress = ComponentUpgradeProgress{
        TotalNodes:    0,
        UpgradedNodes: 0,
        FailedNodes:   0,
    }
    
    // 获取集群的所有Machine
    machineList := &clusterapiv1beta1.MachineList{}
    if err := r.List(ctx, machineList, client.MatchingLabels{"cluster.x-k8s.io/cluster-name": upgrade.Spec.ClusterName}); err != nil {
        return ctrl.Result{}, err
    }
    
    upgrade.Status.Progress.TotalNodes = int32(len(machineList.Items))
    
    if err := r.Status().Update(ctx, upgrade); err != nil {
        return ctrl.Result{}, err
    }
    
    // 根据升级策略开始升级
    switch upgrade.Spec.Strategy.Type {
    case "rolling":
        return r.startRollingUpdate(ctx, upgrade, machineList.Items)
    case "batch":
        return r.startBatchUpdate(ctx, upgrade, machineList.Items)
    case "inplace":
        return r.startInplaceUpdate(ctx, upgrade, machineList.Items)
    default:
        return ctrl.Result{}, fmt.Errorf("unsupported upgrade strategy: %s", upgrade.Spec.Strategy.Type)
    }
}

func (r *ComponentUpgradeReconciler) startRollingUpdate(ctx context.Context, upgrade *ComponentUpgrade, 
    machines []clusterapiv1beta1.Machine) (ctrl.Result, error) {
    
    log := ctrl.LoggerFrom(ctx)
    
    // 计算可以同时升级的节点数
    maxUnavailable := upgrade.Spec.Strategy.RollingUpdate.MaxUnavailable
    if maxUnavailable <= 0 {
        maxUnavailable = 1
    }
    
    // 选择需要升级的节点
    var nodesToUpgrade []clusterapiv1beta1.Machine
    for _, machine := range machines {
        // 跳过已经升级的节点
        if hasBeenUpgraded(&machine, upgrade) {
            continue
        }
        
        nodesToUpgrade = append(nodesToUpgrade, machine)
        if len(nodesToUpgrade) >= int(maxUnavailable) {
            break
        }
    }
    
    // 升级节点
    for _, machine := range nodesToUpgrade {
        if err := r.upgradeMachine(ctx, &machine, upgrade); err != nil {
            log.Error(err, "Failed to upgrade machine", "machine", machine.Name)
            upgrade.Status.Progress.FailedNodes++
            continue
        }
        
        upgrade.Status.Progress.UpgradedNodes++
    }
    
    // 更新状态
    if err := r.Status().Update(ctx, upgrade); err != nil {
        return ctrl.Result{}, err
    }
    
    // 检查是否完成
    if upgrade.Status.Progress.UpgradedNodes+upgrade.Status.Progress.FailedNodes >= upgrade.Status.Progress.TotalNodes {
        if upgrade.Status.Progress.FailedNodes == 0 {
            upgrade.Status.Phase = "Completed"
        } else {
            upgrade.Status.Phase = "Failed"
        }
        
        if err := r.Status().Update(ctx, upgrade); err != nil {
            return ctrl.Result{}, err
        }
        
        return ctrl.Result{}, nil
    }
    
    // 继续升级
    return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}

func (r *ComponentUpgradeReconciler) upgradeMachine(ctx context.Context, machine *clusterapiv1beta1.Machine, 
    upgrade *ComponentUpgrade) error {
    
    log := ctrl.LoggerFrom(ctx)
    
    // 创建新的Machine组件配置
    newComponents := make([]clusterapiv1beta1.MachineComponent, len(machine.Spec.Components))
    copy(newComponents, machine.Spec.Components)
    
    // 更新组件配置引用
    for i, component := range newComponents {
        for _, upgradeItem := range upgrade.Spec.Components {
            if component.Name == upgradeItem.ComponentType {
                newComponents[i].ConfigRef = upgradeItem.ConfigRef
                break
            }
        }
    }
    
    // 更新Machine
    machine.Spec.Components = newComponents
    
    if err := r.Update(ctx, machine); err != nil {
        return fmt.Errorf("failed to update machine %s: %v", machine.Name, err)
    }
    
    log.Info("Machine updated for upgrade", "machine", machine.Name)
    return nil
}
```
## 六、使用示例
### 6.1 创建业务集群
```yaml
# 1. 创建Cluster资源
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  infrastructureRef:
    kind: Metal3Cluster
    name: my-cluster
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
---

# 2. 创建ComponentConfig资源
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentConfig
metadata:
  name: container-runtime-config
spec:
  componentType: container-runtime
  config:
    containerd:
      version: 1.7.11
      config:
        dataDir: /var/lib/containerd
        systemdCgroup: true
---

apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentConfig
metadata:
  name: kubelet-config
spec:
  componentType: kubelet
  config:
    kubelet:
      version: 1.28.2
      config:
        maxPods: 110
        systemReserved:
          memory: 1Gi
          cpu: 1000m
---

apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentConfig
metadata:
  name: cni-config
spec:
  componentType: cni
  config:
    calico:
      version: 3.26.4
      config:
        ipam:
          type: host-local
          subnet: 192.168.0.0/24
---

# 3. 创建ComponentVersion资源
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.11
spec:
  componentType: container-runtime
  name: containerd
  version: 1.7.11
  source:
    type: HTTP
    url: https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-amd64.tar.gz
    checksum: sha256:abc123...
  # 安装脚本等...
---

# 4. 创建MachineTemplate资源
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineTemplate
metadata:
  name: control-plane-template
spec:
  template:
    spec:
      clusterName: my-cluster
      version: v1.28.2
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: control-plane-template
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: Metal3MachineTemplate
        name: control-plane-template
      components:
        - name: container-runtime
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: container-runtime-config
        - name: kubelet
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: kubelet-config
        - name: cni
          configRef:
            apiVersion: cluster.x-k8s.io/v1beta1
            kind: ComponentConfig
            name: cni-config
---

# 5. 创建KubeadmControlPlane资源
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
spec:
  replicas: 3
  version: v1.28.2
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: Metal3MachineTemplate
      name: control-plane-template
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        certSANs:
          - localhost
          - 127.0.0.1
      controlPlaneEndpoint:
        host: my-cluster-control-plane
        port: 6443
    initConfiguration:
      nodeRegistration:
        criSocket: /run/containerd/containerd.sock
        kubeletExtraArgs:
          cgroup-driver: systemd
    joinConfiguration:
      controlPlane: {}
      nodeRegistration:
        criSocket: /run/containerd/containerd.sock
        kubeletExtraArgs:
          cgroup-driver: systemd
```
### 6.2 升级组件
```yaml
# 1. 创建新版本的ComponentConfig
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentConfig
metadata:
  name: container-runtime-config-v1.7.12
spec:
  componentType: container-runtime
  config:
    containerd:
      version: 1.7.12
      config:
        dataDir: /var/lib/containerd
        systemdCgroup: true
---

# 2. 创建新版本的ComponentVersion
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.12
spec:
  componentType: container-runtime
  name: containerd
  version: 1.7.12
  source:
    type: HTTP
    url: https://github.com/containerd/containerd/releases/download/v1.7.12/containerd-1.7.12-linux-amd64.tar.gz
    checksum: sha256:def456...
  # 安装脚本等...
---

# 3. 创建ComponentUpgrade资源
apiVersion: cluster.x-k8s.io/v1beta1
kind: ComponentUpgrade
metadata:
  name: upgrade-containerd
spec:
  clusterName: my-cluster
  components:
    - componentType: container-runtime
      configRef:
        apiVersion: cluster.x-k8s.io/v1beta1
        kind: ComponentConfig
        name: container-runtime-config-v1.7.12
  strategy:
    type: rolling
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
    timeout:
      duration: 2h
```
## 七、方案优势
### 7.1 Cluster API原生集成
- **无缝集成**：扩展Cluster API资源，而非独立并行
- **统一控制平面**：通过Cluster API管理所有集群资源
- **标准化流程**：遵循Cluster API的声明式管理和状态协调模式
### 7.2 灵活的组件管理
- **声明式配置**：使用CRD定义组件配置和版本
- **多组件支持**：支持container-runtime、kubelet、CNI、CSI等
- **版本兼容性**：自动检查组件与Kubernetes、OS的兼容性
### 7.3 强大的升级能力
- **多种升级策略**：支持滚动升级、批处理升级、原地升级
- **自动化流程**：从版本检测到升级验证的全自动化
- **失败回滚**：升级失败时自动回滚到安全状态
### 7.4 高度可扩展
- **插件化架构**：支持自定义组件类型和安装脚本
- **多Infrastructure Provider**：支持各种基础设施类型
- **社区兼容**：遵循Cluster API的扩展规范
## 八、总结
本方案将节点组件管理与Cluster API深度集成，实现了：
1. **统一的集群生命周期管理**：从基础设施到应用层的完整管理
2. **灵活的组件部署方式**：支持多种组件和版本的声明式配置
3. **自动化的升级流程**：基于策略的组件升级和健康检查
4. **高度的可扩展性**：支持自定义组件和Infrastructure Provider

该方案完全遵循Cluster API的设计理念，提供了一个标准、可扩展的节点组件管理框架，适用于各种规模和类型的Kubernetes集群部署。

     
# 集群管理解决方案版本包设计
## 一、版本包概述
设计一个完整的版本包结构，支持引导集群、管理集群和业务集群的全生命周期管理与自动化升级。版本包采用分层设计，包含解决方案版本元数据、各层级集群的组件定义、升级策略和兼容性矩阵。
### 1.1 版本包核心概念
- **解决方案版本(Solution Version)**：全局唯一的版本标识，如 `v1.2.0`，包含引导集群、管理集群和业务集群的完整定义
- **层级版本(Layer Version)**：各层级集群(引导/管理/业务)的版本，如 `bootstrap-v1.2.0`、`management-v1.2.0`
- **组件版本(Component Version)**：集群内各组件的版本，如 `containerd-v1.7.11`、`kubelet-v1.28.2`
- **升级路径(Upgrade Path)**：从旧版本到新版本的升级流程定义
- **兼容性矩阵(Compatibility Matrix)**：定义版本间的兼容性关系
## 二、版本包结构
```
cluster-manager-solution-v1.2.0/
├── metadata.yaml                     # 解决方案版本元数据
├── CHANGELOG.md                      # 变更日志
├── README.md                         # 版本包说明文档
├── LICENSE                           # 许可证
├── checksum.sha256                   # 文件校验和
├── components/                       # 组件定义目录
│   ├── bootstrap/                    # 引导集群组件
│   │   ├── k3s/                      # K3s组件
│   │   │   ├── v1.27.0/              # 版本目录
│   │   │   │   ├── component.yaml    # 组件元数据
│   │   │   │   ├── install.sh        # 安装脚本
│   │   │   │   ├── upgrade.sh        # 升级脚本
│   │   │   │   └── files/            # 组件文件
│   │   ├── kind/                     # Kind组件
│   │   │   └── v0.17.0/              # 版本目录
│   │   └── minikube/                 # Minikube组件
│   │       └── v1.30.0/              # 版本目录
│   ├── management/                   # 管理集群组件
│   │   ├── container-runtime/        # 容器运行时组件
│   │   │   ├── containerd/           # Containerd组件
│   │   │   │   └── v1.7.11/          # 版本目录
│   │   ├── kubelet/                  # Kubelet组件
│   │   │   └── v1.28.2/              # 版本目录
│   │   ├── cni/                      # CNI组件
│   │   │   └── calico/               # Calico组件
│   │   │       └── v3.26.4/          # 版本目录
│   │   └── capi/                     # Cluster API组件
│   │       └── v1.5.2/               # 版本目录
│   └── workload/                     # 业务集群组件
│       ├── container-runtime/        # 容器运行时组件
│       │   └── containerd/           # Containerd组件
│       │       └── v1.7.11/          # 版本目录
│       ├── kubelet/                  # Kubelet组件
│       │   └── v1.28.2/              # 版本目录
│       ├── cni/                      # CNI组件
│       │   └── calico/               # Calico组件
│       │       └── v3.26.4/          # 版本目录
│       └── csi/                      # CSI组件
│           └── hostpath/             # HostPath CSI组件
│               └── v1.10.1/          # 版本目录
├── templates/                        # 资源模板目录
│   ├── bootstrap/                    # 引导集群模板
│   │   ├── k3s-config.yaml.tmpl      # K3s配置模板
│   │   └── kind-config.yaml.tmpl     # Kind配置模板
│   ├── management/                   # 管理集群模板
│   │   ├── cluster.yaml.tmpl         # Cluster资源模板
│   │   ├── component-config.yaml.tmpl # ComponentConfig资源模板
│   │   └── machine-template.yaml.tmpl # MachineTemplate资源模板
│   └── workload/                     # 业务集群模板
│       ├── cluster.yaml.tmpl         # Cluster资源模板
│       └── machine-deployment.yaml.tmpl # MachineDeployment资源模板
├── upgrade/                          # 升级定义目录
│   ├── solution/                     # 解决方案升级路径
│   │   ├── v1.1.0_to_v1.2.0/         # 从v1.1.0到v1.2.0的升级
│   │   │   ├── metadata.yaml         # 升级元数据
│   │   │   ├── pre-check.sh          # 前置检查脚本
│   │   │   ├── upgrade.sh            # 升级脚本
│   │   │   └── rollback.sh           # 回滚脚本
│   ├── bootstrap/                    # 引导集群升级路径
│   │   └── bootstrap-v1.1.0_to_v1.2.0/ # 引导集群升级
│   ├── management/                   # 管理集群升级路径
│   │   └── management-v1.1.0_to_v1.2.0/ # 管理集群升级
│   └── workload/                     # 业务集群升级路径
│       └── workload-v1.1.0_to_v1.2.0/ # 业务集群升级
├── compatibility/                    # 兼容性矩阵目录
│   ├── solution.yaml                 # 解决方案兼容性
│   ├── bootstrap.yaml                # 引导集群兼容性
│   ├── management.yaml               # 管理集群兼容性
│   └── workload.yaml                 # 业务集群兼容性
├── tools/                            # 工具目录
│   ├── cluster-manager-cli           # 命令行工具
│   ├── upgrade-agent                 # 升级代理
│   └── health-checker                # 健康检查工具
└── manifests/                        # 部署清单目录
    ├── bootstrap/                    # 引导集群部署清单
    ├── management/                   # 管理集群部署清单
    └── workload/                     # 业务集群部署清单
```
## 三、核心文件内容设计
### 3.1 解决方案元数据 (metadata.yaml)
```yaml
apiVersion: cluster-manager.io/v1alpha1
kind: SolutionVersion
metadata:
  name: cluster-manager-solution-v1.2.0
spec:
  # 解决方案版本信息
  version: v1.2.0
  releaseDate: "2024-01-15T00:00:00Z"
  description: "Cluster Manager Solution v1.2.0"
  releaseNotes: "CHANGELOG.md"
  
  # 层级集群版本映射
  layers:
    bootstrap:
      version: bootstrap-v1.2.0
      components:
        - name: k3s
          version: v1.27.0
        - name: kind
          version: v0.17.0
        - name: minikube
          version: v1.30.0
    
    management:
      version: management-v1.2.0
      components:
        - name: container-runtime
          type: containerd
          version: v1.7.11
        - name: kubelet
          version: v1.28.2
        - name: cni
          type: calico
          version: v3.26.4
        - name: capi
          version: v1.5.2
    
    workload:
      version: workload-v1.2.0
      components:
        - name: container-runtime
          type: containerd
          version: v1.7.11
        - name: kubelet
          version: v1.28.2
        - name: cni
          type: calico
          version: v3.26.4
        - name: csi
          type: hostpath
          version: v1.10.1
  
  # 依赖关系
  dependencies:
    - name: kubectl
      version: ">=1.26.0"
    - name: helm
      version: ">=3.9.0"
  
  # 支持的操作系统
  supportedOS:
    - name: ubuntu
      versions: ["20.04", "22.04"]
      architectures: ["amd64", "arm64"]
    - name: centos
      versions: ["7", "8", "9"]
      architectures: ["amd64"]
  
  # 升级路径
  upgradePaths:
    - fromVersion: v1.1.0
      toVersion: v1.2.0
      path: upgrade/solution/v1.1.0_to_v1.2.0/
    - fromVersion: v1.0.0
      toVersion: v1.2.0
      path: upgrade/solution/v1.0.0_to_v1.2.0/
      requires:
        - v1.0.0_to_v1.1.0
  
  # 校验信息
  checksums:
    - file: components/bootstrap/k3s/v1.27.0/component.yaml
      sha256: "abc123..."
    - file: components/management/container-runtime/containerd/v1.7.11/component.yaml
      sha256: "def456..."
```
### 3.2 组件元数据 (components/management/container-runtime/containerd/v1.7.11/component.yaml)
```yaml
apiVersion: cluster-manager.io/v1alpha1
kind: ComponentVersion
metadata:
  name: containerd-v1.7.11
spec:
  componentType: container-runtime
  name: containerd
  version: v1.7.11
  
  # 组件来源
  source:
    type: HTTP
    url: https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-amd64.tar.gz
    checksum: sha256:abc123...
  
  # 安装配置
  installation:
    script: install.sh
    dependencies:
      - name: systemd
      - name: tar
      - name: curl
  
  # 升级配置
  upgrade:
    script: upgrade.sh
    dependencies:
      - name: systemd
      - name: tar
      - name: curl
  
  # 健康检查
  healthCheck:
    command: "systemctl is-active containerd"
    expectedOutput: "active"
    timeout: 30s
  
  # 兼容性信息
  compatibility:
    kubernetesVersions:
      min: "1.26.0"
      max: "1.30.0"
    os:
      - name: ubuntu
        versions: ["20.04", "22.04"]
        architectures: ["amd64", "arm64"]
      - name: centos
        versions: ["7", "8", "9"]
        architectures: ["amd64"]
  
  # 配置模板
  configTemplate: config.toml.tmpl
```
### 3.3 升级路径元数据 (upgrade/solution/v1.1.0_to_v1.2.0/metadata.yaml)
```yaml
apiVersion: cluster-manager.io/v1alpha1
kind: UpgradePath
metadata:
  name: solution-v1.1.0_to_v1.2.0
spec:
  fromVersion: v1.1.0
  toVersion: v1.2.0
  type: solution
  
  # 升级顺序
  upgradeOrder:
    - layer: bootstrap
      path: upgrade/bootstrap/bootstrap-v1.1.0_to_v1.2.0/
    - layer: management
      path: upgrade/management/management-v1.1.0_to_v1.2.0/
    - layer: workload
      path: upgrade/workload/workload-v1.1.0_to_v1.2.0/
  
  # 前置检查
  preCheck:
    description: "Pre-upgrade checks for solution v1.1.0 to v1.2.0"
    script: pre-check.sh
    timeout: 5m
  
  # 升级配置
  upgrade:
    description: "Upgrade script for solution v1.1.0 to v1.2.0"
    script: upgrade.sh
    timeout: 2h
  
  # 回滚配置
  rollback:
    description: "Rollback script for solution v1.1.0 to v1.2.0"
    script: rollback.sh
    timeout: 1h
  
  # 组件变更
  componentChanges:
    - layer: management
      component: container-runtime
      fromVersion: v1.7.10
      toVersion: v1.7.11
      changeType: patch
      description: "Fix security vulnerabilities"
    
    - layer: management
      component: kubelet
      fromVersion: v1.28.1
      toVersion: v1.28.2
      changeType: patch
      description: "Bug fixes and performance improvements"
    
    - layer: workload
      component: csi
      fromVersion: v1.10.0
      toVersion: v1.10.1
      changeType: patch
      description: "Fix volume attachment issues"
  
  # 兼容性信息
  compatibility:
    requiredTools:
      - name: kubectl
        version: ">=1.26.0"
      - name: helm
        version: ">=3.9.0"
    
    supportedOS:
      - name: ubuntu
        versions: ["20.04", "22.04"]
      - name: centos
        versions: ["7", "8", "9"]
```
### 3.4 兼容性矩阵 (compatibility/solution.yaml)
```yaml
apiVersion: cluster-manager.io/v1alpha1
kind: CompatibilityMatrix
metadata:
  name: solution-compatibility
spec:
  # 解决方案版本兼容性
  solutionVersions:
    - version: v1.2.0
      compatibleWith:
        - version: v1.1.0
          upgradePath: upgrade/solution/v1.1.0_to_v1.2.0/
        - version: v1.0.0
          upgradePath: upgrade/solution/v1.0.0_to_v1.2.0/
          requires:
            - v1.0.0_to_v1.1.0
    
    - version: v1.1.0
      compatibleWith:
        - version: v1.0.0
          upgradePath: upgrade/solution/v1.0.0_to_v1.1.0/
  
  # 层级集群版本兼容性
  layerCompatibility:
    bootstrap:
      - version: bootstrap-v1.2.0
        compatibleWith:
          - bootstrap-v1.1.0
    
    management:
      - version: management-v1.2.0
        compatibleWith:
          - management-v1.1.0
    
    workload:
      - version: workload-v1.2.0
        compatibleWith:
          - workload-v1.1.0
          - workload-v1.0.0
  
  # 组件版本兼容性
  componentCompatibility:
    container-runtime:
      - version: v1.7.11
        compatibleWith:
          - v1.7.10
          - v1.7.9
    
    kubelet:
      - version: v1.28.2
        compatibleWith:
          - v1.28.1
          - v1.28.0
          - v1.27.5
```
## 四、自动化升级流程
### 4.1 升级流程概述
```
┌─────────────────────────────────────────────────────────────────────┐
│                         自动化升级流程                             │
└─────────────────────────────────────────────────────────────────────┘

1. 版本检测与兼容性检查
   └──> 2. 下载并验证版本包
          └──> 3. 执行前置检查
                 └──> 4. 备份当前配置
                        └──> 5. 按顺序升级各层级集群
                               └──> 6. 升级引导集群
                                      └──> 7. 升级管理集群
                                             └──> 8. 升级业务集群
                                                    └──> 9. 执行健康检查
                                                           └──> 10. 更新版本元数据
                                                                  └──> 11. 清理临时文件
```
### 4.2 升级命令示例
```bash
# 检查当前版本和可用升级
cluster-manager upgrade check

# 输出：
# Current solution version: v1.1.0
# Available upgrades:
#   - v1.2.0 (security fixes, bug fixes)

# 执行升级
cluster-manager upgrade apply v1.2.0

# 输出：
# ℹ️  Downloading version package cluster-manager-solution-v1.2.0...
# ✅ Version package downloaded and verified
# ℹ️  Running pre-upgrade checks...
# ✅ Pre-upgrade checks passed
# ℹ️  Backing up current configuration...
# ✅ Configuration backed up to /var/lib/cluster-manager/backups/v1.1.0-20240115-143022
# ℹ️  Starting upgrade process...
#   - Upgrading bootstrap cluster (bootstrap-v1.1.0 → bootstrap-v1.2.0)...
#     - Upgrading k3s from v1.27.0 to v1.27.0... (no change)
#   ✅ Bootstrap cluster upgraded successfully
#   - Upgrading management cluster (management-v1.1.0 → management-v1.2.0)...
#     - Upgrading containerd from v1.7.10 to v1.7.11...
#     - Upgrading kubelet from v1.28.1 to v1.28.2...
#   ✅ Management cluster upgraded successfully
#   - Upgrading workload clusters...
#     - Upgrading cluster-1 (workload-v1.1.0 → workload-v1.2.0)...
#       - Upgrading containerd from v1.7.10 to v1.7.11...
#       - Upgrading kubelet from v1.28.1 to v1.28.2...
#       - Upgrading csi from v1.10.0 to v1.10.1...
#     ✅ Cluster-1 upgraded successfully
#   ✅ All workload clusters upgraded successfully
# ℹ️  Running post-upgrade health checks...
# ✅ All health checks passed
# ℹ️  Updating version metadata...
# ✅ Version metadata updated
# ℹ️  Cleaning up temporary files...
# ✅ Cleanup completed
# 
# 🎉 Upgrade completed successfully!
# Current solution version: v1.2.0
```
### 4.3 升级工具实现
```go
package cmd

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    "time"
    
    "github.com/spf13/cobra"
    
    "cluster-manager.io/pkg/cli"
    "cluster-manager.io/pkg/upgrade"
    "cluster-manager.io/pkg/version"
)

var upgradeCmd = &cobra.Command{
    Use:   "upgrade",
    Short: "Upgrade the cluster manager solution",
    Long:  "Upgrade the cluster manager solution to a newer version",
}

var checkCmd = &cobra.Command{
    Use:   "check",
    Short: "Check for available upgrades",
    Long:  "Check for available upgrades for the current solution version",
    RunE:  runUpgradeCheck,
}

var applyCmd = &cobra.Command{
    Use:   "apply VERSION",
    Short: "Apply an upgrade",
    Long:  "Apply an upgrade to the specified version",
    Args:  cobra.ExactArgs(1),
    RunE:  runUpgradeApply,
}

func init() {
    rootCmd.AddCommand(upgradeCmd)
    upgradeCmd.AddCommand(checkCmd)
    upgradeCmd.AddCommand(applyCmd)
    
    // 升级相关参数
    applyCmd.Flags().BoolP("dry-run", "d", false, "Dry run upgrade")
    applyCmd.Flags().BoolP("force", "f", false, "Force upgrade without prompts")
    applyCmd.Flags().StringP("backup-dir", "b", "", "Backup directory")
    applyCmd.Flags().BoolP("skip-pre-check", "s", false, "Skip pre-upgrade checks")
}

func runUpgradeCheck(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    
    // 获取当前版本
    currentVersion, err := version.GetCurrentSolutionVersion(ctx)
    if err != nil {
        return fmt.Errorf("get current version failed: %v", err)
    }
    
    fmt.Printf("Current solution version: %s\n", currentVersion)
    
    // 检查可用升级
    availableUpgrades, err := upgrade.CheckAvailableUpgrades(ctx, currentVersion)
    if err != nil {
        return fmt.Errorf("check available upgrades failed: %v", err)
    }
    
    if len(availableUpgrades) == 0 {
        fmt.Println("No available upgrades")
        return nil
    }
    
    fmt.Println("Available upgrades:")
    for _, upgrade := range availableUpgrades {
        fmt.Printf("  - %s (%s)\n", upgrade.Version, upgrade.Description)
    }
    
    return nil
}

func runUpgradeApply(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    
    // 解析参数
    targetVersion := args[0]
    dryRun, _ := cmd.Flags().GetBool("dry-run")
    force, _ := cmd.Flags().GetBool("force")
    backupDir, _ := cmd.Flags().GetString("backup-dir")
    skipPreCheck, _ := cmd.Flags().GetBool("skip-pre-check")
    
    // 获取当前版本
    currentVersion, err := version.GetCurrentSolutionVersion(ctx)
    if err != nil {
        return fmt.Errorf("get current version failed: %v", err)
    }
    
    fmt.Printf("Current solution version: %s\n", currentVersion)
    fmt.Printf("Target solution version: %s\n", targetVersion)
    
    // 确认升级
    if !force && !dryRun {
        if !cli.Confirm("Are you sure you want to proceed with the upgrade?") {
            return fmt.Errorf("upgrade cancelled by user")
        }
    }
    
    // 获取升级路径
    upgradePath, err := upgrade.GetUpgradePath(ctx, currentVersion, targetVersion)
    if err != nil {
        return fmt.Errorf("get upgrade path failed: %v", err)
    }
    
    // 下载版本包
    fmt.Printf("ℹ️  Downloading version package cluster-manager-solution-%s...\n", targetVersion)
    versionPackage, err := upgrade.DownloadVersionPackage(ctx, targetVersion)
    if err != nil {
        return fmt.Errorf("download version package failed: %v", err)
    }
    fmt.Printf("✅ Version package downloaded and verified\n")
    
    // 执行前置检查
    if !skipPreCheck && !dryRun {
        fmt.Println("ℹ️  Running pre-upgrade checks...")
        if err := upgrade.RunPreCheck(ctx, upgradePath, versionPackage); err != nil {
            return fmt.Errorf("pre-upgrade check failed: %v", err)
        }
        fmt.Println("✅ Pre-upgrade checks passed")
    }
    
    // 备份配置
    if !dryRun {
        fmt.Println("ℹ️  Backing up current configuration...")
        backupPath, err := upgrade.BackupConfiguration(ctx, currentVersion, backupDir)
        if err != nil {
            return fmt.Errorf("backup configuration failed: %v", err)
        }
        fmt.Printf("✅ Configuration backed up to %s\n", backupPath)
    }
    
    // 执行升级
    fmt.Println("ℹ️  Starting upgrade process...")
    if err := upgrade.ExecuteUpgrade(ctx, upgradePath, versionPackage, dryRun); err != nil {
        // 升级失败，尝试回滚
        if !dryRun {
            fmt.Printf("❌ Upgrade failed: %v\n", err)
            fmt.Println("ℹ️  Attempting rollback...")
            if rollbackErr := upgrade.RollbackUpgrade(ctx, currentVersion, versionPackage); rollbackErr != nil {
                return fmt.Errorf("upgrade failed and rollback also failed: upgrade error: %v, rollback error: %v", err, rollbackErr)
            }
            fmt.Println("✅ Rollback completed")
        }
        return fmt.Errorf("upgrade failed: %v", err)
    }
    
    // 执行健康检查
    if !dryRun {
        fmt.Println("ℹ️  Running post-upgrade health checks...")
        if err := upgrade.RunHealthChecks(ctx, upgradePath, versionPackage); err != nil {
            return fmt.Errorf("health check failed: %v", err)
        }
        fmt.Println("✅ All health checks passed")
    }
    
    // 更新版本元数据
    if !dryRun {
        fmt.Println("ℹ️  Updating version metadata...")
        if err := version.UpdateSolutionVersion(ctx, targetVersion); err != nil {
            return fmt.Errorf("update version metadata failed: %v", err)
        }
        fmt.Println("✅ Version metadata updated")
    }
    
    // 清理临时文件
    if !dryRun {
        fmt.Println("ℹ️  Cleaning up temporary files...")
        if err := upgrade.CleanupUpgrade(ctx, versionPackage); err != nil {
            fmt.Printf("⚠️  Cleanup failed: %v\n", err)
        } else {
            fmt.Println("✅ Cleanup completed")
        }
    }
    
    fmt.Println("\n🎉 Upgrade completed successfully!")
    fmt.Printf("Current solution version: %s\n", targetVersion)
    
    return nil
}
```
## 五、版本管理最佳实践
### 5.1 版本命名规范
- **解决方案版本**：采用语义化版本，如 `v1.2.0`
- **层级版本**：采用 `{layer}-{version}` 格式，如 `management-v1.2.0`
- **组件版本**：采用 `{component}-{version}` 格式，如 `containerd-v1.7.11`
### 5.2 版本兼容性策略
1. **解决方案版本**：
   - 主版本号(MAJOR)变更：不兼容的API变更
   - 次版本号(MINOR)变更：向下兼容的功能新增
   - 修订号(PATCH)变更：向下兼容的bug修复
2. **层级版本**：
   - 与解决方案版本保持一致
   - 支持跨小版本升级
3. **组件版本**：
   - 支持独立升级
   - 必须与Kubernetes版本兼容
### 5.3 升级安全策略
1. **备份机制**：
   - 升级前自动备份配置
   - 支持手动备份和恢复
2. **回滚机制**：
   - 升级失败自动回滚
   - 支持手动回滚到指定版本
3. **健康检查**：
   - 升级前的前置检查
   - 升级过程中的实时检查
   - 升级后的完整性检查
4. **灰度升级**：
   - 支持按比例升级业务集群
   - 支持暂停和继续升级
## 六、总结
本版本包设计提供了一个完整的集群管理解决方案版本管理和自动化升级框架：
1. **完整的版本包结构**：包含解决方案、各层级集群和组件的定义
2. **丰富的元数据**：描述版本信息、组件定义、升级路径和兼容性
3. **自动化升级流程**：从版本检测到升级完成的全自动化流程
4. **安全可靠的升级机制**：包含备份、回滚和健康检查
5. **灵活的扩展性**：支持自定义组件和升级策略

该设计解决了跨集群层级的版本管理和自动化升级问题，为用户提供了一个统一、安全、可靠的集群管理解决方案。

# `upgrade.ExecuteUpgrade` 实现思路
## 一、核心功能概述
`upgrade.ExecuteUpgrade` 是集群管理解决方案升级流程的核心函数，负责协调执行从引导集群、管理集群到业务集群的全流程升级。该函数需要处理版本兼容性验证、组件升级顺序、错误回滚和健康检查等关键任务。
## 二、整体设计思路
### 2.1 架构设计
```
┌─────────────────────────────────────────────────────────────────────┐
│                      ExecuteUpgrade 架构                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐
│  ExecuteUpgrade         │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  UpgradeContext         │     │  VersionPackage         │
└─────────────────────────┘     └─────────────────────────┘
            │                           │
            ▼                           ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  UpgradeManager         │────▶│  UpgradePath            │
└─────────────────────────┘     └─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│  LayerUpgradeExecutors  │
│  ┌─────────────────────┐│
│  │  BootstrapExecutor  ││
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │  ManagementExecutor ││
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │  WorkloadExecutor   ││
│  └─────────────────────┘│
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│  ComponentExecutor      │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│  UpgradeSteps           │
│  ┌─────────────────────┐│
│  │  PreCheckStep       ││
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │  BackupStep         ││
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │  InstallStep        ││
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │  VerifyStep         ││
│  └─────────────────────┘│
└─────────────────────────┘
```
### 2.2 关键设计原则
1. **分层执行**：按引导集群 → 管理集群 → 业务集群的顺序执行升级
2. **组件级隔离**：每个组件的升级独立封装，便于维护和扩展
3. **幂等性**：所有升级操作均可重复执行，不会产生副作用
4. **可回滚**：支持在升级失败时回滚到安全状态
5. **可观测性**：详细的日志和指标收集，便于问题定位
6. **Dry-Run支持**：允许在不实际修改系统的情况下测试升级流程
## 三、详细实现思路
### 3.1 函数签名与参数
```go
func ExecuteUpgrade(ctx context.Context, upgradePath *UpgradePath, 
    versionPackage *VersionPackage, dryRun bool) error {
    // 实现代码
}
```
**参数说明**：
- `ctx`：上下文，用于控制超时和取消
- `upgradePath`：升级路径定义，包含从旧版本到新版本的详细升级步骤
- `versionPackage`：版本包，包含所有组件的安装包、脚本和配置
- `dryRun`：是否为模拟运行，不实际修改系统
### 3.2 核心执行流程
```go
func ExecuteUpgrade(ctx context.Context, upgradePath *UpgradePath, 
    versionPackage *VersionPackage, dryRun bool) error {
    
    // 1. 初始化升级上下文
    upgradeCtx, err := initializeUpgradeContext(ctx, upgradePath, versionPackage, dryRun)
    if err != nil {
        return fmt.Errorf("initialize upgrade context failed: %v", err)
    }
    
    // 2. 加载升级管理器
    upgradeManager := NewUpgradeManager(upgradeCtx)
    
    // 3. 按顺序执行各层级集群升级
    for _, layerUpgrade := range upgradePath.Spec.UpgradeOrder {
        // 执行层级升级
        if err := upgradeManager.ExecuteLayerUpgrade(layerUpgrade); err != nil {
            // 升级失败，执行回滚
            if rollbackErr := upgradeManager.RollbackLayerUpgrade(layerUpgrade); rollbackErr != nil {
                return fmt.Errorf("upgrade %s failed and rollback also failed: upgrade error: %v, rollback error: %v", 
                    layerUpgrade.Layer, err, rollbackErr)
            }
            return fmt.Errorf("upgrade %s failed and rolled back: %v", layerUpgrade.Layer, err)
        }
    }
    
    // 4. 升级完成，执行最终验证
    if err := upgradeManager.FinalizeUpgrade(); err != nil {
        return fmt.Errorf("finalize upgrade failed: %v", err)
    }
    
    return nil
}
```
### 3.3 升级上下文初始化
```go
func initializeUpgradeContext(ctx context.Context, upgradePath *UpgradePath, 
    versionPackage *VersionPackage, dryRun bool) (*UpgradeContext, error) {
    
    // 创建升级上下文
    upgradeCtx := &UpgradeContext{
        Context:        ctx,
        UpgradePath:    upgradePath,
        VersionPackage: versionPackage,
        DryRun:         dryRun,
        StartTime:      time.Now(),
        Steps:          make(map[string]*UpgradeStep),
        Logger:         NewUpgradeLogger(upgradePath.Spec.ToVersion),
        Metrics:        NewUpgradeMetrics(),
    }
    
    // 加载配置
    config, err := loadUpgradeConfig()
    if err != nil {
        return nil, err
    }
    upgradeCtx.Config = config
    
    // 初始化状态存储
    upgradeCtx.StateStore, err = NewStateStore(config.StateStorePath)
    if err != nil {
        return nil, err
    }
    
    // 记录开始状态
    if err := upgradeCtx.StateStore.SaveInitialState(); err != nil {
        return nil, err
    }
    
    return upgradeCtx, nil
}
```
### 3.4 升级管理器实现
```go
type UpgradeManager struct {
    ctx     *UpgradeContext
    executors map[string]LayerUpgradeExecutor
}

func NewUpgradeManager(ctx *UpgradeContext) *UpgradeManager {
    return &UpgradeManager{
        ctx:     ctx,
        executors: make(map[string]LayerUpgradeExecutor),
    }
}

func (m *UpgradeManager) ExecuteLayerUpgrade(layerUpgrade LayerUpgrade) error {
    m.ctx.Logger.Info("Starting layer upgrade", "layer", layerUpgrade.Layer)
    
    // 获取或创建层级执行器
    executor, err := m.getLayerExecutor(layerUpgrade.Layer)
    if err != nil {
        return fmt.Errorf("get layer executor failed: %v", err)
    }
    
    // 加载层级升级配置
    layerUpgradeConfig, err := m.ctx.VersionPackage.GetLayerUpgradeConfig(layerUpgrade.Path)
    if err != nil {
        return fmt.Errorf("get layer upgrade config failed: %v", err)
    }
    
    // 执行层级升级
    if err := executor.Execute(layerUpgradeConfig); err != nil {
        m.ctx.Logger.Error(err, "Layer upgrade failed", "layer", layerUpgrade.Layer)
        return err
    }
    
    m.ctx.Logger.Info("Layer upgrade completed", "layer", layerUpgrade.Layer)
    return nil
}

func (m *UpgradeManager) getLayerExecutor(layer string) (LayerUpgradeExecutor, error) {
    // 检查执行器是否已存在
    if executor, ok := m.executors[layer]; ok {
        return executor, nil
    }
    
    // 创建新的执行器
    var executor LayerUpgradeExecutor
    switch layer {
    case "bootstrap":
        executor = NewBootstrapUpgradeExecutor(m.ctx)
    case "management":
        executor = NewManagementUpgradeExecutor(m.ctx)
    case "workload":
        executor = NewWorkloadUpgradeExecutor(m.ctx)
    default:
        return nil, fmt.Errorf("unsupported layer: %s", layer)
    }
    
    m.executors[layer] = executor
    return executor, nil
}
```
### 3.5 层级执行器实现（以管理集群为例）
```go
type ManagementUpgradeExecutor struct {
    ctx      *UpgradeContext
    client   client.Client
    sshMgr   *SSHManager
    kubeconfig string
}

func NewManagementUpgradeExecutor(ctx *UpgradeContext) *ManagementUpgradeExecutor {
    // 初始化客户端和SSH管理器
    kubeconfig, err := getManagementClusterKubeconfig()
    if err != nil {
        return nil, err
    }
    
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return nil, err
    }
    
    k8sClient, err := client.New(config, client.Options{})
    if err != nil {
        return nil, err
    }
    
    return &ManagementUpgradeExecutor{
        ctx:      ctx,
        client:   k8sClient,
        sshMgr:   NewSSHManager(),
        kubeconfig: kubeconfig,
    }
}

func (e *ManagementUpgradeExecutor) Execute(config *LayerUpgradeConfig) error {
    // 1. 执行前置检查
    if err := e.runPreChecks(config); err != nil {
        return fmt.Errorf("pre-checks failed: %v", err)
    }
    
    // 2. 备份当前状态
    if err := e.backupCurrentState(config); err != nil {
        return fmt.Errorf("backup failed: %v", err)
    }
    
    // 3. 执行组件升级
    for _, componentChange := range config.ComponentChanges {
        if err := e.upgradeComponent(componentChange); err != nil {
            return fmt.Errorf("upgrade component %s failed: %v", componentChange.Name, err)
        }
    }
    
    // 4. 执行后置检查
    if err := e.runPostChecks(config); err != nil {
        return fmt.Errorf("post-checks failed: %v", err)
    }
    
    // 5. 更新层级版本信息
    if err := e.updateLayerVersion(config); err != nil {
        return fmt.Errorf("update layer version failed: %v", err)
    }
    
    return nil
}

func (e *ManagementUpgradeExecutor) upgradeComponent(change *ComponentChange) error {
    e.ctx.Logger.Info("Upgrading component", "component", change.Name, 
        "from", change.FromVersion, "to", change.ToVersion)
    
    // 获取组件版本定义
    componentVersion, err := e.ctx.VersionPackage.GetComponentVersion(
        "management", change.Name, change.Type, change.ToVersion)
    if err != nil {
        return err
    }
    
    // 获取组件当前版本定义
    currentVersion, err := e.ctx.VersionPackage.GetComponentVersion(
        "management", change.Name, change.Type, change.FromVersion)
    if err != nil {
        return err
    }
    
    // 创建组件执行器
    executor := NewComponentUpgradeExecutor(e.ctx, e.client, e.sshMgr)
    
    // 执行组件升级
    if err := executor.Upgrade(componentVersion, currentVersion); err != nil {
        return err
    }
    
    e.ctx.Logger.Info("Component upgraded successfully", "component", change.Name)
    return nil
}
```
### 3.6 组件执行器实现
```go
type ComponentUpgradeExecutor struct {
    ctx      *UpgradeContext
    client   client.Client
    sshMgr   *SSHManager
}

func NewComponentUpgradeExecutor(ctx *UpgradeContext, client client.Client, sshMgr *SSHManager) *ComponentUpgradeExecutor {
    return &ComponentUpgradeExecutor{
        ctx:    ctx,
        client: client,
        sshMgr: sshMgr,
    }
}

func (e *ComponentUpgradeExecutor) Upgrade(targetVersion, currentVersion *ComponentVersion) error {
    // 1. 渲染升级脚本
    upgradeScript, err := e.renderUpgradeScript(targetVersion, currentVersion)
    if err != nil {
        return fmt.Errorf("render upgrade script failed: %v", err)
    }
    
    // 2. 获取所有需要升级的节点
    nodes, err := e.getTargetNodes(targetVersion)
    if err != nil {
        return fmt.Errorf("get target nodes failed: %v", err)
    }
    
    // 3. 按顺序升级每个节点
    for _, node := range nodes {
        if err := e.upgradeNode(node, upgradeScript, targetVersion); err != nil {
            return fmt.Errorf("upgrade node %s failed: %v", node.Name, err)
        }
    }
    
    // 4. 验证组件升级结果
    if err := e.verifyComponentUpgrade(targetVersion); err != nil {
        return fmt.Errorf("verify component upgrade failed: %v", err)
    }
    
    return nil
}

func (e *ComponentUpgradeExecutor) upgradeNode(node *corev1.Node, upgradeScript string, 
    componentVersion *ComponentVersion) error {
    
    e.ctx.Logger.Info("Upgrading node", "node", node.Name, "component", componentVersion.Spec.Name)
    
    // 获取节点IP
    nodeIP := getNodeIP(node)
    
    // 获取SSH密钥
    sshKey, err := e.getSSHKey()
    if err != nil {
        return err
    }
    
    // 创建SSH客户端
    sshClient, err := e.sshMgr.NewClient(nodeIP, 22, "root", sshKey)
    if err != nil {
        return err
    }
    defer sshClient.Close()
    
    // 如果是Dry-Run模式，只记录不执行
    if e.ctx.DryRun {
        e.ctx.Logger.Info("Dry-run: would execute upgrade script", "node", node.Name)
        return nil
    }
    
    // 上传升级脚本
    scriptPath := fmt.Sprintf("/tmp/upgrade-%s-%s.sh", 
        componentVersion.Spec.Name, componentVersion.Spec.Version)
    
    if err := sshClient.UploadFile(scriptPath, []byte(upgradeScript)); err != nil {
        return fmt.Errorf("upload upgrade script failed: %v", err)
    }
    
    // 执行升级脚本
    cmd := fmt.Sprintf("chmod +x %s && %s", scriptPath, scriptPath)
    output, err := sshClient.ExecuteCommand(cmd)
    if err != nil {
        return fmt.Errorf("execute upgrade script failed: %v, output: %s", err, output)
    }
    
    // 清理脚本
    sshClient.ExecuteCommand(fmt.Sprintf("rm %s", scriptPath))
    
    e.ctx.Logger.Info("Node upgraded successfully", "node", node.Name)
    return nil
}
```
### 3.7 回滚机制实现
```go
func (m *UpgradeManager) RollbackLayerUpgrade(layerUpgrade LayerUpgrade) error {
    m.ctx.Logger.Info("Starting layer rollback", "layer", layerUpgrade.Layer)
    
    // 获取层级执行器
    executor, err := m.getLayerExecutor(layerUpgrade.Layer)
    if err != nil {
        return fmt.Errorf("get layer executor failed: %v", err)
    }
    
    // 加载层级升级配置
    layerUpgradeConfig, err := m.ctx.VersionPackage.GetLayerUpgradeConfig(layerUpgrade.Path)
    if err != nil {
        return fmt.Errorf("get layer upgrade config failed: %v", err)
    }
    
    // 执行回滚
    if err := executor.Rollback(layerUpgradeConfig); err != nil {
        m.ctx.Logger.Error(err, "Layer rollback failed", "layer", layerUpgrade.Layer)
        return err
    }
    
    m.ctx.Logger.Info("Layer rollback completed", "layer", layerUpgrade.Layer)
    return nil
}

func (e *ManagementUpgradeExecutor) Rollback(config *LayerUpgradeConfig) error {
    // 1. 执行回滚前检查
    if err := e.runRollbackPreChecks(config); err != nil {
        return fmt.Errorf("rollback pre-checks failed: %v", err)
    }
    
    // 2. 恢复备份
    if err := e.restoreBackup(config); err != nil {
        return fmt.Errorf("restore backup failed: %v", err)
    }
    
    // 3. 执行组件回滚
    for i := len(config.ComponentChanges) - 1; i >= 0; i-- {
        componentChange := config.ComponentChanges[i]
        if err := e.rollbackComponent(componentChange); err != nil {
            return fmt.Errorf("rollback component %s failed: %v", componentChange.Name, err)
        }
    }
    
    // 4. 执行回滚后检查
    if err := e.runRollbackPostChecks(config); err != nil {
        return fmt.Errorf("rollback post-checks failed: %v", err)
    }
    
    // 5. 恢复层级版本信息
    if err := e.restoreLayerVersion(config); err != nil {
        return fmt.Errorf("restore layer version failed: %v", err)
    }
    
    return nil
}
```
### 3.8 Dry-Run模式支持
```go
func (e *ComponentUpgradeExecutor) Upgrade(targetVersion, currentVersion *ComponentVersion) error {
    // ...
    
    // 3. 按顺序升级每个节点
    for _, node := range nodes {
        if err := e.upgradeNode(node, upgradeScript, targetVersion); err != nil {
            return fmt.Errorf("upgrade node %s failed: %v", node.Name, err)
        }
    }
    
    // ...
}

func (e *ComponentUpgradeExecutor) upgradeNode(node *corev1.Node, upgradeScript string, 
    componentVersion *ComponentVersion) error {
    
    // ...
    
    // 如果是Dry-Run模式，只记录不执行
    if e.ctx.DryRun {
        e.ctx.Logger.Info("Dry-run: would execute upgrade script", 
            "node", node.Name,
            "script", upgradeScript[:100] + "...")  // 只显示脚本开头
        return nil
    }
    
    // ... 实际执行升级操作
}
```
## 四、关键技术实现细节
### 4.1 版本兼容性验证
```go
func (e *ComponentUpgradeExecutor) verifyCompatibility(targetVersion, currentVersion *ComponentVersion) error {
    // 检查当前版本是否在目标版本的兼容列表中
    compatible := false
    for _, compatibleVersion := range targetVersion.Spec.Compatibility.PreviousVersions {
        if currentVersion.Spec.Version == compatibleVersion {
            compatible = true
            break
        }
    }
    
    if !compatible {
        return fmt.Errorf("component %s version %s is not compatible with target version %s", 
            targetVersion.Spec.Name, currentVersion.Spec.Version, targetVersion.Spec.Version)
    }
    
    // 检查Kubernetes版本兼容性
    kubernetesVersion, err := e.getKubernetesVersion()
    if err != nil {
        return err
    }
    
    if !isVersionInRange(kubernetesVersion, targetVersion.Spec.Compatibility.KubernetesVersions.Min, 
        targetVersion.Spec.Compatibility.KubernetesVersions.Max) {
        return fmt.Errorf("Kubernetes version %s is not compatible with component %s version %s", 
            kubernetesVersion, targetVersion.Spec.Name, targetVersion.Spec.Version)
    }
    
    return nil
}
```
### 4.2 并行升级与批量控制
```go
func (e *WorkloadUpgradeExecutor) Execute(config *LayerUpgradeConfig) error {
    // ...
    
    // 3. 执行组件升级（支持批量并行）
    upgradeStrategy := config.UpgradeStrategy
    
    // 根据策略分组节点
    nodeGroups := e.groupNodesByStrategy(upgradeStrategy)
    
    // 按组执行升级
    for _, group := range nodeGroups {
        // 并行升级组内节点
        var wg sync.WaitGroup
        errs := make(chan error, len(group.Nodes))
        
        for _, node := range group.Nodes {
            wg.Add(1)
            go func(n *corev1.Node) {
                defer wg.Done()
                
                // 升级节点
                if err := e.upgradeNode(n, config); err != nil {
                    errs <- err
                    return
                }
                
                // 等待节点就绪
                if err := e.waitForNodeReady(n, upgradeStrategy.NodeReadyTimeout); err != nil {
                    errs <- err
                    return
                }
            }(node)
        }
        
        wg.Wait()
        close(errs)
        
        // 检查错误
        for err := range errs {
            if err != nil {
                return fmt.Errorf("upgrade group failed: %v", err)
            }
        }
        
        // 组间等待
        time.Sleep(upgradeStrategy.GroupInterval.Duration)
    }
    
    // ...
}
```
### 4.3 健康检查与监控
```go
func (e *ManagementUpgradeExecutor) runPostChecks(config *LayerUpgradeConfig) error {
    e.ctx.Logger.Info("Running post-upgrade health checks")
    
    // 检查集群状态
    if err := e.checkClusterHealth(); err != nil {
        return fmt.Errorf("cluster health check failed: %v", err)
    }
    
    // 检查节点状态
    if err := e.checkNodesHealth(); err != nil {
        return fmt.Errorf("nodes health check failed: %v", err)
    }
    
    // 检查组件状态
    for _, componentChange := range config.ComponentChanges {
        if err := e.checkComponentHealth(componentChange); err != nil {
            return fmt.Errorf("component %s health check failed: %v", componentChange.Name, err)
        }
    }
    
    // 检查API可用性
    if err := e.checkAPIAvailability(); err != nil {
        return fmt.Errorf("API availability check failed: %v", err)
    }
    
    return nil
}

func (e *ManagementUpgradeExecutor) checkComponentHealth(componentChange *ComponentChange) error {
    // 构建健康检查命令
    healthCheckCmd := fmt.Sprintf("systemctl is-active %s", componentChange.Name)
    
    // 获取所有节点
    nodes, err := e.getTargetNodesForComponent(componentChange)
    if err != nil {
        return err
    }
    
    // 在每个节点上执行健康检查
    for _, node := range nodes {
        nodeIP := getNodeIP(node)
        sshClient, err := e.sshMgr.NewClient(nodeIP, 22, "root", e.getSSHKey())
        if err != nil {
            return err
        }
        defer sshClient.Close()
        
        output, err := sshClient.ExecuteCommand(healthCheckCmd)
        if err != nil {
            return fmt.Errorf("health check failed on node %s: %v", node.Name, err)
        }
        
        if strings.TrimSpace(output) != "active" {
            return fmt.Errorf("component %s is not active on node %s", componentChange.Name, node.Name)
        }
    }
    
    return nil
}
```
## 五、总结
`upgrade.ExecuteUpgrade` 函数通过分层架构和模块化设计，实现了集群管理解决方案的全流程自动化升级。其核心特点包括：
1. **分层执行**：按引导集群 → 管理集群 → 业务集群的顺序执行升级
2. **组件级隔离**：每个组件的升级独立封装，便于维护和扩展
3. **安全可靠**：包含完善的备份、回滚和健康检查机制
4. **可观测性**：详细的日志和指标收集，便于问题定位
5. **灵活扩展**：支持自定义升级策略和组件执行器
6. **Dry-Run支持**：允许在不实际修改系统的情况下测试升级流程

这种设计确保了升级过程的安全性、可靠性和可维护性，能够有效支持集群管理解决方案的版本迭代和升级需求。
