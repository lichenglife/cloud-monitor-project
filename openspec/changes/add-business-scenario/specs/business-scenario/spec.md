## ADDED Requirements

### Requirement: 业务场景标签设置与 Agent 附加
MUST 支持管理端为入口 URL 设置业务场景标签（URL pattern 通配，存 t_biz_scenario + Redis biz:url:tag），Agent 采集 HTTP Span 时若 request.url 匹配 pattern MUST 在 Span attribute 附加 biz_scenario 标签（不入 Span 名，控基数）。

#### Scenario: URL 匹配附加标签
- **WHEN** Agent 采集 HTTP Span 且 request.url 匹配 /api/payment/*
- **THEN** Span attribute 附加 biz_scenario="支付交易"，Span 名不变（控基数）

### Requirement: 按业务场景聚合查询
MUST 支持按业务场景标签筛选时聚合展示该标签下关联链路关键指标（QPS/错误率/耗时）+ Top 慢 Trace，MUST 可跳转链路检索（自动带入 filter）。

#### Scenario: 按场景聚合查看
- **WHEN** 用户选择"支付交易"业务场景
- **THEN** 聚合展示该场景 QPS/错误率/耗时 + Top 慢 Trace，可跳转链路检索页（自动带入 biz_scenario filter）
