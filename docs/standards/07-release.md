# 发版规范（通用，语言无关）

适用范围：版本号、Tag、Changelog、发布流程、回滚。依据 Semantic Versioning 2.0 + Keep a Changelog。

## 1. 版本号（MUST）

遵循 SemVer `MAJOR.MINOR.PATCH`：
- **MAJOR**：不兼容的 API 变更（破坏性）。
- **MINOR**：向后兼容的功能新增。
- **PATCH**：向后兼容的缺陷修复。
- 预发布：`1.0.0-rc.1`、`1.0.0-beta.2`；构建元数据 `+20260721`。
- 版本号 **MUST** 单一来源（代码内常量或 `version` 文件），禁止多处硬编码不一致。

## 2. Tag 与分支（MUST）

- 每次发版 **MUST** 打注解 Tag：`v1.2.3`，消息含版本 + 摘要。
- 发布来源 **MUST** 是 `main`/`master` 上的特定 commit，禁止从未评审分支直接发版。
- 维护旧版本用 `release/vX.Y` 分支；hotfix 从该分支切出并合回。
- Tag **MUST** 不可变；发错用新 Patch 号重发，禁止 `git tag -f` 覆盖已公开 Tag。

## 3. 发版闸门（MUST）

合并 / 发版前 CI **MUST** 全绿：
- format + lint + 单测 + 集成测试 + 构建 + 安全扫描（依赖 / 密钥）。
- 发版类变更 **MUST** 通过：契约测试、烟雾测试、容量 / 性能基线（如有）。

## 4. Changelog 与发布说明（MUST）

- 发版前 **MUST** 更新 CHANGELOG（见文档规范），并生成 Release Notes（用户视角，非 git log 堆砌）。
- 破坏性变更 **MUST** 在 Release Notes 醒目提示迁移步骤。

## 5. 渐进式发布（SHOULD）

- 生产发布 **SHOULD** 用灰度 / 金丝雀 / 特性开关，避免全量一次性切换。
- 关键服务 **SHOULD** 支持快速回滚（≤ 分钟级）：保留上一版本镜像 / 可回退数据库迁移。

## 6. 数据库与配置变更（MUST）

- 数据库迁移 **MUST** 向后兼容（先扩后缩），禁止在发版中做不可逆的破坏性 DDL 而不预案。
- 配置 **MUST** 外部化（12-Factor）；发版不得依赖手动改服务器文件。

## 7. 回滚预案（MUST）

- 每次发版 **MUST** 在 PR / Release 中写明回滚步骤与负责人。
- 失败判定标准（健康度 / 错误率阈值）**MUST** 预先定义，触发即自动或人工回滚。

## 8. 禁止事项（红线）

- ❌ 跳过高危变更的评审与灰度直接全量。
- ❌ 覆盖已公开 Tag。
- ❌ 发版后无 Changelog / Release Notes。
- ❌ 不可逆破坏性变更无回滚预案。

## 检查清单

- [ ] 版本号符合 SemVer 且来源单一
- [ ] 已打 `vX.Y.Z` 注解 Tag
- [ ] CI 全绿（含安全扫描）
- [ ] Changelog + Release Notes 更新，破坏性变更有迁移说明
- [ ] 灰度 / 回滚预案就绪
- [ ] 数据库迁移向后兼容
