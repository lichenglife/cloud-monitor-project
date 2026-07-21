## ADDED Requirements

### Requirement: 日志 SDK 复用 Agent TraceID（决议 C）
日志平台 SDK MUST 改造为复用 Agent 注入的 MDC TraceID（检测 SkyWalking ContextManager，存在则取，不存在降级自生成兼容无 Agent 场景），日志平台侧 MUST 建立 trace_id 倒排索引，保证同一请求日志 traceId = 链路 traceId。

#### Scenario: 日志复用 Agent TraceID
- **WHEN** 应用接入 SkyWalking Agent 并打印日志
- **THEN** 日志 SDK 从 ContextManager 取 Agent traceId，日志 traceId = 链路 traceId，不再自生成

#### Scenario: 无 Agent 降级兼容
- **WHEN** 应用未接入 Agent
- **THEN** 日志 SDK 降级自生成 traceId，不报错，兼容无 Agent 场景

### Requirement: 按 TraceID 关联日志查询与反向定位
MUST 支持 GET /logs?traceId 按 traceId 从日志平台拉取关联日志条目（侧栏展示，支持级别筛选），MUST 支持 GET /logs/{logId}/trace 日志->Trace 反向定位至 trace_npm_jaeger，MUST 查询限频（同 TraceID 每分钟 ≤20 次）+ 结果缓存（Redis trace:logs:cache 120s TTL）。

#### Scenario: 按 TraceID 查关联日志
- **WHEN** 用户在链路详情点击"关联日志"
- **THEN** 按 traceId 从日志平台拉取日志条目，侧栏展示，可按级别筛选

#### Scenario: 日志->Trace 反向定位
- **WHEN** 用户在日志详情点击"定位链路"
- **THEN** 跳转至该日志 traceId 对应的链路详情（trace_npm_jaeger）
