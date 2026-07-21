## ADDED Requirements

### Requirement: 统一 REST 契约（OpenAPI 3.0）
平台所有对外 REST 接口 MUST 以 OpenAPI 3.0 定义字段级 request/response，基址 `/api/v1`，统一响应结构 `{code,message,data,traceId}`，覆盖 DD-01 §3 全部模块接口（管理平台/采集/监控/全链路分析/APM/基建/日志/告警）。

#### Scenario: 前后端并行开发依赖契约
- **WHEN** 前端开发某页面而对应后端服务未实现
- **THEN** 前端按 OpenAPI 生成的 TS 类型与 Mock（MSW/本地）开发，待后端就绪后切真实接口，契约字段无须改动

#### Scenario: 契约破坏性变更被门禁拦截
- **WHEN** PR 修改了已发布接口的字段（删除/改类型/改必填）
- **THEN** CI 的 `openapi diff` 判定破坏性变更并阻断合入，要求升 `/v2` 或走 Deprecation 周期

### Requirement: 统一 gRPC 契约（.proto）
14 个微服务间内部调用 MUST 以 `.proto` 定义（package `tracing.v1`），含 DD-01 §5 全部 RPC（PushSpan/PushMetric/GetAgentConfig/SaveTrace/QueryTrace/CalcTopology/Evaluate/Watch/Publish/VerifyToken/CheckPerm 等），Java/Go 各自生成 stub。

#### Scenario: 服务间调用基于生成 stub
- **WHEN** ingestion-service 调用 trace-store 的 SaveTrace RPC
- **THEN** 双方使用由 .proto 生成的 stub，超时 1000ms，仅幂等 RPC 重试 2 次指数退避，非幂等写不重试

### Requirement: 统一错误码与响应结构
所有 REST 响应 MUST 使用统一结构 `{code,message,data,traceId}`，错误码遵循 DD-01 §7（0/40001/40101/40301/40901/42901/50001），500 错误 MUST 带 `traceId` 供定位。

#### Scenario: 内部错误带 traceId
- **WHEN** 服务发生未预期异常返回 50001
- **THEN** 响应体含 `traceId` 字段，前端据提示"错误编号 ERR-xxx"并可用于后端链路定位
