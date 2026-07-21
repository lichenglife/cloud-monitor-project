## Context

M1 Agent 已采集 HTTP Span。M17 为入口 URL 设业务标签，Agent 匹配 pattern 附加 biz_scenario attribute，按标签聚合查询。设计见系统设计 §3.5 + DD-02 t_biz_scenario。

## Goals / Non-Goals

**Goals:**
- URL pattern -> 业务场景标签设置（t_biz_scenario + Redis biz:url:tag）
- Agent 侧 HTTP Span 附加 biz_scenario attribute
- 按标签聚合查询（指标 + Top 慢 Trace）
- 前端场景管理 + 看板

**Non-Goals:**
- 不重建链路检索（复用 M3，本变更只加标签筛选维度）
- 不做业务拓扑（M4 通用拓扑已覆盖）

## Decisions

### DM1: URL pattern 通配匹配
t_biz_scenario.url_pattern 支持通配（如 /api/payment/*），Agent 采集 HTTP Span 时匹配 request.url，命中则在 Span attribute 附加 biz_scenario 标签（不入 Span 名，控基数，DD-03.5）。

### DM2: 按标签聚合
查询按 span.attribute.biz_scenario 聚合（ES），展示 QPS/错误率/耗时 + Top 慢 Trace，可跳转 M3 链路检索（自动带入 filter）。

## Risks / Trade-offs

- **URL pattern 匹配性能**：Agent 侧每请求匹配。缓解--pattern 缓存内存 + 低基数编译为正则一次。
- **标签基数控制**：biz_scenario 为低基数枚举（场景名有限），可索引不爆炸。
