# 关于OpenShift节点配置方式的研究
## OpenShift节点配置方式
### 核心结论
OpenShift的Bootstrap和管理节点**确实是从虚拟机镜像开始的**，而不是手工安装操作系统。
### 技术细节
1. **使用预构建的VM镜像**
   - RHCOS (Red Hat Enterprise Linux CoreOS) 用于OpenShift
   - FCOS (Fedora CoreOS) 用于OKD (开源版本)
   - 这些镜像已经包含了操作系统和必要的组件
2. **Ignition配置机制**
   - 节点启动时通过Ignition配置文件进行初始化
   - Ignition负责：
     - 配置网络
     - 设置存储
     - 创建用户和SSH密钥
     - 部署系统服务
     - 配置kubelet等Kubernetes组件
3. **IPI vs UPI的区别**
   - **IPI (Installer-Provisioned Infrastructure)**: 自动创建VM并应用配置
   - **UPI (User-Provisioned Infrastructure)**: 用户手动创建VM，但仍然使用镜像和Ignition配置
4. **不可变基础设施**
   - CoreOS采用不可变设计
   - 系统更新通过替换整个镜像实现
   - 配置变更通过新的Ignition配置应用
### 与传统方式的对比
| 方面 | 传统方式 | OpenShift方式 |
|------|---------|--------------|
| OS安装 | 手工安装 | 预构建镜像 |
| 配置管理 | 手工配置 | Ignition自动配置 |
| 系统更新 | 包管理器更新 | 镜像替换 |
| 一致性 | 难以保证 | 高度一致 |

这种方式的优势在于：
- 快速部署
- 配置一致性
- 易于版本管理和回滚
- 符合云原生和不可变基础设施的最佳实践

# 详细解答OpenShift在物理机环境下的安装架构
## 核心答案
**不需要**。OpenShift在裸金属环境下，节点**直接运行在物理机上**，不需要虚拟化软件。
## 详细说明
### 1. OpenShift在裸金属环境下的部署架构
```
┌─────────────────────────────────────────────────────────┐
│                    物理服务器                             │
│  ┌───────────────────────────────────────────────────┐  │
│  │         RHCOS/FCOS (直接运行在硬件上)              │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │         OpenShift/Kubernetes                │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │         容器工作负载                   │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```
**没有虚拟化层！**
### 2. 用户需要提供什么？
#### IPI方式（安装程序自动配置）
- ✅ 物理服务器（带BMC/IPMI支持）
- ✅ 网络基础设施（交换机、路由器）
- ✅ DNS和DHCP服务
- ❌ 不需要预装操作系统
- ❌ 不需要预装虚拟化软件
#### UPI方式（用户手动配置）
- ✅ 物理服务器
- ✅ 网络基础设施
- ✅ DNS/DHCP服务
- ✅ 负载均衡器（如HAProxy）
- ✅ Bastion节点（用于提供Ignition配置和镜像仓库）
- ❌ 不需要预装操作系统
- ❌ 不需要预装虚拟化软件
### 3. 安装流程对比
| 环境 | 虚拟化层 | 节点运行方式 |
|------|---------|-------------|
| **裸金属** | 无 | RHCOS直接运行在硬件上 |
| **vSphere** | VMware ESXi | RHCOS运行在虚拟机中 |
| **KVM/libvirt** | KVM | RHCOS运行在虚拟机中 |
| **云平台** | 云厂商虚拟化 | RHCOS运行在云实例中 |
### 4. 裸金属安装的关键技术
#### PXE引导 + Ignition配置
```bash
# 1. 物理机通过PXE网络引导
# 2. 加载RHCOS镜像
# 3. 应用Ignition配置
# 4. 自动加入OpenShift集群
```
#### Bastion节点的作用
```yaml
Bastion节点需要提供：
- DNS服务
- DHCP/PXE服务
- HTTP服务（提供Ignition配置）
- 镜像仓库
- 负载均衡器
```
### 5. 裸金属部署的优势
根据实际测试数据：

| 指标 | 裸金属 | 虚拟化环境 |
|------|--------|-----------|
| 网络延迟 | 基准 | 增加40-60% |
| 存储IOPS | 基准 | 降低3-5倍 |
| 容器启动时间 | 基准 | 增加30% |
| 性能损耗 | 0% | 15-30% |
### 6. 实际案例
从搜索结果中的案例：
> "最近我们给一家保险公司部署OpenShift时，他们的数据库服务在虚拟化环境下延迟高达8ms，迁移到裸金属后直接降到1.2ms。"
## 总结
**OpenShift在裸金属环境下：**
1. ✅ 用户只需提供物理服务器
2. ✅ OpenShift节点直接运行在RHCOS/FCOS上
3. ❌ **不需要**安装KVM、VMware等虚拟化软件
4. ❌ **不需要**手工安装操作系统
5. ✅ 通过PXE+Ignition实现自动化部署

这种方式特别适合：
- 性能敏感型应用（数据库、AI/ML）
- 安全合规要求高的场景（金融、医疗）
- 需要直接访问硬件资源的工作负载

# OpenShift 中登录节点的认证方式
## OpenShift 节点登录方式：SSH 密钥免密登录
根据搜索结果和官方文档，**OpenShift 中登录节点使用的是 SSH 密钥免密登录，而不是用户名/密码登录**。
### 一、SSH 密钥认证流程
```
┌─────────────────────────────────────────────────────────────┐
│              OpenShift 节点 SSH 认证流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 安装前准备：生成 SSH 密钥对                              │
│     ┌──────────────┐                                        │
│     │ Bastion      │                                        │
│     │ (安装节点)   │                                        │
│     │              │───ssh-keygen───> 生成密钥对            │
│     └──────────────┘                       │                │
│                                            │                │
│                     ┌──────────────────────┤                │
│                     │                      │                │
│                     ▼                      ▼                │
│              ┌────────────┐         ┌────────────┐         │
│              │ id_rsa     │         │ id_rsa.pub │         │
│              │ (私钥)     │         │ (公钥)     │         │
│              └────────────┘         └────────────┘         │
│                                                              │
│  2. 公钥分发到集群节点                                       │
│     ┌──────────────┐                                        │
│     │ Bastion      │                                        │
│     │              │───ssh-copy-id───> 分发公钥到节点      │
│     └──────────────┘                                        │
│                                            │                │
│                     ┌──────────────────────┤                │
│                     │                      │                │
│                     ▼                      ▼                │
│              ┌────────────┐         ┌────────────┐         │
│              │ Master节点 │         │ Worker节点 │         │
│              │            │         │            │         │
│              │ ~/.ssh/    │         │ ~/.ssh/    │         │
│              │ authorized │         │ authorized │         │
│              │ _keys      │         │ _keys      │         │
│              └────────────┘         └────────────┘         │
│                                                              │
│  3. 免密登录节点                                             │
│     ┌──────────────┐                                        │
│     │ Bastion      │                                        │
│     │              │───ssh core@node-ip───> 免密登录       │
│     └──────────────┘                                        │
│                                            │                │
│                                            ▼                │
│                                     ┌────────────┐         │
│                                     │ 节点       │         │
│                                     │ (无需密码) │         │
│                                     └────────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```
### 二、具体实现步骤
#### 2.1 生成 SSH 密钥对
```bash
# 在 Bastion 节点（安装节点）上执行
$ ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

# 参数说明：
# -t rsa: 使用 RSA 算法
# -b 4096: 密钥长度 4096 位（安全要求）
# -N '': 密钥密码为空（免密登录）
# -f ~/.ssh/id_rsa: 指定密钥文件路径

# 生成结果：
# ~/.ssh/id_rsa (私钥 - 保密)
# ~/.ssh/id_rsa.pub (公钥 - 分发到节点)
```
#### 2.2 分发公钥到节点
```bash
# 方式1: 使用 ssh-copy-id（推荐）
$ ssh-copy-id -i ~/.ssh/id_rsa.pub core@<node-ip>

# 方式2: 手动复制（如果 ssh-copy-id 不可用）
$ cat ~/.ssh/id_rsa.pub | ssh core@<node-ip> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
#### 2.3 测试免密登录
```bash
# 测试登录 Master 节点
$ ssh core@master0.okd4.example.com

# 测试登录 Worker 节点
$ ssh core@worker0.okd4.example.com

# 成功标志：无需输入密码即可登录
```
### 三、OpenShift 节点用户说明
```
┌─────────────────────────────────────────────────────────────┐
│              OpenShift 节点用户类型                          │
├──────────────────┬──────────────────────────────────────────┤
│     用户类型     │              说明                        │
├──────────────────┼──────────────────────────────────────────┤
│ core             │ OpenShift 4.x 默认用户                   │
│                  │ 用于 SSH 登录节点                        │
│                  │ 具有 sudo 权限                           │
│                  │ 使用 Fedora CoreOS/RHCOS                │
├──────────────────┼──────────────────────────────────────────┤
│ root             │ 超级用户                                 │
│                  │ 默认禁用 SSH 登录                        │
│                  │ 通过 core 用户 sudo 切换                 │
├──────────────────┼──────────────────────────────────────────┤
│ system:admin     │ 集群管理员                               │
│                  │ 用于 kubectl/oc 命令                     │
│                  │ 不是系统用户                             │
└──────────────────┴──────────────────────────────────────────┘
```
### 四、为什么使用 SSH 密钥而不是密码？
```
┌─────────────────────────────────────────────────────────────┐
│              SSH 密钥 vs 密码认证对比                        │
├──────────────────┬──────────────────┬──────────────────────┤
│     特性         │   SSH 密钥       │     密码认证         │
├──────────────────┼──────────────────┼──────────────────────┤
│ 安全性           │ 高               │ 低                   │
│                  │ 4096位RSA密钥    │ 容易暴力破解         │
│                  │ 几乎无法破解     │                      │
├──────────────────┼──────────────────┼──────────────────────┤
│ 暴力破解防护     │ 天然防护         │ 需要额外配置         │
│                  │ 私钥不可猜测     │ fail2ban等           │
├──────────────────┼──────────────────┼──────────────────────┤
│ 自动化支持       │ 支持             │ 困难                 │
│                  │ 脚本无需交互     │ 需要交互输入         │
├──────────────────┼──────────────────┼──────────────────────┤
│ 密钥管理         │ 集中管理         │ 分散管理             │
│                  │ 可撤销单个密钥   │ 需改所有密码         │
├──────────────────┼──────────────────┼──────────────────────┤
│ 审计追踪         │ 可追踪密钥       │ 难以追踪             │
│                  │ 每个用户独立密钥 │ 共享密码问题         │
├──────────────────┼──────────────────┼──────────────────────┤
│ OpenShift要求    │ 强制要求         │ 不支持               │
│                  │ 安装时必须配置   │                      │
└──────────────────┴──────────────────┴──────────────────────┘
```
### 五、OpenShift 安装配置示例
#### 5.1 install-config.yaml 配置
```yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 3
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: okd4
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths": ...}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC...'  # SSH 公钥
```
**关键点**：
- `sshKey` 字段必须配置
- 值为 `~/.ssh/id_rsa.pub` 的内容
- 安装程序会自动将此公钥注入到所有节点
#### 5.2 节点 Ignition 配置
```json
{
  "ignition": {
    "version": "3.2.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "sshAuthorizedKeys": [
          "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC..."
        ]
      }
    ]
  }
}
```
### 六、安全最佳实践
```
┌─────────────────────────────────────────────────────────────┐
│              SSH 密钥安全最佳实践                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 密钥强度要求                                            │
│     ✓ RSA 密钥长度 >= 4096 位                               │
│     ✓ 或使用 ED25519 算法                                   │
│     ✗ 避免使用 RSA-2048（已不够安全）                       │
│                                                              │
│  2. 私钥保护                                                │
│     ✓ 私钥文件权限: chmod 600 ~/.ssh/id_rsa                 │
│     ✓ 私钥目录权限: chmod 700 ~/.ssh                        │
│     ✓ 考虑为私钥设置密码短语                                │
│     ✗ 不要共享私钥                                          │
│     ✗ 不要上传到 GitHub 等公开仓库                          │
│                                                              │
│  3. 公钥管理                                                │
│     ✓ 每个管理员使用独立密钥                                │
│     ✓ 定期轮换密钥（建议 6-12 个月）                        │
│     ✓ 离职员工及时删除公钥                                  │
│     ✓ 使用 SSH CA 签名密钥（企业环境）                      │
│                                                              │
│  4. 节点安全配置                                            │
│     ✓ 禁用密码登录:                                         │
│       PasswordAuthentication no                              │
│     ✓ 禁用 root 登录:                                       │
│       PermitRootLogin no                                     │
│     ✓ 仅允许密钥认证:                                       │
│       PubkeyAuthentication yes                               │
│     ✓ 限制登录用户:                                         │
│       AllowUsers core                                        │
│                                                              │
│  5. 审计与监控                                              │
│     ✓ 启用 SSH 登录日志                                     │
│     ✓ 监控异常登录尝试                                      │
│     ✓ 定期审计 authorized_keys                              │
│     ✓ 使用堡垒机集中管理                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```
### 七、常见问题排查
#### 7.1 权限问题
```bash
# 检查本地私钥权限
$ ls -l ~/.ssh/id_rsa
-rw------- 1 user user 3243 Jan 1 12:00 /home/user/.ssh/id_rsa
# 必须是 600 (rw-------)

# 检查远程节点权限
$ ssh core@node "ls -la ~/.ssh"
total 8
drwx------ 2 core core 4096 Jan 1 12:00 .
drwx------ 1 core core 4096 Jan 1 12:00 ..
-rw------- 1 core core  420 Jan 1 12:00 authorized_keys
# 目录权限: 700 (drwx------)
# 文件权限: 600 (rw-------)

# 修复权限
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/id_rsa
$ chmod 600 ~/.ssh/authorized_keys
```
#### 7.2 SELinux 问题
```bash
# 检查 SELinux 上下文
$ ls -Z ~/.ssh/authorized_keys
unconfined_u:object_r:ssh_home_t:s0 ~/.ssh/authorized_keys

# 恢复正确的 SELinux 上下文
$ restorecon -Rv ~/.ssh

# 临时关闭 SELinux（仅用于测试）
$ sudo setenforce 0
```
#### 7.3 调试 SSH 连接
```bash
# 详细模式查看连接过程
$ ssh -v core@node-ip

# 更详细的调试
$ ssh -vvv core@node-ip

# 检查 SSH 配置
$ ssh -G node-ip | grep -i "identityfile"
identityfile ~/.ssh/id_rsa
```
### 八、总结
**OpenShift 节点登录方式**：
1. **使用 SSH 密钥免密登录**（强制要求）
   - 安装时必须配置 SSH 公钥
   - 使用 `core` 用户登录
   - 无需密码，使用私钥认证
2. **不使用用户名/密码登录**
   - 安全性考虑
   - OpenShift 默认禁用密码认证
   - 防止暴力破解
3. **安全优势**
   - 4096位 RSA 密钥几乎无法破解
   - 支持自动化运维
   - 集中管理密钥
   - 审计追踪
4. **企业级实践**
   - 使用堡垒机集中管理
   - SSH CA 签名密钥
   - 定期轮换密钥
   - 严格的权限控制

这种设计确保了 OpenShift 集群节点的安全性，符合企业级安全要求。

# SSH 密钥免密登录
**SSH 密钥免密登录的原理是基于公钥加密：客户端持有私钥，服务器保存公钥，登录时通过加密挑战验证身份，无需输入密码。**这种方式比传统密码更安全，也更适合自动化场景。
## 📌 核心原理
1. **密钥对生成**  
   - 使用 `ssh-keygen` 在客户端生成一对密钥：**私钥**（保存在本地）和 **公钥**（复制到服务器）。  
   - 常用算法：Ed25519（推荐，安全且性能好）、RSA（传统）。  
2. **公钥存储**  
   - 公钥被写入服务器的 `~/.ssh/authorized_keys` 文件。  
   - 服务器只接受在该文件中存在的公钥对应的客户端连接。  
3. **认证过程**  
   - 客户端发起连接时，服务器用公钥生成一个随机挑战。  
   - 客户端用私钥解密并返回结果。  
   - 服务器验证结果正确，即确认客户端持有对应私钥 → 登录成功。  
4. **安全性**  
   - 私钥永远不离开客户端，不会在网络上传输。  
   - 即使有人截获通信，也无法伪造登录。  
   - 比密码更安全，因为密码容易被暴力破解或泄露。  
## 🧩 工作流程简化
1. 客户端生成密钥对：`ssh-keygen -t ed25519`  
2. 将公钥复制到服务器：`ssh-copy-id user@server`  
3. 登录时：`ssh user@server` → 自动使用私钥认证，无需密码。  
## 📊 对比：密码 vs 公钥认证
| 特性 | 密码认证 | 公钥认证 |
|------|----------|----------|
| 安全性 | 容易被暴力破解或泄露 | 私钥不传输，安全性更高 |
| 便利性 | 每次都要输入密码 | 一次配置，后续免密登录 |
| 自动化 | 不适合脚本和批量任务 | 适合 rsync、scp、自动化部署 |
| 管理 | 用户多时难以统一管理 | 可集中管理公钥文件 |
## ⚠️ 风险与注意事项
- **私钥保护**：必须妥善保管，建议设置文件权限 `chmod 600 ~/.ssh/id_ed25519`。  
- **不要泄露私钥**：一旦泄露，攻击者可直接登录服务器。  
- **多用户管理**：在服务器上为不同用户配置不同公钥，避免混用。  

✅ **总结**：SSH 密钥免密登录的核心是 **公钥加密认证**，客户端持有私钥，服务器保存公钥，通过挑战-响应机制验证身份。它比密码更安全、更方便，尤其适合自动化任务和大规模运维。  

# Bastion节点的作用和架构
## Bastion节点详解
### 1. 在OpenShift UPI安装中的角色
```
┌─────────────────────────────────────────────────────────────┐
│                    OpenShift UPI 架构                        │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Bastion节点 (堡垒机)                      │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  核心服务：                                      │  │  │
│  │  │  • DNS服务                          │  │  │
│  │  │  • DHCP/PXE服务 (网络引导)                       │  │  │
│  │  │  • HTTP服务 (提供Ignition配置)                   │  │  │
│  │  │  • 镜像仓库                        │  │  │
│  │  │  • 负载均衡器                      │  │  │
│  │  │  • OpenShift安装工具                             │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            目标集群节点                                │   │
│  │  • Bootstrap节点                                      │   │
│  │  • Master节点 (3个)                                   │   │
│  │  • Worker节点 (N个)                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```
### 2. Bastion节点提供的服务
| 服务 | 作用 | 端口 |
|------|------|------|
| **DNS** | 解析集群域名和节点主机名 | 53/TCP,UDP |
| **DHCP/PXE** | 为节点分配IP并提供引导 | 67/UDP, 69/TFTP |
| **HTTP** | 提供Ignition配置和RHCOS镜像 | 80/TCP |
| **Harbor** | 离线镜像仓库 | 443/TCP |
| **HAProxy** | API Server负载均衡 | 6443/TCP, 22623/TCP |
| **NTP** | 时间同步服务 | 123/UDP |
### 3. Bastion节点配置示例
根据之前搜索到的OpenShift文档：
```yaml
节点信息：
  Hostname: bastion.okd4.example.com
  IP: 192.168.72.20
  CPU: 2核
  内存: 4GB
  磁盘: 100GB
  OS: Ubuntu 20.04 / RHEL / CentOS
```
### 4. 与BKE架构的对比
在BKE架构中，**没有显式的Bastion节点概念**，而是采用不同的设计：
```
┌─────────────────────────────────────────────────────────────┐
│                    BKE 架构                                  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │         管理集群                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  bke-controller-manager Pod:                    │  │  │
│  │  │  • cluster-api-provider-bke                     │  │  │
│  │  │  • bke-manifests (InitContainer)                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  openfuyao-system-controller Pod                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         目标集群                        │   │
│  │  • Master节点                                          │   │
│  │  • Worker节点                                          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```
### 5. 关键区别
| 方面 | OpenShift UPI | BKE |
|------|--------------|-----|
| **Bastion节点** | 必需（物理机或VM） | 不需要独立节点 |
| **服务部署位置** | Bastion节点上 | 管理集群的Pod中 |
| **DNS服务** | Bastion上的Bind9 | 集群内DNS或外部DNS |
| **镜像仓库** | Bastion上的Harbor | 独立的镜像仓库服务 |
| **负载均衡** | Bastion上的HAProxy | 集群内或外部LB |
| **配置分发** | HTTP+Ignition | CRD+Controller |
### 6. BKE中的等价功能
BKE将Bastion节点的功能分散到不同组件：
```go
// 来自 bkeadm 的设计
┌─────────────────────────────────────────────────────────┐
│  bkeadm 命令行工具                                       │
│  • 生成配置文件 (类似Ignition)                           │
│  • 管理镜像仓库                          │
│  • 部署管理组件                                          │
└─────────────────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────────────────┐
│  cluster-api-provider-bke                                │
│  • 管理目标集群生命周期                                  │
│  • 分发配置到目标节点                                    │
└─────────────────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────────────────┐
│  openfuyao-system-controller                             │
│  • 系统级控制逻辑                                        │
│  • 插件管理                                              │
└─────────────────────────────────────────────────────────┘
```
### 7. 总结
**OpenShift的Bastion节点**：
- 是一个独立的物理机或虚拟机
- 提供所有基础设施服务
- 是安装过程的控制中心
- 安装完成后可以保留或删除

**BKE的设计**：
- 没有独立的Bastion节点
- 功能由管理集群的Pod提供
- 更加云原生化
- 更适合Kubernetes环境

# 设计一个云原生的Bastion节点替代方案
## 云原生Bastion替代方案设计
### 1. 整体架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                    管理集群                          │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Namespace: infrastructure-system                               │ │
│  │                                                                  │ │
│  │  ┌──────────────────────────────────────────────────────────┐  │ │
│  │  │  Bastion Operator (核心控制器)                            │  │ │
│  │  │  • 管理所有基础设施服务                                    │  │ │
│  │  │  • 协调服务生命周期                                        │  │ │
│  │  │  • 提供统一API                                            │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  │                                                                  │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │ │
│  │  │ CoreDNS     │ │ MetalLB/    │ │ Harbor      │              │ │
│  │  │ Operator    │ │ Keepalived  │ │ Operator    │              │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘              │ │
│  │                                                                  │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │ │
│  │  │ DHCP/DNS    │ │ PXE/TFTP    │ │ Ignition    │              │ │
│  │  │ Operator    │ │ Operator    │ │ Server      │              │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    目标集群节点                        │
│  Bootstrap节点 → Master节点 → Worker节点                             │
└─────────────────────────────────────────────────────────────────────┘
```
### 2. 核心CRD设计
#### 2.1 InfrastructureCluster CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: infrastructureclusters.infrastructure.bocloud.com
spec:
  group: infrastructure.bocloud.com
  names:
    kind: InfrastructureCluster
    listKind: InfrastructureClusterList
    plural: infrastructureclusters
    singular: infrastructurecluster
    shortNames: [ic]
  scope: Namespaced
  versions:
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                clusterName:
                  type: string
                  description: 目标集群名称
                
                baseDomain:
                  type: string
                  description: 基础域名
                  example: example.com
                
                network:
                  type: object
                  properties:
                    podCIDR:
                      type: string
                      default: "10.128.0.0/14"
                    serviceCIDR:
                      type: string
                      default: "172.30.0.0/16"
                    machineCIDR:
                      type: string
                      description: 节点网络CIDR
                    vip:
                      type: string
                      description: API Server虚拟IP
                
                dns:
                  type: object
                  properties:
                    upstreamServers:
                      type: array
                      items:
                        type: string
                    forwardRules:
                      type: array
                      items:
                        type: object
                        properties:
                          domain:
                            type: string
                          server:
                            type: string
                
                dhcp:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    range:
                      type: object
                      properties:
                        start:
                          type: string
                        end:
                          type: string
                    gateway:
                      type: string
                    dnsServers:
                      type: array
                      items:
                        type: string
                
                pxe:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    tftpServer:
                      type: string
                    httpServer:
                      type: string
                    bootImage:
                      type: string
                
                ignition:
                  type: object
                  properties:
                    serverURL:
                      type: string
                    configTemplates:
                      type: object
                      additionalProperties:
                        type: string
                
                registry:
                  type: object
                  properties:
                    url:
                      type: string
                    mirrorConfig:
                      type: object
                      additionalProperties:
                        type: string
                    authSecretRef:
                      type: object
                      properties:
                        name:
                          type: string
                        namespace:
                          type: string
                
                loadBalancer:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: [metalLB, keepalived, external]
                      default: metalLB
                    ipPool:
                      type: object
                      properties:
                        start:
                          type: string
                        end:
                          type: string
                
                nodes:
                  type: array
                  items:
                    type: object
                    properties:
                      hostname:
                        type: string
                      macAddress:
                        type: string
                      ipAddress:
                        type: string
                      role:
                        type: string
                        enum: [bootstrap, master, worker]
                      ignitionConfigRef:
                        type: object
                        properties:
                          name:
                            type: string
                          namespace:
                            type: string
                
                services:
                  type: object
                  properties:
                    dns:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                        replicas:
                          type: integer
                          default: 2
                    dhcp:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                    pxe:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                    registry:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                        storageSize:
                          type: string
                          default: "100Gi"
                    loadBalancer:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Provisioning, Provisioned, Failed, Deleting]
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
                      reason:
                        type: string
                      message:
                        type: string
                serviceEndpoints:
                  type: object
                  properties:
                    dns:
                      type: string
                    dhcp:
                      type: string
                    pxe:
                      type: string
                    registry:
                      type: string
                    apiServer:
                      type: string
```
#### 2.2 IgnitionConfig CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ignitionconfigs.infrastructure.bocloud.com
spec:
  group: infrastructure.bocloud.com
  names:
    kind: IgnitionConfig
    listKind: IgnitionConfigList
    plural: ignitionconfigs
    singular: ignitionconfig
    shortNames: [ic]
  scope: Namespaced
  versions:
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                nodeSelector:
                  type: object
                  properties:
                    matchLabels:
                      type: object
                      additionalProperties:
                        type: string
                
                template:
                  type: object
                  description: Ignition配置模板
                  properties:
                    ignition:
                      type: object
                      properties:
                        version:
                          type: string
                          default: "3.2.0"
                    passwd:
                      type: object
                      properties:
                        users:
                          type: array
                          items:
                            type: object
                            properties:
                              name:
                                type: string
                              sshAuthorizedKeys:
                                type: array
                                items:
                                  type: string
                    storage:
                      type: object
                      properties:
                        files:
                          type: array
                          items:
                            type: object
                            properties:
                              path:
                                type: string
                              contents:
                                type: object
                                properties:
                                  source:
                                    type: string
                                  verification:
                                    type: object
                                    properties:
                                      hash:
                                        type: string
                              mode:
                                type: integer
                              overwrite:
                                type: boolean
                    systemd:
                      type: object
                      properties:
                        units:
                          type: array
                          items:
                            type: object
                            properties:
                              name:
                                type: string
                              enabled:
                                type: boolean
                              contents:
                                type: string
                
                overrides:
                  type: object
                  description: 节点特定的配置覆盖
                  additionalProperties:
                    type: object
                    
            status:
              type: object
              properties:
                generatedConfig:
                  type: string
                checksum:
                  type: string
                lastUpdateTime:
                  type: string
                  format: date-time
```
### 3. Operator实现设计
#### 3.1 Bastion Operator架构
```go
package controller

import (
    "context"
    "fmt"
    
    infrastructurev1beta1 "github.com/bocloud/infrastructure-operator/api/v1beta1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/record"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

type InfrastructureClusterReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
    
    DNSController        *DNSController
    DHCPController       *DHCPController
    PXEController        *PXEController
    IgnitionController   *IgnitionController
    RegistryController   *RegistryController
    LoadBalancerController *LoadBalancerController
}

func (r *InfrastructureClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    var infraCluster infrastructurev1beta1.InfrastructureCluster
    if err := r.Get(ctx, req.NamespacedName, &infraCluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if !infraCluster.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, &infraCluster)
    }
    
    return r.reconcileNormal(ctx, &infraCluster)
}

func (r *InfrastructureClusterReconciler) reconcileNormal(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    phases := []struct {
        name string
        fn   func(context.Context, *infrastructurev1beta1.InfrastructureCluster) error
    }{
        {"DNS", r.DNSController.Reconcile},
        {"DHCP", r.DHCPController.Reconcile},
        {"PXE", r.PXEController.Reconcile},
        {"Ignition", r.IgnitionController.Reconcile},
        {"Registry", r.RegistryController.Reconcile},
        {"LoadBalancer", r.LoadBalancerController.Reconcile},
    }
    
    for _, phase := range phases {
        log.Info("Reconciling phase", "phase", phase.name)
        if err := phase.fn(ctx, infraCluster); err != nil {
            r.Recorder.Eventf(infraCluster, "Warning", "ReconcileFailed", 
                "Failed to reconcile %s: %v", phase.name, err)
            return ctrl.Result{}, err
        }
    }
    
    infraCluster.Status.Phase = "Provisioned"
    if err := r.Status().Update(ctx, infraCluster); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```
#### 3.2 DNS Controller实现
```go
package controller

import (
    "context"
    "fmt"
    
    infrastructurev1beta1 "github.com/bocloud/infrastructure-operator/api/v1beta1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type DNSController struct {
    client.Client
    Scheme *runtime.Scheme
}

func (c *DNSController) Reconcile(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    if !infraCluster.Spec.Services.DNS.Enabled {
        return nil
    }
    
    if err := c.createCoreDNSConfigMap(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create CoreDNS ConfigMap: %w", err)
    }
    
    if err := c.createCoreDNSDeployment(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create CoreDNS Deployment: %w", err)
    }
    
    if err := c.createCoreDNSService(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create CoreDNS Service: %w", err)
    }
    
    return nil
}

func (c *DNSController) createCoreDNSConfigMap(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    corefile := c.generateCorefile(infraCluster)
    hostsFile := c.generateHostsFile(infraCluster)
    
    configMap := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-coredns", infraCluster.Name),
            Namespace: infraCluster.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: infrastructurev1beta1.GroupVersion.String(),
                    Kind:       "InfrastructureCluster",
                    Name:       infraCluster.Name,
                    UID:        infraCluster.UID,
                },
            },
        },
        Data: map[string]string{
            "Corefile": corefile,
            "hosts":    hostsFile,
        },
    }
    
    return c.Create(ctx, configMap)
}

func (c *DNSController) generateCorefile(infraCluster *infrastructurev1beta1.InfrastructureCluster) string {
    baseDomain := infraCluster.Spec.BaseDomain
    clusterName := infraCluster.Spec.ClusterName
    
    return fmt.Sprintf(`
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    hosts {
        fallthrough
    }
    forward . %s
    cache 30
    loop
    reload
    loadbalance
}

%s.%s:53 {
    errors
    cache 30
    hosts {
        fallthrough
    }
    forward . 8.8.8.8 8.8.4.4
}

api.%s.%s:53 {
    errors
    cache 30
    hosts {
        %s api.%s.%s
        fallthrough
    }
}

*.apps.%s.%s:53 {
    errors
    cache 30
    hosts {
        %s apps.%s.%s
        fallthrough
    }
}
`, 
        strings.Join(infraCluster.Spec.DNS.UpstreamServers, " "),
        clusterName, baseDomain,
        clusterName, baseDomain, infraCluster.Spec.Network.VIP, clusterName, baseDomain,
        clusterName, baseDomain, infraCluster.Spec.Network.VIP, clusterName, baseDomain,
    )
}

func (c *DNSController) generateHostsFile(infraCluster *infrastructurev1beta1.InfrastructureCluster) string {
    var hosts strings.Builder
    
    for _, node := range infraCluster.Spec.Nodes {
        hosts.WriteString(fmt.Sprintf("%s %s\n", node.IPAddress, node.Hostname))
    }
    
    return hosts.String()
}
```

#### 3.3 DHCP Controller实现

```go
package controller

import (
    "context"
    "fmt"
    
    infrastructurev1beta1 "github.com/bocloud/infrastructure-operator/api/v1beta1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type DHCPController struct {
    client.Client
    Scheme *runtime.Scheme
}

func (c *DHCPController) Reconcile(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    if !infraCluster.Spec.Services.DHCP.Enabled {
        return nil
    }
    
    if err := c.createDHCPConfigMap(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create DHCP ConfigMap: %w", err)
    }
    
    if err := c.createDHCPDeployment(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create DHCP Deployment: %w", err)
    }
    
    return nil
}

func (c *DHCPController) createDHCPDeployment(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-dhcp", infraCluster.Name),
            Namespace: infraCluster.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: infrastructurev1beta1.GroupVersion.String(),
                    Kind:       "InfrastructureCluster",
                    Name:       infraCluster.Name,
                    UID:        infraCluster.UID,
                },
            },
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: pointer.Int32Ptr(1),
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app": fmt.Sprintf("%s-dhcp", infraCluster.Name),
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app": fmt.Sprintf("%s-dhcp", infraCluster.Name),
                    },
                },
                Spec: corev1.PodSpec{
                    HostNetwork: true,
                    Containers: []corev1.Container{
                        {
                            Name:  "dhcp",
                            Image: "networkboot/dhcpd:latest",
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "config",
                                    MountPath: "/etc/dhcp",
                                },
                            },
                            SecurityContext: &corev1.SecurityContext{
                                Capabilities: &corev1.Capabilities{
                                    Add: []corev1.Capability{"NET_ADMIN"},
                                },
                            },
                        },
                    },
                    Volumes: []corev1.Volume{
                        {
                            Name: "config",
                            VolumeSource: corev1.VolumeSource{
                                ConfigMap: &corev1.ConfigMapVolumeSource{
                                    LocalObjectReference: corev1.LocalObjectReference{
                                        Name: fmt.Sprintf("%s-dhcp", infraCluster.Name),
                                    },
                                },
                            },
                        },
                    },
                },
            },
        },
    }
    
    return c.Create(ctx, deployment)
}
```
#### 3.4 Ignition Server实现
```go
package controller

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    
    infrastructurev1beta1 "github.com/bocloud/infrastructure-operator/api/v1beta1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type IgnitionController struct {
    client.Client
    Scheme *runtime.Scheme
}

func (c *IgnitionController) Reconcile(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    if err := c.createIgnitionConfigs(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create Ignition configs: %w", err)
    }
    
    if err := c.createIgnitionServer(ctx, infraCluster); err != nil {
        return fmt.Errorf("failed to create Ignition server: %w", err)
    }
    
    return nil
}

func (c *IgnitionController) createIgnitionConfigs(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    for _, node := range infraCluster.Spec.Nodes {
        ignitionConfig := c.generateNodeIgnitionConfig(infraCluster, node)
        
        configJSON, err := json.Marshal(ignitionConfig)
        if err != nil {
            return fmt.Errorf("failed to marshal Ignition config: %w", err)
        }
        
        configMap := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      fmt.Sprintf("%s-ignition-%s", infraCluster.Name, node.Hostname),
                Namespace: infraCluster.Namespace,
                OwnerReferences: []metav1.OwnerReference{
                    {
                        APIVersion: infrastructurev1beta1.GroupVersion.String(),
                        Kind:       "InfrastructureCluster",
                        Name:       infraCluster.Name,
                        UID:        infraCluster.UID,
                    },
                },
            },
            Data: map[string]string{
                "ignition.json": string(configJSON),
            },
        }
        
        if err := c.Create(ctx, configMap); err != nil {
            return fmt.Errorf("failed to create Ignition ConfigMap: %w", err)
        }
    }
    
    return nil
}

func (c *IgnitionController) createIgnitionServer(ctx context.Context, infraCluster *infrastructurev1beta1.InfrastructureCluster) error {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-ignition-server", infraCluster.Name),
            Namespace: infraCluster.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: pointer.Int32Ptr(2),
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app": fmt.Sprintf("%s-ignition-server", infraCluster.Name),
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app": fmt.Sprintf("%s-ignition-server", infraCluster.Name),
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "ignition-server",
                            Image: "nginx:alpine",
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: 80,
                                },
                            },
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "ignition-configs",
                                    MountPath: "/usr/share/nginx/html",
                                },
                            },
                        },
                    },
                    Volumes: []corev1.Volume{
                        {
                            Name: "ignition-configs",
                            VolumeSource: corev1.VolumeSource{
                                EmptyDir: &corev1.EmptyDirVolumeSource{},
                            },
                        },
                    },
                },
            },
        },
    }
    
    return c.Create(ctx, deployment)
}
```
### 4. 部署架构
#### 4.1 完整部署YAML示例
```yaml
apiVersion: infrastructure.bocloud.com/v1beta1
kind: InfrastructureCluster
metadata:
  name: openshift-prod
  namespace: infrastructure-system
spec:
  clusterName: ocp4
  baseDomain: example.com
  
  network:
    podCIDR: "10.128.0.0/14"
    serviceCIDR: "172.30.0.0/16"
    machineCIDR: "192.168.72.0/24"
    vip: "192.168.72.20"
  
  dns:
    upstreamServers:
      - "8.8.8.8"
      - "8.8.4.4"
    forwardRules:
      - domain: "internal.example.com"
        server: "192.168.1.10"
  
  dhcp:
    enabled: true
    range:
      start: "192.168.72.100"
      end: "192.168.72.200"
    gateway: "192.168.72.1"
    dnsServers:
      - "192.168.72.20"
  
  pxe:
    enabled: true
    tftpServer: "192.168.72.20"
    httpServer: "http://192.168.72.20:8080"
    bootImage: "rhcos-4.12.0-x86_64-metal.x86_64.raw.gz"
  
  ignition:
    serverURL: "http://192.168.72.20:8080/ignition"
  
  registry:
    url: "registry.example.com:5000"
    mirrorConfig:
      "quay.io/openshift-release-dev": "registry.example.com/openshift-release-dev"
      "registry.redhat.io": "registry.example.com/redhat"
  
  loadBalancer:
    type: metalLB
    ipPool:
      start: "192.168.72.20"
      end: "192.168.72.30"
  
  nodes:
    - hostname: bootstrap
      macAddress: "00:50:56:00:00:21"
      ipAddress: "192.168.72.21"
      role: bootstrap
      
    - hostname: master-0
      macAddress: "00:50:56:00:00:22"
      ipAddress: "192.168.72.22"
      role: master
      
    - hostname: master-1
      macAddress: "00:50:56:00:00:23"
      ipAddress: "192.168.72.23"
      role: master
      
    - hostname: master-2
      macAddress: "00:50:56:00:00:24"
      ipAddress: "192.168.72.24"
      role: master
      
    - hostname: worker-0
      macAddress: "00:50:56:00:00:25"
      ipAddress: "192.168.72.25"
      role: worker
  
  services:
    dns:
      enabled: true
      replicas: 2
    dhcp:
      enabled: true
    pxe:
      enabled: true
    registry:
      enabled: true
      storageSize: "500Gi"
    loadBalancer:
      enabled: true
```
### 5. 服务组件详细设计
#### 5.1 DNS服务
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ocp4-coredns
  namespace: infrastructure-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ocp4-coredns
  template:
    metadata:
      labels:
        app: ocp4-coredns
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: ocp4-coredns
              topologyKey: kubernetes.io/hostname
      containers:
        - name: coredns
          image: coredns/coredns:1.9.3
          args:
            - -conf
            - /etc/coredns/Corefile
          volumeMounts:
            - name: config
              mountPath: /etc/coredns
          ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 70Mi
            limits:
              cpu: 500m
              memory: 200Mi
      volumes:
        - name: config
          configMap:
            name: ocp4-coredns
---
apiVersion: v1
kind: Service
metadata:
  name: ocp4-dns
  namespace: infrastructure-system
spec:
  type: LoadBalancer
  selector:
    app: ocp4-coredns
  ports:
    - port: 53
      targetPort: 53
      protocol: UDP
      name: dns
    - port: 53
      targetPort: 53
      protocol: TCP
      name: dns-tcp
  loadBalancerIP: 192.168.72.20
```
#### 5.2 镜像仓库
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ocp4-registry
  namespace: infrastructure-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ocp4-registry
  template:
    metadata:
      labels:
        app: ocp4-registry
    spec:
      containers:
        - name: registry
          image: registry:2
          env:
            - name: REGISTRY_HTTP_ADDR
              value: ":5000"
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: registry-storage
              mountPath: /var/lib/registry
            - name: certs
              mountPath: /certs
            - name: auth
              mountPath: /auth
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
      volumes:
        - name: registry-storage
          persistentVolumeClaim:
            claimName: ocp4-registry-pvc
        - name: certs
          secret:
            secretName: ocp4-registry-certs
        - name: auth
          secret:
            secretName: ocp4-registry-auth
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ocp4-registry-pvc
  namespace: infrastructure-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: standard
```
#### 5.3 负载均衡器
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ocp4-ip-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.72.20-192.168.72.30
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: ocp4-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - ocp4-ip-pool
---
apiVersion: v1
kind: Service
metadata:
  name: ocp4-api
  namespace: infrastructure-system
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.72.20
  ports:
    - name: api
      port: 6443
      targetPort: 6443
      protocol: TCP
    - name: machine-config
      port: 22623
      targetPort: 22623
      protocol: TCP
  selector:
    app: ocp4-api-server
```
### 6. 与BKE集成设计
```yaml
apiVersion: bke.bocloud.com/v1beta1
kind: BKECluster
metadata:
  name: target-cluster
spec:
  infrastructureRef:
    apiVersion: infrastructure.bocloud.com/v1beta1
    kind: InfrastructureCluster
    name: openshift-prod
    namespace: infrastructure-system
  
  clusterConfig:
    addons:
      - name: infrastructure-operator
        version: v1.0.0
        type: chart
        param:
          manageDNS: "true"
          manageDHCP: "true"
          managePXE: "true"
          manageRegistry: "true"
          manageLoadBalancer: "true"
```
### 7. 优势对比
| 特性 | 传统Bastion节点 | 云原生方案 |
|------|----------------|-----------|
| **高可用** | 单点故障 | 多副本部署 |
| **可扩展性** | 手动扩展 | 自动扩缩容 |
| **配置管理** | 手动配置文件 | CRD声明式配置 |
| **版本控制** | 难以追踪 | GitOps友好 |
| **监控告警** | 需要额外部署 | 集成Prometheus |
| **日志收集** | 手动配置 | 自动收集 |
| **升级回滚** | 手动操作 | 滚动更新 |
| **资源利用** | 独占服务器 | 共享集群资源 |
| **成本** | 固定成本 | 按需使用 |
### 8. 实施路线图
```
Phase 1: 核心Operator开发 (2-3个月)
├── CRD定义和验证
├── Bastion Operator框架
├── DNS Controller
├── DHCP Controller
└── Ignition Controller

Phase 2: 服务组件集成 (1-2个月)
├── MetalLB集成
├── Harbor集成
├── PXE服务容器化
└── 监控告警集成

Phase 3: 高级功能 (1-2个月)
├── 高可用部署
├── 自动扩缩容
├── GitOps集成
└── 多集群管理

Phase 4: 生产就绪 (1个月)
├── 安全加固
├── 性能优化
├── 文档完善
└── 测试覆盖
```
这个云原生方案完全消除了对独立Bastion节点的依赖，所有服务都运行在Kubernetes中，实现了真正的云原生基础设施管理。



