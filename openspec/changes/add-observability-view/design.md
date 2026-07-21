## Context

M2 聚合索引（npm_online_metric/performe/db + trace_npm_jaeger）就绪。M14 构建统一大盘 + APM（健康评分/topN/热点/火焰图）。议题 B：火焰图/热点/健康评分纳入一期。设计见 DD-03 §9.1/§9.2 + DD-15 + 系统设计 §3.2/§3.4。

## Goals / Non-Goals

**Goals:**
- 统一大盘（健康仪表/指标卡/告警摘要/拓扑缩略图，下钻联动）
- 服务健康评分（4 维惩罚制，DD-03 §9.1）
- 接口/中间件 topN + 热点方法 + 火焰图（议题 B）
- 服务/实例指标图表

**Non-Goals:**
- 不做深度剖析 Profiling（M9 二期，只做议题 B 的热点/火焰图）
- 不做深度 RCA（M10 二期，只做议题 B 的健康评分）
- 不做 SLO 体系（M11 二期）
- 健康评分权重拖拉拽配置（二期，DD-03 §9.1，一期固定权重）

## Decisions

### DM1: 健康评分 4 维惩罚制（DD-03 §9.1 权威模型）
health = clamp(100-(E×0.35+L×0.25+R×0.20+A×0.20),0,100)
- E 错误率（35%）：min(100, errorReq_5min/totalReq_5min×200)，源 npm_online_performe
- L 时延劣化（25%）：clamp((p95-slo_p95)/slo_p95,0,1)×100，源 npm_online_performe
- R 资源使用率（20%）：max(cpu,mem,gc_pressure) 归一，源 npm_online_metric
- A 告警事件（20%）：min(100,Σsev_w×cnt)，P0=40/P1=20/P2=10/P3=5，源 npm_online_event
分级 90~100 健康/70~89 关注/40~69 警告/<40 严重。定时 5min 计算，缓存 Redis health:score(1m TTL)。一期固定权重，二期拖拉拽（t_health_model）。

### DM2: 火焰图代表 Trace 聚合（DD-03 §9.2）
不呈现每条 Trace，选桶内 Top-K 慢 Trace（K=100 默认可配）按 parent_span_id 聚合为火焰图层级 + self_time，避免全量渲染。源 trace_npm_jaeger，缓存 Redis 或落 npm_online_hotspot。

### DM3: 大盘下钻联动
任意视图（大盘卡片/健康排行/topN）可下钻至链路/指标/日志/拓扑详情（PRD REQ-M14-U1/E1）。健康分异常->跳转该服务拓扑/链路/告警视图。

### DM4: 前端 Canvas + Worker
火焰图自研 Canvas + Web Worker 层级聚合（DD-15 §7）；趋势图 ECharts；健康仪表 SVG。viz 库路由分包。

## Risks / Trade-offs

- **健康评分权重主观**：缓解--一期固定经验权重（错误+时延 60% 聚焦用户可感知），二期拖拉拽可调。
- **火焰图大数据计算**：缓解--代表 Trace Top-K 聚合 + Worker + 缓存，不全量渲染。
- **大盘聚合查询性能**：缓解--预聚合索引 + 多级缓存（dashboard:overview:cache 60s TTL）。
