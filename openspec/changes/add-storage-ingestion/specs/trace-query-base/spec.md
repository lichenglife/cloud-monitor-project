## ADDED Requirements

### Requirement: TraceID 精确检索 P95<1s
trace-query-service MUST 支持 TraceID 精确检索（查 trace_npm_jaeger），单条 Trace 详情响应 P95 <1s（一期规模），MUST 经多级缓存（Redis trace:query:result 300s TTL + Caffeine 本地）降低重复 ES 查询。

#### Scenario: TraceID 精确查询达标
- **WHEN** 用户输入 TraceID 查询详情
- **THEN** 响应 P95 <1s，热点 Trace 走缓存命中，ES 查询仅未命中时发生

### Requirement: 慢查询记录与容量预警
查询延迟 >1s 时 MUST 记录慢查询并触发容量预警，MUST 不影响其他查询。

#### Scenario: 慢查询预警
- **WHEN** 某 Trace 查询延迟 >1s
- **THEN** 记录慢查询日志并触发容量预警告警（DD-06），其他查询不受影响

### Requirement: 存储节点不可用查询降级
ES 不可用时查询 MUST 返回降级视图/缓存提示，MUST 不雪崩，MUST 标注"数据暂不可用"。

#### Scenario: ES 故障查询降级
- **WHEN** ES 故障时用户查询 Trace
- **THEN** 返回降级提示"数据暂不可用，请稍后重试"，不抛 500 雪崩，已缓存数据仍可返回
