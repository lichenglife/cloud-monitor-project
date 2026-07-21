## MODIFIED Requirements

### Requirement: Agent HTTP Span 附加 biz_scenario
agent-instrumentation（M1）Java/Go Agent MUST 扩展在 HTTP Span 采集时匹配 t_biz_scenario URL pattern，命中则在 Span attribute 附加 biz_scenario 标签（低基数枚举，可索引）。

#### Scenario: Agent 侧标签附加
- **WHEN** M17 配置 URL pattern 后 Agent 采集匹配的 HTTP 请求
- **THEN** Span attribute 含 biz_scenario 标签，供 M17 按场景聚合查询
