# Java 规范索引

本目录存放 **Java 项目** 的专项开发规范（强约束），配合根 `CLAUDE.md` 与 `docs/standards/README.md` 使用。

## 文件清单

| 文件 | 内容 | 配套通用规范 |
|---|---|---|
| `01-directory.md` | Maven/Gradle 目录布局、包命名、分层依赖 | — |
| `02-api.md` | REST、统一响应 `Result`、统一异常、校验、版本化 | 文档规范（OpenAPI） |
| `03-code.md` | 命名、类设计、并发、异常、单测 | 补充规范（测试/安全） |
| `05-error-handling.md` | 异常体系、错误码、事务回滚、对外暴露 | API 规范、日志规范 |
| `06-logging.md` | SLF4J、占位符、级别、MDC、脱敏、结构化 | 补充规范（可观测性） |

## 强制工具链（建议落地为 CI 门禁）

- 格式化 / 静态检查：`google-java-format` 或 `spotless` + `checkstyle` + `pmd` + `errorprone`。
- 测试：`JUnit 5` + `Mockito` + `testcontainers`；覆盖率门槛 ≥ 80%。
- 安全：`dependency-check` / `snyk` 依赖扫描；密钥扫描。
- 构建：`Maven` / `Gradle`，lockfile 锁定版本。

## 优先级

安全 > 发版 > 错误处理 > 日志 > 代码 > API > 目录。规范冲突时以此为准，并在 PR 中说明。

详见根 `CLAUDE.md`「验收闸门」。
