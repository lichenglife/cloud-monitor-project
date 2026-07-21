## Why

M2 已落库 ES trace_npm_jaeger，但用户无法查询--没有链路检索 UI，Trace 数据不可用。M3 是故障定位的核心工具（PRD US-03：研发下钻调用链到方法/SQL/代码栈定位慢请求）。M2 的 trace-query-base 只提供 TraceID 精确检索 gRPC，M3 在其上构建前端瀑布图、代码栈、方法/SQL 下钻、错误/慢筛选。

本 change 落地 M3，对应研发任务 T-M3-1~T-M3-3。

## What Changes

1. **Trace 列表查询**：按 service/status/time/duration/httpStatus 多条件筛选（ES bool query 组合过滤），分页返回 Trace 列表（DD-01 GET /traces）。
2. **瀑布图详情**：Trace 详情按 parent_span_id 树形组装瀑布图，Span 颜色编码（正常蓝/慢橙/错误红/跨中心紫），点击 Span 下钻方法/SQL/代码栈（DD-01 GET /traces/{traceId}）。
3. **代码栈与堆栈**：代码栈树形展示（包.类.方法 - 耗时 - 占比），异常自动记录堆栈红色高亮（PRD REQ-M3-E2/U2）。
4. **下钻方法/SQL**：点击 Span 下钻展示方法级/SQL 级明细与耗时占比（PRD REQ-M3-E1）。
5. **错误/慢调用筛选**：按错误码/慢调用阈值筛选与排序（DD-01 GET /traces/error-analysis）。
6. **链路完整性标注**：跨中心 Span 缺失标注"链路断裂"，跨中心数据缺失标注"跨中心数据缺失"（PRD REQ-M3-W1）。
7. **超长 Trace 截断**：采集 5 万/展示 1 万 spans，超出保留错误/慢 span + 采样子集，标记 truncated=true（DD-03 §9.4）。

## Capabilities

### New Capabilities
- `trace-search`: Trace 列表多条件筛选查询 + 错误/慢调用分析（ES bool query，分页）
- `trace-waterfall`: Trace 详情瀑布图组装 + 代码栈 + 方法/SQL 下钻 + 链路完整性标注 + 超长截断
- `trace-frontend`: 链路检索页 + 瀑布图详情页前端（Vue3，Canvas 虚拟化，Web Worker 建树）

### Modified Capabilities
- `trace-query-base`: M2 的 TraceID 精确检索扩展为列表筛选 + 详情组装（本变更深化 trace-query-service）

## Impact

- **依赖**：`add-storage-ingestion`（ES trace_npm_jaeger + npm_online_metric 已就绪）+ `chore-foundation`（trace-query 脚手架 + 前端 + 契约）。
- **下游解锁**：M6 三域关联（Trace 详情页的"关联指标/日志"入口）、M14 APM（火焰图复用 trace 数据）。
- **外部依赖**：无。
- **对应 PRD 模块**：M3（链路追踪与检索）。
- **设计依据**：DD-01 §3.4（/traces 接口）、DD-03 §4（模块三 全链路分析）、DD-03 §9.4（超长截断+三域时间窗）、系统设计说明书 §3.3、DD-15 §2/§7（瀑布图 Canvas+Worker）、DD-17 §6.3（M3 验收）。
