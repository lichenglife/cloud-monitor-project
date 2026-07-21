# Go 规范索引

本目录存放 **Go 项目** 的专项开发规范（强约束），配合根 `CLAUDE.md` 与 `docs/standards/README.md` 使用。

## 文件清单

| 文件 | 内容 | 配套通用规范 |
|---|---|---|
| `01-directory.md` | standard layout、internal/pkg、包命名、分层 | — |
| `02-api.md` | HTTP/gRPC、统一响应、recover 中间件、版本化、超时 | 文档规范（OpenAPI/proto） |
| `03-code.md` | 命名、错误、接口、并发、格式化、单测 | 补充规范（测试/安全） |
| `05-error-handling.md` | error as value、%w 包装、sentinel、recover | API 规范、日志规范 |
| `06-logging.md` | slog/zap、结构化、context、脱敏 | 补充规范（可观测性） |

## 强制工具链（建议落地为 CI 门禁）

- 格式化 / lint：`gofmt` + `goimports` + `golangci-lint`（errcheck / govet / staticcheck / revive）。
- 测试：`go test -race -cover`，覆盖率门槛 ≥ 80%；集成用 `httptest` / testcontainers。
- 安全：`govulncheck` 漏洞扫描 + 密钥扫描。
- 构建：`go build` / `go mod verify`，`go.sum` 锁版本。

## 优先级

安全 > 发版 > 错误处理 > 日志 > 代码 > API > 目录。规范冲突时以此为准，并在 PR 中说明。

详见根 `CLAUDE.md`「验收闸门」。
