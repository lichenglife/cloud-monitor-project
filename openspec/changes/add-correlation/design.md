## Context

M1 已在 Agent 侧 MDC 注入 TraceID（决议 C Agent 侧）。M3 已有 Trace 详情页 + "关联指标/日志"入口按钮（占位）。M6 接通实际关联：日志平台 SDK 改造（复用 TraceID）+ Trace×Metric×Log 关联查询 + 前端面板。设计见 DD-03 §7 + 系统设计 §3.6。

## Goals / Non-Goals

**Goals:**
- 日志平台 SDK 改造复用 Agent TraceID（决议 C）
- Trace×Metric（同 service+时间窗）、Trace×Log（按 traceId）关联
- 联合下钻 + 超时降级 + 日志 PII 脱敏

**Non-Goals:**
- 不重建日志平台（复用现有日志平台存储/检索，只改 SDK + 查询桥接）
- 不做日志平台内容脱敏（本平台只注入 TraceID，日志内容由日志平台自身脱敏规则处理，系统设计 §2.19）
- 不做日志全文检索 UI（日志平台已有，本平台只做关联展示）

## Decisions

### DM1: 日志 SDK 复用 Agent TraceID（决议 C）
日志平台 SDK 改造：检测 SkyWalking ContextManager 是否存在 -> 存在则取 Agent traceId -> 不存在降级自生成（兼容无 Agent 场景）。保证同一请求日志 traceId = 链路 traceId。日志平台侧建 trace_id 倒排索引。

### DM2: 三域关联时间窗（DD-03 §9.4）
Metric 关联窗口 [start-5min, end+1min]（环绕 Trace 时间）；Log 关联按精确 traceId 匹配（无需时间窗）。窗口可配（默认 5min）。

### DM3: 关联超时降级（PRD REQ-M6-W1）
日志/指标接口超时或无数据 -> 标注"关联数据暂不可用"，不影响链路本身。日志查询限频（同 TraceID 每分钟 ≤20 次）。

### DM4: 日志 PII 脱敏（展示侧）
日志展示侧对手机号/身份证/卡号正则掩码（DD-14），原始不入本平台（日志平台自有脱敏）。

## Risks / Trade-offs

- **日志平台 SDK 改造跨团队依赖**（议题 C/R5 高危）：排期不在本团队控制。缓解--Agent 侧 MDC（M1）先就绪可独立验证；SDK 改造方案早对齐，本 change 提供 SDK 改造需求规格 + 联调 mock。
- **关联查询性能**：日志平台按 traceId 查询延迟。缓解--结果缓存 Redis trace:logs:cache(120s TTL) + 限频。
- **时间窗不准**：Metric 关联窗口固定 5min 可能漏数据。缓解--窗口可配，按 trace 实际 start/end 环绕。
