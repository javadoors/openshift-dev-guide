# OCI Confi
在 **OCI (Open Container Initiative)** 中，容器镜像的配置文件是 `config.json`，它定义了容器运行时的行为。这个文件通常由 `runc` 或兼容的运行时使用，遵循 OCI Runtime Specification。

## 核心内容
- **`ociVersion`**：指定遵循的 OCI Runtime 版本。  
- **`process`**：定义容器内进程的执行方式，包括 `args`、`env`、`cwd`、`capabilities` 等。  
- **`root`**：指定容器根文件系统路径。  
- **`mounts`**：挂载点配置，如 `/proc`、`/sys`、`/dev`。  
- **`linux`**：Linux 特定配置，如 `namespaces`、`cgroups`、`seccomp`。  
- **`hooks`**：生命周期钩子（prestart、poststart、poststop）。  

## 示例配置（简化版）
```json
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": true,
    "user": { "uid": 0, "gid": 0 },
    "args": ["sh"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "mounts": [
    { "destination": "/proc", "type": "proc", "source": "proc" },
    { "destination": "/dev", "type": "tmpfs", "source": "tmpfs", "options": ["nosuid","strictatime","mode=755","size=65536k"] }
  ],
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "ipc" },
      { "type": "uts" },
      { "type": "mount" }
    ]
  }
}
```

## 使用场景
- **容器运行时**：如 `runc`、`crun`，直接读取 `config.json` 启动容器。  
- **Kubernetes CRI**：底层容器运行时遵循 OCI 配置。  
- **安全隔离**：通过 `linux.namespaces`、`seccomp`、`cgroups` 控制隔离与资源限制。  

## 最佳实践
- **保持最小化配置**：只挂载必要文件系统，减少攻击面。  
- **使用只读 rootfs**：避免容器内修改基础镜像。  
- **启用 seccomp 与 capabilities**：限制系统调用与权限。  
- **版本兼容性**：确保 `ociVersion` 与运行时支持的版本一致。  

# cfg.Config.Labels的构建过程
## `cfg.Config.Labels` 的构建过程
在容器镜像的 **OCI 配置**中，`Config.Labels` 是一个 `map[string]string`，用于存放镜像的元数据标签。它的构建过程通常发生在 **镜像构建阶段**（例如使用 Dockerfile 或构建工具时），最终写入到镜像的 `config.json` 中。
## 构建过程的关键步骤
1. **在 Dockerfile 或构建工具中定义 Label**
   ```dockerfile
   FROM registry.example.com/base:latest
   LABEL release-spec='{"version":"v2.6.0","components":[{"name":"kubernetes","version":"v1.29.0"}]}'
   ```
   - 这里的 `LABEL` 指令会把键值对写入镜像配置的 `Config.Labels` 字段。
2. **镜像构建时写入 OCI Config**
   - 构建工具（如 Docker、Buildah、Kaniko）会解析 Dockerfile，把 `LABEL` 指令转化为镜像配置 JSON 中的 `config.Labels`。
   - 最终镜像的 `config.json` 文件里会出现：
     ```json
     {
       "config": {
         "Labels": {
           "release-spec": "{\"version\":\"v2.6.0\",\"components\":[...]}"
         }
       }
     }
     ```
3. **镜像推送与存储**
   - 当镜像被推送到仓库（如 Harbor、Quay、Docker Hub），这些 Labels 会随镜像一起存储。
   - 拉取镜像时，客户端或 API 可以读取这些 Labels。
4. **运行时读取**
   - 在 Go 代码中，通过 `img.ConfigFile()` 获取镜像配置，再访问 `cfg.Config.Labels["release-spec"]`，即可得到构建时写入的 JSON 字符串。
## 最佳实践
- **结构化 JSON**：像 `release-spec` 这种复杂信息，建议用 JSON 存储，而不是简单字符串。  
- **不可变性**：Label 内容应与镜像内容严格对应，避免后期修改导致不一致。  
- **自动生成**：在 CI/CD 流水线中自动生成 `release-spec`，确保版本号与组件信息准确。  
- **校验与签名**：在构建后对 Label 内容做校验，甚至签名，保证可信性。  
- **统一入口**：所有升级逻辑都从 Label 中读取，不依赖外部散落的配置。  
## 总结
`cfg.Config.Labels["release-spec"]` 的构建过程就是：**在镜像构建阶段通过 Dockerfile 或构建工具写入 Label → 镜像配置文件保存 → 推送到仓库 → 运行时读取并解析为结构体**。  
它的价值在于：让镜像自带完整的发布说明，成为升级与版本管理的单一可信源。

## `cfg.Config.Labels["release-spec"]` 的作用
在容器镜像的 **Config.Labels** 中，`release-spec` 通常存放一段 **JSON 字符串**，描述该镜像的 **发布元信息**。  
在你代码里，它被反序列化为 `ReleaseImage` 结构体，用来承载镜像的版本、组件清单等信息。

**作用可以总结为：**
- **嵌入元数据**：把发布规范（release spec）直接嵌入镜像标签，镜像本身就携带了版本说明。  
- **自描述镜像**：任何人只要拉取镜像，就能通过 `Config.Labels["release-spec"]` 获取完整的组件版本信息。  
- **驱动升级逻辑**：像 OpenShift 的 CVO 就是通过解析 release-spec 来决定如何升级集群。  
## 最佳实践
- **保持 JSON 格式清晰**：release-spec 应该是结构化 JSON，包含版本号、组件列表、依赖关系。  
- **不可变性**：release-spec 应该与镜像内容严格对应，避免后期修改导致镜像与 spec 不一致。  
- **版本化**：在 release-spec 中明确标注整体版本（如 `v2.6.0`）和各组件版本，方便比对。  
- **校验与签名**：在 CI/CD 中对 release-spec 做校验，甚至签名，保证镜像元数据可信。  
- **兼容性字段**：保留必要的兼容字段，避免升级时因字段缺失或重命名导致解析失败。  
- **统一入口**：所有升级逻辑都从 release-spec 读取，不依赖外部散落的配置，保证一致性。  
## 示例（典型 release-spec JSON）
```json
{
  "version": "v2.6.0",
  "components": [
    {"name": "kubernetes", "type": "composite", "version": "v1.29.0"},
    {"name": "etcd", "type": "leaf", "version": "v3.5.12"},
    {"name": "bkeagent", "type": "leaf", "version": "v2.6.0"},
    {"name": "provider-upgrade", "type": "inline", "version": "v1.2.0"},
    {"name": "component-upgrade", "type": "inline", "version": "v1.1.0"}
  ]
}
```
## 总结
`cfg.Config.Labels["release-spec"]` 的核心价值在于：**让镜像自带完整的发布说明，成为升级与版本管理的单一可信源**。  
最佳实践是：**保持 release-spec 的结构化、不可变、版本化，并在 CI/CD 中自动生成与校验**。

# **release-spec 字段设计规范表**
帮助团队统一镜像中 `release-spec` Label 的结构与含义。
## Release-Spec 字段设计规范表
| 字段名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| **version** | string | ✅ | 整体发布版本号，标识该镜像的 Release 版本 | `"v2.6.0"` |
| **components** | array | ✅ | 组件清单，每个元素描述一个组件的元信息 | `[ {...}, {...} ]` |
| **components[].name** | string | ✅ | 组件名称，唯一标识该组件 | `"kubernetes"` |
| **components[].type** | string (enum) | ✅ | 组件类型：`composite`（复合）、`leaf`（叶子）、`inline`（内联） | `"leaf"` |
| **components[].version** | string | ✅ | 组件版本号，遵循语义化版本或特定格式 | `"v1.29.0"` |
| **components[].source** | string | ⬜ | 组件来源（可选），如镜像仓库地址或 Git commit | `"quay.io/org/kubernetes@sha256:..."` |
| **components[].dependencies** | array[string] | ⬜ | 依赖的其他组件名称列表 | `["etcd","cni"]` |
| **metadata** | object | ⬜ | 发布的额外元信息，如构建时间、作者、签名信息 | `{ "buildDate":"2026-05-09T08:00:00Z", "author":"release-bot" }` |
| **annotations** | object | ⬜ | 自定义扩展字段，供特定平台或工具使用 | `{ "upgrade-strategy":"rolling" }` |
## 设计原则与最佳实践
- **结构化 JSON**：保证字段清晰，避免混乱的字符串拼接。  
- **不可变性**：`release-spec` 必须与镜像内容严格对应，构建后不可随意修改。  
- **版本化**：顶层 `version` 字段必须存在，且遵循语义化版本。  
- **可扩展性**：通过 `metadata` 与 `annotations` 提供扩展空间，避免破坏核心结构。  
- **依赖声明**：复杂组件应在 `dependencies` 中明确依赖关系，方便升级与兼容性检查。  
- **校验与签名**：在 CI/CD 中对 `release-spec` 做 schema 校验，并可选地加入签名字段保证可信性。  
## 示例 JSON
```json
{
  "version": "v2.6.0",
  "components": [
    { "name": "kubernetes", "type": "composite", "version": "v1.29.0" },
    { "name": "etcd", "type": "leaf", "version": "v3.5.12" },
    { "name": "bkeagent", "type": "leaf", "version": "v2.6.0" },
    { "name": "provider-upgrade", "type": "inline", "version": "v1.2.0" },
    { "name": "component-upgrade", "type": "inline", "version": "v1.1.0" }
  ],
  "metadata": {
    "buildDate": "2026-05-09T08:00:00Z",
    "author": "release-bot"
  },
  "annotations": {
    "upgrade-strategy": "rolling"
  }
}
```

这样一份规范表可以作为团队的 **契约文档**，保证所有镜像的 `release-spec` Label 一致、可解析、可验证。  
