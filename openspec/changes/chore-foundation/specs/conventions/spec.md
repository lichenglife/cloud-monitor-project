## ADDED Requirements

### Requirement: 统一日志规范
所有组件 MUST 采用结构化日志（Java Logback + logstash-logback-encoder JSON；Go uber/zap JSON），每条日志 MUST 含 `timestamp/level/service/trace_id/span_id` 字段，日志级别统一（DEBUG/INFO/WARN/ERROR/FATAL），MUST 禁止明文敏感信息（密码/令牌/PII）入日志，trace_id MUST 经 MDC/context 贯穿单次请求全链路。

#### Scenario: 请求链路日志可串联
- **WHEN** 一个请求经 gateway 路由到 trace-query-service 再到 ingestion-service
- **THEN** 三组件该请求的日志均含同一 trace_id，可按 trace_id 聚合还原调用链

#### Scenario: 敏感信息不入日志
- **WHEN** 日志内容含手机号或密码字段
- **THEN** 日志输出前经脱敏（手机号 `138****1234`、密码 `***`），原始值不落日志

### Requirement: 统一错误处理规范
所有组件 MUST 经统一异常处理（Java `@ControllerAdvice` + 全局 `ErrorHandler`；Go grpc-middleware `recovery` + 统一 `apperror` 包）捕获异常并映射 DD-01 §7 错误码（0/40001/40101/40301/40901/42901/50001），500 错误 MUST 返回 traceId，MUST 禁止向客户端泄漏内部堆栈，错误 MUST 记录审计/告警（P0 级）。

#### Scenario: 未预期异常返回 traceId 不泄漏堆栈
- **WHEN** 服务发生未捕获异常
- **THEN** 响应 50001 + traceId，客户端只见"错误编号 ERR-xxx"，完整堆栈仅落服务端日志，P0 级触发告警

#### Scenario: 参数校验失败返回字段级错误
- **WHEN** 请求参数缺失或格式错
- **THEN** 返回 40001 + 字段级错误信息（如"参数 phone 格式不正确"），不进入业务逻辑

### Requirement: 统一接口规范
所有 REST 接口 MUST 遵循基址 `/api/v1`、统一响应 `{code,message,data,traceId}`、HTTP 方法语义（GET 查询/POST 创建/PUT 更新/DELETE 删除）、分页约定（`page/pageSize` 入参，`total/items` 出参）、时间格式 ISO 8601 UTC；接口 MUST 经 API 网关鉴权/限流/审计。

#### Scenario: 分页查询统一约定
- **WHEN** 前端请求 GET /api/v1/traces?page=1&pageSize=20
- **THEN** 响应 data 含 `{total, items:[...]}`，分页字段统一，前端组件可复用

### Requirement: 统一代码规范
Java 代码 MUST 遵循 Google Java Style + Spring 官方约定（包 `com.lichenglife.npm.trace.<service>`，分层 controller/service/repository/dto），Go 代码 MUST 遵循 go-clean 分层（cmd/internal/domain/usecase/repository，module `github.com/lichenglife/cloud-monitor/<service>`），MUST 经格式化（Java spotless/Google Java Format，Go gofmt+golangci-lint）与静态扫描（无高危漏洞），MUST 遵循 SRP/OCP/LSP/ISP/DIP（设计原则基线）。

#### Scenario: 代码分层禁止跨层
- **WHEN** 开发者在 controller 直接访问 ES
- **THEN** lint 规则/评审拒绝，须经 service -> repository 抽象层

### Requirement: 统一文档规范
每个组件 MUST 含 README（启动/构建/依赖/配置/健康检查）、API 文档由 OpenAPI/proto 自动生成、变更 MUST 更新对应 DD 与 CHANGELOG，架构决策 MUST 记录于 design.md，文档与代码同版本管理。

#### Scenario: 组件可独立上手
- **WHEN** 新成员接手某组件
- **THEN** 按 README 可独立构建启动并理解职责，无需口头交接

### Requirement: 统一代码提交规范
提交信息 MUST 格式 `[任务号] <type>: <简述>`（type ∈ feat/fix/refactor/docs/test/chore/perf），MUST 关联 TAPD 任务 ID，PR MUST ≥1 人 Review + CI 全绿方可入 develop，main 仅经 release 合并，提交 MUST 不含明文密钥（pre-commit 扫描拦截）。

#### Scenario: 提交未关联任务号被拒
- **WHEN** 提交信息未含任务号或 type
- **THEN** commitlint 钩子拒绝提交，提示规范格式

### Requirement: 组件访问协议（gRPC + HTTP）
服务间内部调用 MUST 用 gRPC（mTLS，package `tracing.v1`，超时 1000ms，幂等 RPC 重试 2 次指数退避 200ms->400ms，非幂等不重试），对外/前端 MUST 用 RESTful JSON（经 gateway），Agent->Collector MUST 用 OTLP（gRPC:4317/HTTP:4318，TLS 1.3 + 注册令牌），所有跨进程通信 MUST 鉴权 + 加密传输。

#### Scenario: 服务间 gRPC 超时与重试
- **WHEN** ingestion-service 调 trace-store SaveTrace 超时
- **THEN** 因 SaveTrace 为幂等 RPC，按 200ms->400ms 指数退避重试 2 次，仍失败则降级本地缓冲不阻塞

#### Scenario: 非幂等写不重试
- **WHEN** 调用非幂等写 RPC（如创建告警规则）超时
- **THEN** 不重试，直接返回错误由调用方决策，避免重复创建

### Requirement: 组件可独立启动运行
每个组件脚手架 MUST 可独立启动（Java `mvn spring-boot:run` / Go `make run`），`/health`（Liveness/Readiness）返回 200，依赖中间件经 TestContainers（Java）/ testcontainers-go 或本地 docker-compose 起本地 MySQL/Redis/ES/Kafka，MUST 不依赖其他业务组件即可冒烟运行。

#### Scenario: 新组件独立冒烟
- **WHEN** 开发者从脚手架派生新组件并 `make run`
- **THEN** 组件启动，/health 返回 200，可接收 gRPC/HTTP 请求并返回统一响应，无需启动其他 12 个业务组件
