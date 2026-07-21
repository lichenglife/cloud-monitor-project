## Why

M2/M3/M4 已落库链路/指标/拓扑，但缺乏统一视图与性能监测--用户需在多页面间跳转拼凑健康状况。M14 构建统一可观测大盘 + APM 性能监测（服务健康评分、接口 topN、热点方法、火焰图），实现"任意视图可下钻至链路/指标/日志"（PRD REQ-M14-U1）。议题 B 决议：火焰图/热点方法/服务健康评分纳入一期。

本 change 落地 M14，对应研发任务 T-M14-1~T-M14-4。

## What Changes

1. **统一大盘**：全局健康仪表、核心指标卡（错误率/P95/吞吐）、告警摘要、多中心健康、服务健康评分排行，任意视图可下钻（DD-01 GET /dashboard/overview）。
2. **服务健康评分（议题 B，DD-03 §9.1）**：4 维惩罚制（错误率35%/时延25%/资源20%/告警20%），health=clamp(100-(E×0.35+L×0.25+R×0.20+A×0.20),0,100)，分级 90/70/40，定时 5min 计算，缓存 Redis health:score(1m TTL)。
3. **接口/中间件 topN（DD-01 §3.5）**：按吞吐量/错误率/耗时 topN 排行（读 npm_online_performe/db）。
4. **热点方法分析（议题 B，DD-03 §9.2）**：按 operation_name/代码级 span 聚合 self_time+调用次数 TopN（读 npm_online_performe + code_level span）。
5. **火焰图（议题 B，DD-03 §9.2）**：以 trace_npm_jaeger 为源，按 parent_span_id 在 (service,1m bucket) 构建调用层级，选代表 Trace（桶内 Top-K 慢 Trace，K=100）聚合为火焰图，不呈现每条 Trace。
6. **服务/实例指标图表**：JVM/资源/接口指标图表（读 npm_online_metric），服务级/实例级切换。
7. **前端大盘 + APM 页**：Vue3，ECharts 趋势/仪表，火焰图自研 Canvas + Web Worker。

## Capabilities

### New Capabilities
- `health-score`: 服务健康评分（4 维惩罚制 DD-03 §9.1，定时计算 + 缓存）
- `apm-analysis`: 接口/中间件 topN + 热点方法分析 + 火焰图（议题 B，DD-03 §9.2）
- `observability-dashboard`: 统一大盘 + 服务/实例指标图表 + 下钻联动

### Modified Capabilities
<!-- 无，M14 为新能力；读 M2 聚合索引 -->

## Impact

- **依赖**：`add-storage-ingestion`（npm_online_metric/performe/db + trace_npm_jaeger）+ `add-trace-query`（火焰图源 trace）+ `add-topology`（大盘拓扑缩略图）+ `add-alert-center`（活跃告警摘要）+ `chore-foundation`。
- **下游解锁**：统一可观测视图是各模块数据的汇总出口。
- **对应 PRD 模块**：M14（统一可观测视图与性能监测）。
- **设计依据**：DD-15 §2（大盘/APM 页）、DD-03 §9.1/§9.2（健康评分/火焰图热点）、DD-01 §3.3/§3.5（/dashboard /apm /services 接口）、系统设计说明书 §3.2/§3.4、议题 B、DD-17 §6.2/§6.4。
