# 项目开发规范索引（docs/standards）

本目录存放**可直接被 Claude Code 作为强约束加载**的项目开发规范。所有规则默认是 **MUST（强制）**；建议项标注 **SHOULD**。

## 如何使用

1. **入口**：项目根 `CLAUDE.md` 已声明这些规范为强约束，Claude Code 启动即自动加载。
2. **编码前**：先读对应语言 + 类别的规范文件（见下表）。
3. **提交前**：用每个文件末尾的「检查清单」自检；CI 门禁会二次拦截。
4. **扩展**：新增规范请在对应目录新增 `NN-name.md` 并在本索引与 `CLAUDE.md` 登记。

## 文件清单

### 通用（语言无关）
| 文件 | 内容 |
|---|---|
| `03-commit.md` | 提交规范：Conventional Commits、分支、原子提交、PR |
| `04-documentation.md` | 文档规范：README、ADR、API 文档、CHANGELOG、注释 |
| `07-release.md` | 发版规范：SemVer、Tag、Changelog、回滚、门禁 |
| `08-others.md` | 补充规范：安全、测试、配置、CI/CD、性能、依赖、评审 |

### Java（`java/`）
| 文件 | 内容 |
|---|---|
| `java/01-directory.md` | 目录规范：Maven/Gradle 布局、分层、包命名 |
| `java/02-api.md` | API 规范：REST、统一响应、统一异常、校验、版本化 |
| `java/03-code.md` | 代码规范：命名、类设计、并发、异常、单测 |
| `java/05-error-handling.md` | 错误处理：异常体系、全局处理、错误码、事务回滚 |
| `java/06-logging.md` | 日志规范：SLF4J、级别、MDC、脱敏、结构化 |

### Go（`go/`）
| 文件 | 内容 |
|---|---|
| `go/01-directory.md` | 目录规范：standard layout、internal/pkg、包命名 |
| `go/02-api.md` | API 规范：HTTP/gRPC、统一响应、中间件、版本化 |
| `go/03-code.md` | 代码规范：命名、错误、接口、并发、格式化、单测 |
| `go/05-error-handling.md` | 错误处理：error as value、wrap、sentinel、recover |
| `go/06-logging.md` | 日志规范：slog/zap、结构化、context、脱敏 |

## 规范依据（业界通用 / 大厂实践）

- 提交：Conventional Commits 1.0、Angular Commit 规范
- 发版：Semantic Versioning 2.0、Keep a Changelog
- Java 代码：Google Java Style Guide、阿里巴巴《Java 开发手册》、Oracle Java Code Conventions
- Go 代码：Uber Go Style Guide、Go Code Review Comments、Effective Go
- API：REST API 设计成熟实践、OpenAPI 3、Google API Design Guide、gRPC
- 配置：12-Factor App
- 错误处理 / 日志：各语言官方最佳实践 + 可观测性（OpenTelemetry）惯例
