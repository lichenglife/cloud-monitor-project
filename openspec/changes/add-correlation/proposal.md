## Why

三域关联（Trace×Metric×Log）是"一次排障不必跨系统跳转"的核心（痛点 P2 数据孤岛）。M3 已能查 Trace，M2 已有指标，但日志仍独立--排障需跨平台跳转。M6 以 TraceID 为关联主键联动三方，并改造日志平台 SDK 复用 Agent TraceID（决议 C），使"一条链路看到全部上下文"（PRD US-09/REQ-M6-U1/E2）。

M1 已在 Agent 侧 MDC 注入 TraceID；本 change 做日志平台 SDK 改造（复用 TraceID 不再自生成）+ 三域关联查询 + 前端关联面板。对应研发任务 T-M6-1~T-M6-3。

## What Changes

1. **日志平台 SDK 改造（决议 C）**：日志平台 SDK 不再自行生成 TraceID，改为复用 Agent 注入的 MDC TraceID（检测 SkyWalking ContextManager，存在则取，不存在降级自生成兼容无 Agent 场景）。日志平台侧建立 trace_id 倒排索引。
2. **Trace×Metric 关联**：链路详情点击"关联指标"，同 service+时间窗（[start-5min, end+1min]，DD-03 §9.4）展示该 Trace 时间窗内指标曲线（读 npm_online_metric）。
3. **Trace×Log 关联**：链路详情点击"关联日志"，按 traceId 从日志平台拉取关联日志条目（GET /logs?traceId，DD-01 §3.7），侧栏展示，支持级别筛选与日志->Trace 反向定位。
4. **联合下钻上下文**：链->指->日志联合下钻面板（PRD REQ-M6-E1）。
5. **关联平台超时降级**：日志/指标接口超时或无数据标注"关联数据暂不可用"，不影响链路本身（PRD REQ-M6-W1）。
6. **日志 PII 脱敏**：日志展示侧关键字段掩码（手机号/身份证/卡号，DD-14）。

## Capabilities

### New Capabilities
- `log-correlation`: 日志平台 SDK 改造（复用 Agent TraceID）+ 按 TraceID 关联日志查询 + 日志->Trace 反向定位
- `three-domain-correlation`: Trace×Metric×Log 三域关联 + 联合下钻 + 超时降级
- `correlation-frontend`: 链路详情三域关联面板（指标曲线/日志侧栏/联合下钻）

### Modified Capabilities
- `trace-waterfall`: M3 瀑布图详情页的"关联指标/关联日志"入口按钮接通实际关联面板

## Impact

- **依赖**：`add-trace-query`（Trace 详情页 + 入口按钮）+ `add-storage-ingestion`（npm_online_metric）+ `chore-foundation`。外部：日志平台 SDK 改造 + 查询 API（议题 C，✅已定一期，改造工作量待评估，跨团队）。
- **下游解锁**：M14 统一视图（三域下钻）。
- **外部依赖**：日志平台团队 SDK 改造（议题 C/R5，跨团队，排期不在本团队控制；本 change Agent 侧 MDC 已由 M1 就绪，可独立验证 Agent 侧）。
- **对应 PRD 模块**：M6（三域关联诊断）。
- **设计依据**：DD-03 §7（模块六 日志分析）、DD-03 §9.4（三域时间窗）、DD-01 §3.7（/logs 接口）、DD-14（PII 脱敏）、系统设计说明书 §3.6、决议 C、DD-17 §6.6。
