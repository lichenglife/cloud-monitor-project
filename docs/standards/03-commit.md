# 提交规范（通用，语言无关）

适用范围：所有提交信息、分支命名、Pull Request。依据 Conventional Commits 1.0 + Angular 规范。

## 1. 提交信息格式（MUST）

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **type（MUST）**：`feat` | `fix` | `docs` | `style` | `refactor` | `perf` | `test` | `build` | `ci` | `chore` | `revert`
- **scope（SHOULD）**：受影响的模块 / 包，如 `auth`、`order`、`api`。
- **subject（MUST）**：祈使句、小写开头、不加句号，**≤ 72 字符**，描述"做了什么"而非"为什么"。
- **body（SHOULD）**：解释动机与对比（为什么、与之前差异）。换行 ≤ 72 字符。
- **footer（SHOULD）**：`BREAKING CHANGE:` 或 `Closes #123` / `Refs #456`。

示例：
```
feat(order): support partial refund for mixed cart

Allow refunding a subset of items in a paid order. Refund amount
is validated against remaining payable before submission.

Closes #1024
```

## 2. 类型约束（MUST）

- `feat` / `fix` 会进入 CHANGELOG；`fix` 对应补丁版本，`feat` 对应次版本。
- 破坏性变更 **MUST** 在 footer 写 `BREAKING CHANGE:`，type 仍用 `feat`/`fix` 但触发主版本号提升。
- `style` 仅指格式化改动，不含逻辑变更；**禁止**把逻辑改动塞进 `style`/`chore` 规避评审。

## 3. 原子提交（MUST）

- 一个提交只解决**一件事**；不在同一提交混合"重构 + 功能 + 格式化"。
- 禁止 `WIP`、`temp`、`fix again` 等无意义信息。
- 提交前 **MUST** 自检：`format + lint + 单测` 全绿。

## 4. 分支命名（MUST）

- `feature/<issue>-<short>`、`fix/<issue>-<short>`、`refactor/<short>`、`chore/<short>`、`release/vX.Y.Z`。
- 长生命周期分支禁止直接推送 `main`/`master`；必须经 PR 评审 + CI 绿。

## 5. Pull Request（MUST）

- PR 标题沿用 Conventional Commits 格式。
- 描述 **MUST** 包含：变更目的、影响范围、测试方法、回滚方案（发版类）。
- 关联 issue；禁止无描述、无审查的"大杂烩"PR。
- 禁止 `force-push` 已评审分支（除非整体重写且通知评审人）。

## 6. 禁止事项（红线）

- ❌ 提交密钥、Token、`.env`、凭据文件。
- ❌ 提交大二进制 / 依赖（`node_modules`、`vendor` 规则见语言规范）。
- ❌ 提交未通过 CI 的代码（CI 为合并门禁）。

## 检查清单

- [ ] type / scope / subject 符合格式，subject ≤ 72 字符
- [ ] 破坏性变更标注 `BREAKING CHANGE:`
- [ ] 一个提交一件事，无混合意图
- [ ] 关联 issue，PR 描述完整
- [ ] 无密钥 / 大文件 / 未通过 CI 的内容
