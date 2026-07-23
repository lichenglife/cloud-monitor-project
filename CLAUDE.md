# 项目开发强约束规范（Claude Code 自动加载）

> 本文件是项目对 **Claude Code 及所有 AI 编码助手** 的**强制约束入口**。
> 违反以下任何一条，即视为不合格交付，必须在提交前自行修正。

## 0. 总原则

1. **规范优先于习惯**：动笔写任何代码 / API / 文档 / 提交信息之前，必须先读取 `docs/standards/` 下对应规范文件。
2. **规范冲突时的优先级**（高 → 低）：
   安全 > 发版 > 错误处理 > 日志 > 代码规范 > API 规范 > 目录规范 > 文档规范 > 提交规范。
3. **可验证优先**：能用 lint / format / 单测 / 构建拦截的规则，必须在 CI 中落地为门禁，不能只靠人工评审。
4. **小步提交、显式意图**：不产出"能跑就行"的黑盒代码；每个变更都可追溯、可回滚。
5. **工件即契约**：OpenSpec 的 `spec` Scenario 是验收的唯一事实源；任何"完成"都必须能对照 Scenario 验证。

## 1. 规范索引（强制阅读）

| 类别 | 通用（语言无关） | Java | Go |
|---|---|---|---|
| 目录规范 | — | `docs/standards/java/01-directory.md` | `docs/standards/go/01-directory.md` |
| API 规范 | — | `docs/standards/java/02-api.md` | `docs/standards/go/02-api.md` |
| 代码规范 | — | `docs/standards/java/03-code.md` | `docs/standards/go/03-code.md` |
| 提交规范 | `docs/standards/03-commit.md` | 同左 | 同左 |
| 文档规范 | `docs/standards/04-documentation.md` | 同左 | 同左 |
| 错误处理 | — | `docs/standards/java/05-error-handling.md` | `docs/standards/go/05-error-handling.md` |
| 日志规范 | — | `docs/standards/java/06-logging.md` | `docs/standards/go/06-logging.md` |
| 发版规范 | `docs/standards/07-release.md` | 同左 | 同左 |
| 补充规范（安全/测试/配置/CI/性能/依赖） | `docs/standards/08-others.md` | 同左 | 同左 |

完整索引见 `docs/standards/README.md`。

## 2. 语言判定

- **仅 Java 项目** → 遵守 `docs/standards/java/*`。
- **仅 Go 项目** → 遵守 `docs/standards/go/*`。
- **多语言 monorepo** → 每个模块遵守各自语言规范；跨语言约定以通用规范为准。

## 3. 三套工具的协作模型（精确版）

> **前置**：Superpowers 已通过 `/plugin install` 安装且其 `SessionStart Hook` 已生效（这是"自动纪律"的根）；
> OpenSpec 已在仓库内 `openspec init`；RalphLoop 插件可用。三者缺一，全流程自动化降级为人工逐步执行。

### 3.1 职责边界（互斥、不重叠）

> **铁律：OpenSpec 是"规划的唯一事实源"。Superpowers 在本项目只提供工程纪律，绝不重复规划。**

| 工具 | 角色 | 负责 | 明确不做 |
|---|---|---|---|
| **OpenSpec** | 事实源 + 生命周期 | 需求/设计/验收工件（`proposal` / `design` / `tasks` / `spec`）+ `propose → apply → archive`；`archive` 把 delta 合并回 `openspec/specs/`。 | 不写实现代码；不跑 TDD。 |
| **Superpowers** | 工程纪律（仅） | 执行实现时提供 **TDD（RED→GREEN→REFACTOR）** + **git worktree 隔离**。 | ❌ **禁用** `brainstorming` / `writing-plans` / `subagent-driven-development`——这些会与 OpenSpec 的 `tasks.md` 重复规划、内容对不上、反复确认、拖长流程。 |
| **RalphLoop** | 自主执行 | `apply` 之后**按 `tasks.md` 逐项**过夜自修（补测/提覆盖/重构/修边界），以 §3.4 验收门禁为 `--completion-promise`。 | 不重新规划；不改 `spec` / `design`。 |

> **你遇到的"内容不匹配 + 效率低"根因**：Superpowers 的 `brainstorming` / `writing-plans` 会生成自己的规划工件，与 OpenSpec 的 `proposal` / `design` / `tasks` 形成"双规划系统"。两者对"做什么"的描述必然漂移，且规划阶段交互确认多、耗时久。既然规划权已交给 OpenSpec，Superpowers 必须**只保留其最高价值的纪律（TDD + worktree）**，其余规划类技能一律禁用。

> **模型一致性（GLM-5.2 前提，直接影响执行速度）**：本项目主模型是 **GLM-5.2**，而 Superpowers 的 `subagent-driven-development` / `requesting-code-review` 默认会把子任务路由到 **Haiku / Sonnet / Opus** 等不同模型。这是执行慢的**额外主因**：每个子 agent 都是一次独立的远程模型往返（含上下文传输、冷启动），跨模型叠加后延迟显著；且与非 Anthropic 主模型存在行为不匹配。对策：(1) 已禁用 `subagent-driven-development`；(2) **禁止把任何子 agent / 审查任务路由到其他模型**——统一用主模型；(3) 在 Claude Code `settings.json` 中将子 agent 默认模型设为"继承主模型"，不要显式指定 Haiku/Sonnet/Opus。本项目 `.claude/agents/*.md` 均不指定 `model`，自动继承主模型（正确）。

### 3.2 联动机制（明确、无歧义）

- ❌ **误解**：`/opsx:apply` 会自动调用 Superpowers 的 `brainstorming` / `writing-plans`。
- ✅ **真相**：
  1. OpenSpec 的 `apply` 是**自包含** skill，只读 `tasks.md` 逐项实现，不知道 Superpowers 存在。
  2. Superpowers 仍经 `SessionStart Hook` 全局武装模型——但本文件**显式禁用其规划类技能**，所以模型在 `apply` 时只应用 TDD + worktree，不会重新 brainstorm / 写 plan。
  3. 硬门禁（`openspec/config.yaml`）校验工件合规；`/opsx:verify` 校验验收门禁。
- **一句话**：OpenSpec 给"计划"，Superpowers 给"纪律"，RalphLoop 给"持续执行"。三者靠"文件 + 约束"对接，不是代码调用。

### 3.3 全流程工作流（精简、非交互优先）

```
[需求]
  │
  ├─① /opsx:propose   ← OpenSpec 独立完成规划，产出 proposal/design/tasks/spec
  │                     （本文件禁止触发 Superpowers 规划技能；模型不得在此 brainstorm）
  │
  ├─② 人工闸门：审阅 design.md + spec Scenario   ← 不可跳过，成本低收益高
  │
  ├─③ /opsx:apply（直接实现，不再规划）
  │     └─ 对 tasks.md 逐项：
  │          - 在 git worktree 中开隔离分支
  │          - RED：先写失败测试（对照对应 Scenario）
  │          - GREEN：最小实现让测试通过
  │          - REFACTOR：不破坏测试下重构
  │          - 每项完成打 [x]，跑该模块测试
  │     ❌ 禁止调用 brainstorming / writing-plans / subagent-driven-development
  │
  ├─④ 自动验收闸门（§3.4）：spec Scenario 全满足 + CI 门禁全绿
  │
  ├─⑤ 过夜自修：/ralph-loop "<change-id> 按 tasks.md 补测/提覆盖/重构/修边界" \
  │        --completion-promise "ALL SCENARIOS PASS AND COVERAGE≥80% AND LINT CLEAN"
  │
  ├─⑥ 早晨人工抽检：git log + 全量测试 + 手测关键路径
  │
  └─⑦ /opsx:archive   ← delta 合并回 openspec/specs/，commit
```

> **阶段 ③ 为何快且非交互**：规划已在 ① 完成，且 Superpowers 规划技能被禁用，模型无需反复确认"做什么"，只需按 `tasks.md` 执行"怎么做"（TDD）。这对 GLM 等较弱模型尤其关键——避免它把多个 skill 串起来过度规划、自我拉扯。

### 3.4 验收自动化（完成信号 = 规范 + spec）

- 每条 spec `#### Scenario:（Given / When / Then）` 都是**可机器校验的验收标准**；实现后必须能演示 / 测试该场景通过。
- **验收门禁**（`apply` 后、`/opsx:archive` 前必须全绿）：
  - [ ] `gofmt`/`goimports` 或 `spotless`/`google-java-format` 无差异；
  - [ ] `golangci-lint` 或 `checkstyle`+`pmd` 零违规；
  - [ ] 单测全绿，变更相关代码覆盖率 ≥ 80%；
  - [ ] 所有 spec Scenario 对应的测试 / 演示通过；
  - [ ] 提交信息符合 Conventional Commits；无密钥 / PII 泄露。
- RalphLoop 的 `--completion-promise` 必须引用上述门禁，否则它只是空转自改。

## 4. 红线（禁止事项）

- ❌ 提交未通过 format + lint + 单测 + 构建 的代码。
- ❌ 在代码、配置、日志、提交信息中硬编码密钥 / Token / PII。
- ❌ 用 `System.out.println` / `fmt.Println` 充当生产日志。
- ❌ 吞掉异常 / `error`（必须处理，或显式忽略并注释原因）。
- ❌ 超大函数 / 超大文件（行数上限见各语言代码规范）。
- ❌ 绕过规范直接生成"能跑就行"的代码。
- ❌ 对外暴露内部错误堆栈 / 数据库结构 / 基础设施细节。
- ❌ 在 OpenSpec 驱动的变更中启用 Superpowers 的 `brainstorming` / `writing-plans` / `subagent-driven-development`（规划权唯一属于 OpenSpec，重复规划是效率低/内容不匹配的根因）。
- ❌ 把子 agent / 代码审查 / 任何派发任务路由到 Haiku / Sonnet / Opus 等非主模型（本项目统一 GLM-5.2；跨模型往返是执行慢的主因之一）。
- ❌ 在 `tasks.md` 不写"单元测试 + 量化验收标准"就进入 `apply`（否则 RalphLoop 无收敛信号）。

## 5. 验收闸门（每次产出后自检）

- [ ] 已读取对应语言 / 类别的规范文件。
- [ ] 通过了 `gofmt`/`goimports` 或 `spotless`/`google-java-format` + `golangci-lint` 或 `checkstyle`/`pmd`。
- [ ] 新增逻辑有对应单测，覆盖率达标（通用门槛 ≥ 80%）。
- [ ] 错误处理、日志、API 响应符合规范。
- [ ] 提交信息符合 Conventional Commits。
- [ ] 无密钥 / 敏感信息泄露。
- [ ] 实现对照 `spec` 的每条 Scenario 均可验证通过。
