# 项目工程纪律（AI 编码强制规则）

## 流程
- 任何需求变更必须先走 OpenSpec：/opsx:propose → 人工审核 → /opsx:apply → /opsx:archive。
- 禁止在 main 分支直接实现；非平凡任务一律在 git worktree 中完成。

## 测试
- 强制 TDD：先写失败测试（RED），再写最小实现（GREEN），最后重构。
- 不允许"生成即交付"；没有测试覆盖的代码视为未完成。
- 目标覆盖率 ≥ 80%。

## 自主执行（RalphLoop）
- 过夜自修只允许在"已 apply 的变更 worktree"上运行。
- 必须设置 --max-iterations 与 --completion-promise "DONE"。
- RalphLoop 只动实现，不修改 openspec/specs/ 与 proposal.md。

## 收尾
- 每天早晨人工验收：git log、跑测试、手测关键路径、检查密钥与性能。
- 验收通过后才 /opsx:archive 并 git commit。

## 项目开发强约束规范（Claude Code 自动加载）

> 本文件是项目对 **Claude Code 及所有 AI 编码助手** 的**强制约束入口**。
> 违反以下任何一条，即视为不合格交付，必须在提交前自行修正。

### 0. 总原则

1. **规范优先于习惯**：动笔写任何代码 / API / 文档 / 提交信息之前，必须先读取 `docs/standards/` 下对应规范文件。
2. **规范冲突时的优先级**（高 → 低）：
   安全 > 发版 > 错误处理 > 日志 > 代码规范 > API 规范 > 目录规范 > 文档规范 > 提交规范。
3. **可验证优先**：能用 lint / format / 单测 / 构建拦截的规则，必须在 CI 中落地为门禁，不能只靠人工评审。
4. **小步提交、显式意图**：不产出"能跑就行"的黑盒代码；每个变更都可追溯、可回滚。

### 1. 规范索引（强制阅读）

| 类别 | 通用（语言无关） | Java | Go |
|---|---|---|---|
| 目录规范 | — | `docs/standards/java/01-directory.md` | `docs/standards/go/01-directory.md` |
| API 规范 | — | `docs/standards/java/02-api.md` | `docs/standards/go/02-api.md` |
| 代码规范 | — | `docs/standards/java/03-code.md` | `docs/standards/go/03-code.md` |
| 提交规范 | `docs/standards/03-commit.md` | 同左 | 同左 |
| 文档规范 | `docs/standards/04-documentation.md` | 同左 | 同左 |
| 错误处理 | `docs/standards/java/05-error-handling.md` | 见 Java | 见 Go |
| 日志规范 | `docs/standards/java/06-logging.md` | 见 Java | 见 Go |
| 发版规范 | `docs/standards/07-release.md` | 同左 | 同左 |
| 补充规范（安全/测试/配置/CI/性能/依赖） | `docs/standards/08-others.md` | 同左 | 同左 |

完整索引见 `docs/standards/README.md`。

### 2. 语言判定

- **仅 Java 项目** → 遵守 `docs/standards/java/*`。
- **仅 Go 项目** → 遵守 `docs/standards/go/*`。
- **多语言 monorepo** → 每个模块遵守各自语言规范；跨语言约定以通用规范为准。

### 3. 与 OpenSpec / Superpowers / RalphLoop 协同（推荐工作流）

- 任何需求变更**先走 OpenSpec**：`/opsx:propose` → **人工审核 proposal/design** → `/opsx:apply`（Superpowers 接管，强制 TDD + worktree）→ `/opsx:archive`。
- 编码前**先写失败测试**（RED → GREEN → REFACTOR）。
- 非平凡任务**必须在 git worktree 中实现**，禁止直接改 `main` / `master`。
- 过夜自修用 **RalphLoop**：仅动实现，不修改 `openspec/specs/` 与 `proposal.md`。
- 规范本身即 OpenSpec 的"完成信号"来源：验收标准必须可量化（测试 / 覆盖率 / lint 全绿）。

### 4. 红线（禁止事项）

- ❌ 提交未通过 format + lint + 单测 + 构建 的代码。
- ❌ 在代码、配置、日志、提交信息中硬编码密钥 / Token / PII。
- ❌ 用 `System.out.println` / `fmt.Println` 充当生产日志。
- ❌ 吞掉异常 / `error`（必须处理，或显式忽略并注释原因）。
- ❌ 超大函数 / 超大文件（行数上限见各语言代码规范）。
- ❌ 绕过规范直接生成"能跑就行"的代码。
- ❌ 对外暴露内部错误堆栈 / 数据库结构 / 基础设施细节。

### 5. 验收闸门（每次产出后自检）

- [ ] 已读取对应语言 / 类别的规范文件。
- [ ] 通过了 `gofmt`/`goimports` 或 `spotless`/`google-java-format` + `golangci-lint` 或 `checkstyle`/`pmd`。
- [ ] 新增逻辑有对应单测，覆盖率达标（通用门槛 ≥ 80%）。
- [ ] 错误处理、日志、API 响应符合规范。
- [ ] 提交信息符合 Conventional Commits。
- [ ] 无密钥 / 敏感信息泄露。
