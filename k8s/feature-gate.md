# FeatureGate 快速掌握
**FeatureGate** 是一种在 Kubernetes 及其扩展生态中常见的功能开关机制，用来控制某个功能是否启用。它的设计目标是：  
- **安全渐进**：在 Alpha/Beta 阶段通过开关控制功能是否暴露。  
- **灵活覆盖**：支持全局配置，也可通过对象 Annotation 或命令行参数覆盖。  
- **可回滚**：功能出现问题时，可以快速关闭开关，避免影响生产。
## 🔑 核心用法
- **定义 FeatureGate**  
  ```go
  featureGate.Add(map[featuregate.Feature]featuregate.FeatureSpec{
      "DeclarativeUpgrade": {Default: false, PreRelease: featuregate.Alpha},
  })
  ```
  - 注册一个名为 `"DeclarativeUpgrade"` 的功能开关，默认关闭，处于 Alpha 阶段。

- **检查是否启用**  
  ```go
  if featureGate.Enabled("DeclarativeUpgrade") {
      // 执行新功能逻辑
  }
  ```
  - 返回布尔值，决定是否执行新功能。

- **支持覆盖**  
  - 可以通过对象 Annotation 或配置文件覆盖全局开关，例如：
    ```yaml
    metadata:
      annotations:
        cvo.openfuyao.cn/declarative-upgrade: "true"
    ```
## 📋 使用规范清单
| 规范项 | 要求 | 示例 |
|--------|------|------|
| **命名规范** | 使用清晰的功能名，避免缩写或模糊词 | `"DeclarativeUpgrade"` |
| **默认值** | 明确指定 `Default: true/false`，避免隐式行为 | `Default: false` |
| **阶段标识** | 必须标注 `PreRelease: Alpha/Beta/GA` | `PreRelease: featuregate.Alpha` |
| **文档说明** | 每个 FeatureGate 都要在开发文档中说明用途、默认值、启用方式 | Release Notes 中记录 |
| **覆盖策略** | 支持通过 Annotation 或配置覆盖，保证灵活性 | `obj.GetAnnotations()["..."]` |
| **渐进升级** | 随功能成熟度逐步提升阶段，最终移除 FeatureGate | Alpha → Beta → GA → 移除 |
| **回滚能力** | 出现问题时能快速关闭开关，避免影响生产 | `featureGate.Enabled(...)` 返回 false |
## 🌟 最佳实践
- **集中管理**：统一在一个文件中定义所有 FeatureGate，避免分散。  
- **测试覆盖**：在单元测试和集成测试中分别验证开关开启/关闭的行为。  
- **灰度发布**：先在测试环境开启，再逐步推广到生产。  
- **自动化校验**：CI/CD 流水线中检查 FeatureGate 配置是否符合规范。  
- **文档透明**：在 Release Notes 中明确说明 FeatureGate 的状态和启用方式。  

总结：**FeatureGate 是功能渐进发布的安全阀门**，快速掌握的关键在于：**定义清晰、检查统一、覆盖灵活、文档透明、可回滚**。  
