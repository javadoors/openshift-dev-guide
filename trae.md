# **Agent 规范**
帮助你在设计和使用 Trae Agent 时保持统一性和可维护性。  
## 📌 Agent 规范核心要素
### 1. 基本字段
- **name**：Agent 的唯一标识（如 `qa`、`doc`、`code`）。  
- **type**：Agent 的类别（问答、文档、代码、对话、测试等）。  
- **description**：简要说明用途和定位。  
- **input**：定义输入类型（文本、代码、上下文标签）。  
- **output**：定义输出类型（答案、文档、代码片段、对话、测试用例）。  
- **example**：典型调用方式，结合 `@agent` 和 `#context`。  
### 2. 规范设计原则
- **命名规范**：简短、直观，避免歧义（如 `qa` 而不是 `questionAgent`）。  
- **职责单一**：每个 Agent 专注于一个任务类型，避免混合职责。  
- **上下文扩展**：通过 `#` 标签增加主题或背景，而不是在 Agent 内部硬编码。  
- **可组合性**：支持 `@agent + #context + prompt` 的组合调用。  
- **可维护性**：输入输出类型明确，方便后续扩展或替换。  
### 3. 示例规范模板
```yaml
agent:
  name: qa
  type: question-answering
  description: 用于知识问答和解释说明
  input: text (问题)
  output: text (答案)
  example: "@qa #kubernetes 请解释 K3s 架构"
```

```yaml
agent:
  name: doc
  type: documentation
  description: 用于生成说明文档、流程指南
  input: text (主题 + 上下文标签)
  output: structured text (文档)
  example: "@doc #rpm #patch 生成 RPM Patch 制作流程文档"
```

```yaml
agent:
  name: code
  type: code-generation
  description: 用于编写脚本、函数、配置文件
  input: text (编程语言 + 功能描述)
  output: code snippet
  example: "@code #python 写一个合并图片的脚本"
```

```yaml
agent:
  name: chat
  type: conversational
  description: 用于交互式解释、讨论
  input: text (问题或话题)
  output: conversational text
  example: "@chat #k3s 解释 K3s 与 K8s 区别"
```

```yaml
agent:
  name: test
  type: testing
  description: 用于生成测试用例、验证逻辑
  input: text (功能描述)
  output: test cases / validation script
  example: "@test #api 为用户登录接口生成测试用例"
```
## ✅ 总结
- **Agent 规范**的核心是：**统一字段、职责单一、上下文可扩展、输入输出明确**。  
- 通过 `@agent` 指定角色，`#context` 增加背景，保持调用方式简洁一致。  
- 模板化配置让不同 Agent 的创建和维护更高效。  

# Trae 插件
要把上面定义好的 **QA、Doc、Code、Chat、Test Agent** 添加到 VSCode 的 **Trae 插件**，通常需要在插件的配置文件（例如 `trae.json` 或 `.trae/config.yaml`）中声明这些 Agent。下面给你一个统一的配置模板示例：  
## 📌 VSCode Trae 插件 Agent 配置示例

```yaml
agents:
  - name: qa
    type: question-answering
    description: 用于知识问答和解释说明
    input: text
    output: text
    example: "@qa #kubernetes 请解释 K3s 架构"

  - name: doc
    type: documentation
    description: 用于生成说明文档、流程指南
    input: text
    output: structured text
    example: "@doc #rpm #patch 生成 RPM Patch 制作流程文档"

  - name: code
    type: code-generation
    description: 用于编写脚本、函数、配置文件
    input: text
    output: code snippet
    example: "@code #python 写一个合并图片的脚本"

  - name: chat
    type: conversational
    description: 用于交互式解释、讨论
    input: text
    output: conversational text
    example: "@chat #k3s 解释 K3s 与 K8s 区别"

  - name: test
    type: testing
    description: 用于生成测试用例、验证逻辑
    input: text
    output: test cases
    example: "@test #api 为用户登录接口生成测试用例"
```
## 🧩 使用方式
1. 在 VSCode 中安装并启用 **Trae 插件**。  
2. 在项目根目录或用户配置目录下创建/编辑 `trae.yaml` 或 `trae.json`。  
3. 将上述配置粘贴进去，保存。  
4. 在 VSCode 命令面板或编辑器中直接使用：
   - `@qa #kubernetes 请解释 K3s 架构`
   - `@code #python 写一个合并图片的脚本`

✅ **总结**：通过在 Trae 插件配置文件中添加这些 Agent，你就能在 VSCode 中快速调用不同类型的 Agent，结合 `@agent` 和 `#context` 标签实现精准的 Prompt 控制。  

# Trae IDE Skills 编写指南
## 一、什么是 Skill？
Skill（技能）是一份清晰、严谨、可执行的指令文档，用于扩展 AI 的专业能力。它明确在**什么条件下**，按照**哪些步骤**，产出**什么结果**。
### Skill 与其他概念对比
| 特性 | Skills | Sub-Agents | MCP |
|------|--------|------------|-----|
| **目的** | 用专业知识、工作流程扩展 AI | 生成自主代理处理复杂子任务 | 连接外部工具和数据源 |
| **调用方式** | 模型自动发现（基于上下文） | 父代理显式生成 | MCP 服务器工具调用 |
| **复杂度** | 低（只需 SKILL.md + 可选文件） | 中等（需要编排） | 中-高（需要服务器设置） |
| **最适合** | 领域专业知识、工作流程、模板 | 并行任务、研究、探索 | 外部 API、数据库、第三方服务 |
## 二、Skill 目录结构
```
skill-name/
├── SKILL.md              # ✅ 必需：元数据 + 核心指令
├── README.md             # 可选：使用说明
├── templates/            # 可选：可复用的模板
│   └── component.tsx.md
├── examples/             # 可选：输入/输出示例
│   ├── input.md
│   └── output.md
├── references/           # 可选：规则参考文档
│   └── rules.md
└── scripts/              # 可选：可执行脚本
    ├── task.py
    └── task.sh
```
## 三、SKILL.md 文件格式
SKILL.md 是 Skill 的核心文件，由 **YAML 头部元数据** 和 **Markdown 正文** 两部分组成：
### 完整模板
```markdown
---
name: 技能名称
description: 简要描述这个技能的功能和使用场景
version: 1.0.0
author: 作者名称（可选）
tags:
  - 标签1
  - 标签2
---

# 技能名称

## 1. 技能概述

**技能名称**: skill_name
**技能版本**: 1.0.0
**技能描述**: 简短的技能描述
**适用场景**: 适用的场景说明

**触发条件**:
- 触发条件1
- 触发条件2
- 当用户打开或编辑某些文件时

**触发关键词**:
- 关键词1
- 关键词2
- 关键词3

## 2. 文档导航

| 文档名称 | 描述 | 路径 |
|---------|------|------|
| 文档1 | 描述 | [链接](path/doc.md) |

## 3. 核心功能

详细的功能说明...

## 4. 使用示例

代码示例或使用示例...
```
## 四、可定义的属性详解
### 1. YAML 头部元数据
| 属性 | 必需 | 说明 |
|------|------|------|
| `name` | ✅ | 技能名称，建议使用小写和连字符（如 `vue-component-guidelines`） |
| `description` | ✅ | 简要描述技能的功能和使用场景 |
| `version` | 可选 | 技能版本号 |
| `author` | 可选 | 作者信息 |
| `tags` | 可选 | 标签列表，用于分类 |

**示例**:
```yaml
---
name: vue-component-guidelines
description: 为Vue项目创建组件时自动应用团队编码规范
version: 1.0.0
tags:
  - vue
  - frontend
  - code-standards
---
```
### 2. Markdown 正文属性
| 属性 | 说明 |
|------|------|
| **技能概述** | 技能的基本信息 |
| **触发条件** | 明确说明什么情况下使用此 Skill |
| **触发关键词** | 列出相关的关键词，用于自动匹配 |
| **文档导航** | 提供清晰的文档链接表格 |
| **核心功能** | 详细的功能说明和操作步骤 |
| **使用示例** | 实际的使用示例，包含输入输出 |
## 五、创建 Skill 的四种方式
### 方式 A：设置页面手动创建
1. 点击 Trae 右上角 ⚙️ 设置按钮
2. 选择「规则和技能」选项
3. 点击技能栏的「创建」按钮
4. 填写：技能类型（全局/项目）、技能名称、描述、指令
5. 点击确认
### 方式 B：上传 Skill 文件
1. 准备 SKILL.md 文件或包含该文件的 .zip 压缩包
2. 在设置面板点击「上传进行智能解析」
3. 选择文件并确认
### 方式 C：通过对话让 AI 创建
直接向 AI 描述需求：
```
帮我在 .trae/skills 目录下创建一个新的技能
技能的名字叫 xxx
这个技能可以帮我做以下事情：
- xxx
- xxx
```
### 方式 D：从 GitHub 导入
1. 从 GitHub 下载 SKILL.md 文件
2. 通过上传方式导入
## 六、实战示例
### 示例 1：Vue 组件开发规范
```markdown
---
name: vue-component-guidelines
description: Vue组件开发规范，确保代码风格一致
---

# Vue组件开发规范

## 触发条件
- 当用户创建或修改 Vue 组件文件时
- 当用户询问 Vue 组件开发规范时
- 当用户请求代码审查时

## 触发关键词
- Vue组件
- 组件开发
- Vue规范

## 文件命名
- 组件文件使用 PascalCase，如 `UserProfile.vue`
- 工具函数文件使用 kebab-case，如 `user-utils.ts`

## 组件结构
- 单文件组件必须包含三个块：`<template>`、`<script>`、`<style>`
- 使用 `<script setup>` 语法

## CSS 类命名
- 使用 kebab-case，如 `user-profile`、`btn-primary`
- 组件内样式使用作用域（`<style scoped>`）

## 注释要求
- 在组件顶部添加用途说明注释
- 对复杂业务逻辑添加中文注释
```
### 示例 2：Git 提交规范
```markdown
---
name: git-commit-guidelines
description: 生成符合 Conventional Commits 规范的提交信息
---

# Git 提交规范

## 触发条件
- 当用户执行 git commit 时
- 当用户询问提交信息格式时

## 提交格式
遵循 `<type>(<scope>): <subject>` 格式

## Type 类型
- `feat`: 新功能
- `fix`: 修复 bug
- `docs`: 文档变更
- `style`: 代码格式（不影响代码运行）
- `refactor`: 重构
- `test`: 测试相关
- `chore`: 构建过程或辅助工具

## 示例
```
feat(user): 添加用户登录功能
fix(api): 修复用户列表分页问题
docs(readme): 更新安装说明
```
```
## 七、Skill 加载机制
Skill 采用三层分级加载机制：

| 加载级别 | 加载时机 | 内容 | Token 消耗 |
|---------|---------|------|-----------|
| L1：元数据 | Agent 启动时 | YAML 格式的名称、描述 | ~100 Token |
| L2：说明文档 | 技能被触发时 | SKILL.md 工作流程、指南 | <5000 Token |
| L3：资源文件 | 按需加载 | 脚本、配置文件等 | 几乎无限制 |
## 八、编写优质 Skill 的原则
### 1. 单一职责
- ✅ 专注解决一个具体问题
- ❌ 不要试图包揽多任务
### 2. 描述清晰
- ✅ 明确输入输出（如 "根据城市和日期查天气"）
- ❌ 避免术语堆砌、描述模糊
### 3. 参数精简
- ✅ 参数命名语义化，配示例说明
- ❌ 参数繁杂、命名混乱
### 4. 可组合性
- ✅ 输出可作为其他 Skill 的输入
- ❌ 功能封闭，难以联动
## 九、Skill 存放位置
| 类型 | 路径 | 说明 |
|------|------|------|
| **项目级 Skill** | `.trae/skills/{skill_name}/` | 仅当前项目生效 |
| **全局 Skill** | Trae 设置中配置 | 适用于所有项目 |
## 十、相关资源
- **官方文档**: https://docs.trae.ai/ide/skills
- **Skill 下载库**: https://skills.sh/
- **Skill 市场**: https://skillstore.io/zh-hans/skills
- **GitHub 资源**: https://github.com/libukai/awesome-agent-skills

# 代码重构技能
## 1. 技能概述
**技能名称**: refactor-code
**技能版本**: 1.0.0
**技能描述**: 提供系统化的代码重构方法和最佳实践，帮助改善代码质量、可读性和可维护性
**适用场景**: 代码重构、代码优化、技术债务清理、代码审查改进

**触发条件**:
- 当用户请求重构代码时
- 当用户询问如何改进代码结构时
- 当用户提到"重构"、"优化代码"、"改善代码"等关键词时
- 当用户请求代码审查并提出改进建议时

**触发关键词**:
- 重构
- refactor
- 优化代码
- 改善代码
- 代码质量
- 技术债务
- 代码异味
- 简化代码
## 2. 重构原则
### 2.1 核心原则
1. **小步重构**: 每次只做一个小改动，确保每步都可测试
2. **保持行为不变**: 重构不应改变代码的外部行为
3. **频繁测试**: 每次重构后运行测试确保功能正常
4. **版本控制**: 每个重构步骤独立提交，便于回滚
### 2.2 重构时机
- 代码重复（DRY 原则违反）
- 函数过长（超过 50 行）
- 类过大（职责过多）
- 参数列表过长（超过 3-4 个参数）
- 嵌套过深（超过 3 层）
- 命名不清晰
- 注释过多（代码应该自解释）
## 3. 重构方法清单
### 3.1 函数级别重构
| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 提取函数 | 将代码片段提取为独立函数 | 代码重复、函数过长 |
| 内联函数 | 将函数体直接嵌入调用处 | 函数体很简单、函数名不能很好地表达意图 |
| 提取变量 | 将表达式提取为变量 | 复杂表达式、表达式需要解释 |
| 内联变量 | 将变量替换为表达式 | 变量名不能很好地表达意图 |
| 重命名函数/变量 | 使用更有意义的名称 | 命名不清晰 |
| 以查询取代临时变量 | 将临时变量提取为函数 | 临时变量导致函数过长 |
| 分解条件表达式 | 将复杂的条件逻辑提取为函数 | 条件表达式复杂 |
### 3.2 类级别重构
| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 提取类 | 将部分功能提取为新类 | 类职责过多 |
| 内联类 | 将类的功能合并到另一个类 | 类功能很少 |
| 提取接口 | 提取公共接口 | 多个类有相似行为 |
| 以对象取代基本类型 | 将基本类型封装为对象 | 基本类型需要额外行为 |
| 以工厂函数取代构造函数 | 使用工厂方法创建对象 | 创建逻辑复杂 |
### 3.3 代码组织重构
| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 移动函数 | 将函数移到更合适的类 | 函数与另一个类交互更多 |
| 移动字段 | 将字段移到更合适的类 | 字段被另一个类更多使用 |
| 封装字段 | 将公共字段改为私有并提供访问器 | 需要控制字段访问 |
| 隐藏委托关系 | 在类中添加委托方法 | 客户端过多了解委托关系 |
### 3.4 条件逻辑重构
| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 分解条件表达式 | 提取条件判断为函数 | 条件表达式复杂 |
| 合并条件表达式 | 合并多个条件判断 | 多个条件有相同结果 |
| 以多态取代条件表达式 | 使用多态替代条件判断 | 有多个条件分支基于类型 |
| 引入空对象 | 使用空对象替代 null 检查 | 多处需要 null 检查 |
### 3.5 数据结构重构
| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 以对象取代数组 | 将数组元素转为对象字段 | 数组元素含义不明确 |
| 以对象取代记录 | 将记录结构转为对象 | 需要添加行为 |
| 以类取代类型码 | 将类型码转为类 | 类型码需要额外行为 |
## 4. 重构工作流程
### 4.1 标准重构流程
```
1. 识别问题
   ├── 分析代码异味
   ├── 确定重构目标
   └── 评估影响范围

2. 准备重构
   ├── 确保有测试覆盖
   ├── 创建备份/分支
   └── 规划重构步骤

3. 执行重构
   ├── 小步修改
   ├── 每步后运行测试
   └── 提交每个独立改动

4. 验证结果
   ├── 运行完整测试套件
   ├── 检查代码质量指标
   └── 代码审查
```
### 4.2 重构检查清单
**重构前**:
- [ ] 是否有足够的测试覆盖？
- [ ] 是否理解当前代码的行为？
- [ ] 是否有清晰的改进目标？
- [ ] 是否创建了备份或分支？

**重构中**:
- [ ] 每步改动是否足够小？
- [ ] 每步后是否运行了测试？
- [ ] 是否保持了代码可编译/可运行？
- [ ] 是否更新了相关文档？

**重构后**:
- [ ] 所有测试是否通过？
- [ ] 代码行为是否保持不变？
- [ ] 代码是否更易理解？
- [ ] 是否有性能回归？
## 5. 代码异味识别
### 5.1 常见代码异味
| 异味类型 | 症状 | 建议重构 |
|---------|------|---------|
| 重复代码 | 相同/相似代码出现在多处 | 提取函数、模板方法 |
| 过长函数 | 函数超过 50 行 | 提取函数、分解职责 |
| 过大类 | 类超过 500 行或职责过多 | 提取类、单一职责 |
| 过长参数列表 | 参数超过 3-4 个 | 引入参数对象、使用构建器 |
| 发散式变化 | 一个类因多种原因变化 | 提取类 |
| 霰弹式修改 | 一个变化导致多处修改 | 移动函数/字段 |
| 依恋情结 | 过多使用其他类的数据 | 移动函数 |
| 数据泥团 | 多个数据总是一起出现 | 提取类 |
| 基本类型偏执 | 过多使用基本类型 | 引入值对象 |
| switch 语句 | 大量 switch/case | 以多态取代 |
| 注释过多 | 需要大量注释解释代码 | 提取函数、重命名 |
### 5.2 代码异味检测模式
```go
// 示例：检测过长函数
func detectLongFunction(linesOfCode int, threshold int) bool {
    return linesOfCode > threshold
}

// 示例：检测参数过多
func detectLongParameterList(paramCount int, threshold int) bool {
    return paramCount > threshold
}

// 示例：检测嵌套深度
func detectDeepNesting(nestingLevel int, threshold int) bool {
    return nestingLevel > threshold
}
```
## 6. 重构示例
### 6.1 提取函数
**重构前**:
```go
func printOwing(invoice Invoice) {
    outstanding := 0.0
    
    fmt.Println("***********************")
    fmt.Println("*** Customer Owes ***")
    fmt.Println("***********************")
    
    for _, order := range invoice.Orders {
        outstanding += order.Amount
    }
    
    fmt.Printf("name: %s\n", invoice.Customer)
    fmt.Printf("amount: %.2f\n", outstanding)
}
```
**重构后**:
```go
func printOwing(invoice Invoice) {
    printBanner()
    outstanding := calculateOutstanding(invoice)
    printDetails(invoice.Customer, outstanding)
}

func printBanner() {
    fmt.Println("***********************")
    fmt.Println("*** Customer Owes ***")
    fmt.Println("***********************")
}

func calculateOutstanding(invoice Invoice) float64 {
    outstanding := 0.0
    for _, order := range invoice.Orders {
        outstanding += order.Amount
    }
    return outstanding
}

func printDetails(customer string, outstanding float64) {
    fmt.Printf("name: %s\n", customer)
    fmt.Printf("amount: %.2f\n", outstanding)
}
```
### 6.2 以多态取代条件表达式
**重构前**:
```go
func getBirdSpeed(bird Bird) float64 {
    switch bird.Type {
    case "European":
        return bird.BaseSpeed
    case "African":
        return bird.BaseSpeed - bird.LoadFactor
    case "NorwegianBlue":
        if bird.IsNailed {
            return 0
        }
        return bird.BaseSpeed
    default:
        return 0
    }
}
```
**重构后**:
```go
type Bird interface {
    GetSpeed() float64
}

type EuropeanBird struct {
    BaseSpeed float64
}

func (b EuropeanBird) GetSpeed() float64 {
    return b.BaseSpeed
}

type AfricanBird struct {
    BaseSpeed   float64
    LoadFactor  float64
}

func (b AfricanBird) GetSpeed() float64 {
    return b.BaseSpeed - b.LoadFactor
}

type NorwegianBlueBird struct {
    BaseSpeed float64
    IsNailed  bool
}

func (b NorwegianBlueBird) GetSpeed() float64 {
    if b.IsNailed {
        return 0
    }
    return b.BaseSpeed
}
```
### 6.3 引入参数对象
**重构前**:
```go
func amountInvoiced(start time.Time, end time.Time, customerID string, productID string) float64 {
    // ...
}

func amountReceived(start time.Time, end time.Time, customerID string, productID string) float64 {
    // ...
}

func amountOverdue(start time.Time, end time.Time, customerID string, productID string) float64 {
    // ...
}
```
**重构后**:
```go
type InvoiceQuery struct {
    Start      time.Time
    End        time.Time
    CustomerID string
    ProductID  string
}

func amountInvoiced(query InvoiceQuery) float64 {
    // ...
}

func amountReceived(query InvoiceQuery) float64 {
    // ...
}

func amountOverdue(query InvoiceQuery) float64 {
    // ...
}
```
## 7. 语言特定重构建议

### 7.1 Go 语言重构
- 使用 `gofmt` 保持代码格式一致
- 使用 `golint` 和 `staticcheck` 检测问题
- 利用 Go 的接口进行解耦
- 使用错误处理模式替代异常
- 优先组合而非继承
### 7.2 TypeScript/JavaScript 重构
- 使用 ES6+ 特性简化代码
- 利用 TypeScript 类型系统
- 使用解构和展开运算符
- 异步代码使用 async/await
- 模块化组织代码
### 7.3 Python 重构
- 遵循 PEP 8 编码规范
- 使用类型注解
- 利用装饰器提取横切关注点
- 使用上下文管理器管理资源
- 优先使用组合而非继承
## 8. 重构输出格式
当执行重构时，请按以下格式输出：
```markdown
## 重构分析

**检测到的问题**:
- [问题描述1]
- [问题描述2]

**建议的重构方法**:
1. [重构方法1]: [原因]
2. [重构方法2]: [原因]

## 重构方案

### 重构前
[原始代码]

### 重构后
[重构后代码]

### 改进说明
- [改进点1]
- [改进点2]

### 影响范围
- [影响的文件/模块]
- [需要更新的测试]
```
## 9. 注意事项
1. **不要在重构时添加新功能**: 重构和添加功能应该分开进行
2. **保持测试通过**: 每次小改动后都应确保测试通过
3. **关注性能**: 某些重构可能影响性能，需要评估
4. **文档同步**: 重构后更新相关文档和注释
5. **团队沟通**: 大型重构需要与团队沟通协调
6. **版本控制**: 每个重构步骤独立提交，便于追踪和回滚
## 10. 参考资源
| 资源 | 说明 |
|------|------|
| 《重构：改善既有代码的设计》 | Martin Fowler 经典著作 |
| 《代码整洁之道》 | Robert C. Martin |
| 《设计模式》 | GoF 经典模式书籍 |
| Refactoring Guru | https://refactoring.guru/ |
```
## 使用说明
这个 Skill 已经创建在 `.trae/skills/refactor-code/SKILL.md`。您可以通过以下方式触发它：

**触发方式**:
- 直接说 "重构这段代码"
- 说 "优化这个函数"
- 说 "改善代码结构"
- 说 "帮我重构"

**Skill 包含的内容**:
1. ✅ 系统化的重构方法清单（函数级、类级、代码组织、条件逻辑、数据结构）
2. ✅ 代码异味识别指南
3. ✅ 标准重构工作流程
4. ✅ 多语言重构示例（Go、TypeScript、Python）
5. ✅ 重构检查清单
6. ✅ 输出格式规范

请手动创建 `.trae/skills/refactor-code/` 目录并将上述内容保存为 `SKILL.md` 文件。


如需创建具体的 Skill，请告诉我您的需求，我可以帮您生成完整的 SKILL.md 文件。
# 重构skill
```
# 代码重构技能

## 1. 技能概述

**技能名称**: refactor-code
**技能版本**: 1.0.0
**技能描述**: 提供系统化的代码重构方法和最佳实践，帮助改善代码质量、可读性和可维护性

**触发条件**:
- 当用户请求重构代码时
- 当用户询问如何改进代码结构时
- 当用户提到"重构"、"优化代码"、"改善代码"等关键词时

**触发关键词**:
- 重构
- refactor
- 优化代码
- 改善代码
- 代码质量
- 技术债务
- 代码异味

## 2. 重构原则

### 核心原则

1. **小步重构**: 每次只做一个小改动，确保每步都可测试
2. **保持行为不变**: 重构不应改变代码的外部行为
3. **频繁测试**: 每次重构后运行测试确保功能正常
4. **版本控制**: 每个重构步骤独立提交，便于回滚

### 重构时机

- 代码重复（DRY 原则违反）
- 函数过长（超过 50 行）
- 类过大（职责过多）
- 参数列表过长（超过 3-4 个参数）
- 嵌套过深（超过 3 层）
- 命名不清晰

## 3. 重构方法清单

### 函数级别重构

| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 提取函数 | 将代码片段提取为独立函数 | 代码重复、函数过长 |
| 内联函数 | 将函数体直接嵌入调用处 | 函数体很简单 |
| 提取变量 | 将表达式提取为变量 | 复杂表达式 |
| 重命名函数/变量 | 使用更有意义的名称 | 命名不清晰 |
| 分解条件表达式 | 将复杂的条件逻辑提取为函数 | 条件表达式复杂 |

### 类级别重构

| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 提取类 | 将部分功能提取为新类 | 类职责过多 |
| 内联类 | 将类的功能合并到另一个类 | 类功能很少 |
| 提取接口 | 提取公共接口 | 多个类有相似行为 |
| 以工厂函数取代构造函数 | 使用工厂方法创建对象 | 创建逻辑复杂 |

### 条件逻辑重构

| 重构方法 | 说明 | 适用场景 |
|---------|------|---------|
| 分解条件表达式 | 提取条件判断为函数 | 条件表达式复杂 |
| 合并条件表达式 | 合并多个条件判断 | 多个条件有相同结果 |
| 以多态取代条件表达式 | 使用多态替代条件判断 | 有多个条件分支基于类型 |
| 引入空对象 | 使用空对象替代 null 检查 | 多处需要 null 检查 |

## 4. 代码异味识别

| 异味类型 | 症状 | 建议重构 |
|---------|------|---------|
| 重复代码 | 相同/相似代码出现在多处 | 提取函数 |
| 过长函数 | 函数超过 50 行 | 提取函数 |
| 过大类 | 类超过 500 行或职责过多 | 提取类 |
| 过长参数列表 | 参数超过 3-4 个 | 引入参数对象 |
| 发散式变化 | 一个类因多种原因变化 | 提取类 |
| 依恋情结 | 过多使用其他类的数据 | 移动函数 |
| 基本类型偏执 | 过多使用基本类型 | 引入值对象 |
| switch 语句 | 大量 switch/case | 以多态取代 |

## 5. 重构工作流程

```
1. 识别问题 → 分析代码异味，确定重构目标
2. 准备重构 → 确保测试覆盖，创建备份
3. 执行重构 → 小步修改，每步后运行测试
4. 验证结果 → 运行完整测试，代码审查
```

## 6. 重构示例

### 提取函数

**重构前**:
```go
func printOwing(invoice Invoice) {
    outstanding := 0.0
    fmt.Println("***********************")
    fmt.Println("*** Customer Owes ***")
    fmt.Println("***********************")
    for _, order := range invoice.Orders {
        outstanding += order.Amount
    }
    fmt.Printf("name: %s\n", invoice.Customer)
    fmt.Printf("amount: %.2f\n", outstanding)
}
```

**重构后**:
```go
func printOwing(invoice Invoice) {
    printBanner()
    outstanding := calculateOutstanding(invoice)
    printDetails(invoice.Customer, outstanding)
}

func printBanner() {
    fmt.Println("***********************")
    fmt.Println("*** Customer Owes ***")
    fmt.Println("***********************")
}

func calculateOutstanding(invoice Invoice) float64 {
    outstanding := 0.0
    for _, order := range invoice.Orders {
        outstanding += order.Amount
    }
    return outstanding
}

func printDetails(customer string, outstanding float64) {
    fmt.Printf("name: %s\n", customer)
    fmt.Printf("amount: %.2f\n", outstanding)
}
```

### 引入参数对象

**重构前**:
```go
func amountInvoiced(start time.Time, end time.Time, customerID string, productID string) float64
```

**重构后**:
```go
type InvoiceQuery struct {
    Start      time.Time
    End        time.Time
    CustomerID string
    ProductID  string
}

func amountInvoiced(query InvoiceQuery) float64
```

## 7. 输出格式

执行重构时按以下格式输出：

```markdown
## 重构分析

**检测到的问题**:
- [问题描述]

**建议的重构方法**:
- [重构方法]: [原因]

## 重构方案

### 重构前
[原始代码]

### 重构后
[重构后代码]

### 改进说明
- [改进点]
```

## 8. 注意事项

1. 不要在重构时添加新功能
2. 每次小改动后确保测试通过
3. 关注性能影响
4. 重构后更新相关文档
5. 每个重构步骤独立提交
```
```
