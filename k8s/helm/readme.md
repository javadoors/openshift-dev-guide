# Helm
要快速掌握 **Helm**，可以从几个核心概念和常用命令入手。Helm 是 Kubernetes 的包管理工具，类似于 Linux 的 apt/yum，用来简化应用的安装、升级和管理。  

## 📌 核心概念
- **Chart**：Helm 的软件包，包含 Kubernetes 的 YAML 模板和配置。  
- **Release**：Chart 部署到集群后的实例。一个 Chart 可以部署多次，每次就是一个 Release。  
- **Repository**：存放 Charts 的仓库，类似于软件源。  
- **Values**：配置文件，用来覆盖 Chart 默认参数。  

## 📊 常用命令速记表

| 功能 | 命令 | 说明 |
|------|------|------|
| **添加仓库** | `helm repo add myrepo https://charts.example.com` | 添加一个 Chart 仓库 |
| **更新仓库** | `helm repo update` | 更新仓库索引 |
| **搜索 Chart** | `helm search repo nginx` | 在仓库中搜索 Chart |
| **安装应用** | `helm install myapp myrepo/nginx` | 安装 Chart，生成一个 Release |
| **查看 Release** | `helm list` | 查看已安装的 Release |
| **升级应用** | `helm upgrade myapp myrepo/nginx -f values.yaml` | 使用新配置升级 Release |
| **卸载应用** | `helm uninstall myapp` | 删除 Release |
| **查看历史** | `helm history myapp` | 查看 Release 的版本历史 |
| **回滚版本** | `helm rollback myapp 2` | 回滚到指定版本 |

## 🚀 快速学习路径
1. **安装 Helm**：在本地或 CI/CD 环境中安装。  
2. **添加仓库**：如官方 `bitnami` 仓库。  
3. **安装第一个应用**：例如 `helm install mynginx bitnami/nginx`。  
4. **修改配置**：通过 `values.yaml` 自定义参数。  
5. **升级与回滚**：掌握 `helm upgrade` 和 `helm rollback`。  
6. **编写自己的 Chart**：理解模板语法，创建可复用的应用包。  

## 🔑 关键结论
- Helm 的核心是 **Chart → Release → Values**。  
- 常用命令集中在 **安装、升级、卸载、回滚**。  
- 掌握 Helm 后，可以极大简化 Kubernetes 应用的部署与管理。  

 # Helm Chart
 **Helm Chart 的结构是一个目录树，核心文件包括 `Chart.yaml`、`values.yaml`、`templates/` 和 `charts/`，它们共同定义了应用的元数据、默认配置、Kubernetes 模板以及依赖关系。**  

## 📌 Helm Chart 目录结构解析

一个典型的 Chart 目录结构如下：  
```
mychart/
  Chart.yaml          # Chart 元数据（必需）
  values.yaml         # 默认配置值（必需）
  charts/             # 依赖的子 Chart（可选）
  templates/          # 模板文件目录（必需）
    NOTES.txt         # 使用说明（可选）
    _helpers.tpl      # 模板辅助函数（可选）
    deployment.yaml   # Kubernetes Deployment 模板
    service.yaml      # Kubernetes Service 模板
  LICENSE             # 许可证（可选）
  README.md           # 说明文档（可选）
  requirements.yaml   # 依赖声明（旧版，已弃用）
```
  [Helm](https://v2.helm.sh/docs/charts/)  

## 📊 核心文件说明

- **Chart.yaml**  
  - 必需文件，定义 Chart 的元数据。  
  - 包含字段：`apiVersion`、`name`、`version`、`description`、`appVersion`、`maintainers` 等。  
- **values.yaml**  
  - 默认配置文件，用户可通过 `-f values.yaml` 或 `--set` 覆盖。  
  - 用于控制模板渲染时的参数。  
- **templates/**  
  - 存放 Kubernetes YAML 模板文件。  
  - Helm 使用 Go 模板引擎渲染这些文件，并结合 `values.yaml` 生成最终的 Kubernetes 清单。  
  - 常见文件：`deployment.yaml`、`service.yaml`、`ingress.yaml`。  
- **charts/**  
  - 存放依赖的子 Chart。  
  - 例如，一个 Web 应用 Chart 可以依赖 MySQL Chart。  
- **NOTES.txt**  
  - 安装完成后显示的提示信息。  
  - 用于指导用户如何访问或使用部署的应用。  

## 📑 额外文件
- **README.md**：说明文档，帮助用户理解 Chart 的用途。  
- **LICENSE**：许可证文件。  
- **requirements.yaml**：旧版依赖声明文件（Helm v2 使用，v3 已弃用）。  

## 🔑 关键结论
- Helm Chart 的核心是 **Chart.yaml + values.yaml + templates/**。  
- `Chart.yaml` 定义元数据，`values.yaml` 提供默认配置，`templates/` 渲染 Kubernetes 清单。  
- `charts/` 用于依赖管理，`NOTES.txt` 提供安装提示。  

# SubChart
要快速掌握 **Helm Chart 的子 Chart**，你只需要理解它的作用、结构和使用方式。子 Chart 本质上是 **依赖的 Chart**，用于在一个主 Chart 中引入其他 Chart（例如数据库、缓存服务），从而实现复杂应用的模块化管理。  

## 📌 子 Chart 的作用
- **依赖管理**：主 Chart 可以依赖多个子 Chart，例如一个 Web 应用 Chart 依赖 MySQL Chart。  
- **模块化**：将不同功能拆分成独立 Chart，方便复用和维护。  
- **版本控制**：子 Chart 可以指定版本，保证依赖的稳定性。  
- **配置继承**：主 Chart 的 `values.yaml` 可以覆盖子 Chart 的配置。  

## 📊 子 Chart 的结构
在主 Chart 的目录下，子 Chart 通常放在 `charts/` 目录中：  
```
mychart/
  Chart.yaml
  values.yaml
  templates/
  charts/
    mysql/
      Chart.yaml
      values.yaml
      templates/
```
- **charts/**：存放子 Chart。  
- **requirements.yaml**（Helm v2）或 **Chart.yaml 的 dependencies 字段**（Helm v3）：声明依赖。  

## 📑 使用方式
- **声明依赖**（Helm v3 推荐方式）：  
  在主 Chart 的 `Chart.yaml` 中添加：  
  ```yaml
  dependencies:
    - name: mysql
      version: 8.8.26
      repository: https://charts.bitnami.com/bitnami
  ```
- **更新依赖**：  
  ```bash
  helm dependency update
  ```
- **覆盖子 Chart 配置**：  
  在主 Chart 的 `values.yaml` 中：  
  ```yaml
  mysql:
    auth:
      rootPassword: mypassword
  ```

## 📊 对比表：主 Chart vs 子 Chart

| 类型 | 作用范围 | 配置方式 | 示例 |
|------|----------|----------|------|
| **主 Chart** | 应用整体 | `values.yaml` | Web 应用 |
| **子 Chart** | 依赖模块 | 主 Chart 的 `values.yaml` 覆盖 | MySQL、Redis |

## 🔑 关键结论
- 子 Chart 是 **依赖的 Chart**，用于模块化和复用。  
- 在 Helm v3 中通过 `Chart.yaml` 的 `dependencies` 字段声明。  
- 主 Chart 可以覆盖子 Chart 的配置，实现统一管理。  

 #  Helm 主 Chart 与子 Chart 的依赖解析流程
 在 Helm 中，**子 Chart 的引用方式主要有两种**，而依赖的子 Chart 的解析流程也有一套固定机制。掌握这两点，就能快速理解 Helm 的依赖管理。  

## 📌 子 Chart 的引用方式
1. **直接嵌入**  
   - 将子 Chart 放在主 Chart 的 `charts/` 目录下。  
   - 主 Chart 安装时会自动加载该目录中的子 Chart。  
   - 常用于本地开发或自定义依赖。  

2. **依赖声明**  
   - 在主 Chart 的 `Chart.yaml` 中通过 `dependencies` 字段声明：  
     ```yaml
     dependencies:
       - name: mysql
         version: 8.8.26
         repository: https://charts.bitnami.com/bitnami
     ```  
   - 然后运行：  
     ```bash
     helm dependency update
     ```  
   - Helm 会下载依赖 Chart 到 `charts/` 目录。  
   - 常用于官方或第三方仓库的依赖管理。  

## 📊 子 Chart 的解析流程
Helm 在安装主 Chart 时，解析子 Chart 的流程大致如下：  
1. **读取依赖声明**  
   - 从 `Chart.yaml` 的 `dependencies` 字段或 `charts/` 目录中获取子 Chart 信息。  
2. **下载或加载子 Chart**  
   - 如果是远程仓库依赖，执行 `helm dependency update` 时会下载到 `charts/`。  
   - 如果是本地嵌入，则直接读取 `charts/` 下的目录。  
3. **合并配置**  
   - 主 Chart 的 `values.yaml` 可以覆盖子 Chart 的配置。  
   - 例如：  
     ```yaml
     mysql:
       auth:
         rootPassword: mypassword
     ```  
     会覆盖子 Chart `mysql` 的默认值。  
4. **渲染模板**  
   - Helm 使用 Go 模板引擎渲染主 Chart 和子 Chart 的 `templates/`。  
   - 渲染时会将主 Chart 的 values 与子 Chart 的 values 合并。  
5. **生成 Kubernetes 清单**  
   - 主 Chart 与子 Chart 的模板最终合并为一组 Kubernetes YAML 文件，提交给 API Server。  

## 📑 总结
- **引用方式**：  
  - 本地嵌入 (`charts/` 目录)。  
  - 远程依赖 (`Chart.yaml` → `dependencies` → `helm dependency update`)。  
- **解析流程**：  
  - 读取依赖 → 下载/加载 → 合并配置 → 渲染模板 → 生成清单。  

## 🔑 关键结论
- 子 Chart 是 Helm 的依赖管理机制，支持本地嵌入和远程声明两种方式。  
- 主 Chart 的 `values.yaml` 可以覆盖子 Chart 的配置，实现统一管理。  
- Helm 的解析流程保证了主 Chart 与子 Chart 的模板最终合并为一个完整的 Kubernetes 部署清单。  

# 
