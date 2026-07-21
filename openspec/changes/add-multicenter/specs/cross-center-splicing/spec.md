## ADDED Requirements

### Requirement: 跨中心 Trace 拼接
查询时 MUST 将同一 128bit trace_id 跨中心 Span 拼接为完整调用链，按 parent-child 因果关系 + start_time 排序重组，各 Span MUST 携带 datacenter_id 标注来源中心，MUST 经上海 DC1 ES 汇聚查询。

#### Scenario: 跨中心调用拼接完整链
- **WHEN** 同一请求经上海服务->苏州服务（同 trace_id 跨中心）
- **THEN** 查询时拼接为完整调用链，按因果/时间排序，Span 标注来源中心

### Requirement: 多中心查询通道与隔离标注
用户 MUST 经上海 DC1 ES/MySQL 查询跨中心汇聚数据，某中心 biz/dm 隔离时该中心 Agent 仅经 dmz 代理上报（PRD REQ-M7-S1），前端 MUST 按 datacenter_id 标注来源。

#### Scenario: biz/dm 隔离经代理上报
- **WHEN** 某中心 biz/dm 不通
- **THEN** 该中心 Agent 经 dmz 代理上报（M1 管道），查询时数据已汇聚至上海 DC1，前端标注来源中心
