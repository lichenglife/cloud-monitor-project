## ADDED Requirements

### Requirement: ES 7 大索引与 Mapping
MUST 创建 DD-02 §3.3 定义的 7 大索引（trace_npm_jaeger + npm_online_metric/db/performe/topology/event/infra），Mapping 遵循维度字段 keyword 低基数、attributes 用 flattened、statement_hash 替代 SQL 原文，MUST 禁 UserID/OrderID 入索引。

#### Scenario: 索引 Mapping 符合基数控制
- **WHEN** 写入含 UserID 的 span attribute
- **THEN** UserID 仅入 attributes(flattened) 不建索引，索引基数不膨胀

### Requirement: 按天 Rollover 与 ILM 冷热分层
trace_npm_jaeger MUST 按天 Rollover（`-YYYY.MM.DD`，单日 6 主分片+1 副本，单分片 <50GB），MUST 配 ILM 策略 hot(SSD,7d)->warm(降副本)->cold(本地大容量 HDD，D9)->delete(>1 年)，聚合索引 180d 降采样（1m->5m->1h）。

#### Scenario: 索引按天滚动
- **WHEN** 跨日写入 span
- **THEN** 新一日数据写入新索引 trace_npm_jaeger-YYYY.MM.DD，旧索引按 ILM 迁移至温/冷层

#### Scenario: 冷数据落本地 HDD
- **WHEN** 索引超 30 天
- **THEN** ILM 将其迁移至本地大容量 HDD 冷层（D9），查询少可接受中速

### Requirement: 聚合管道预聚合 1m 桶
MUST 经 Collector 流式聚合 + ES Ingest Pipeline/Transform，将原始 Span 派生为 npm_online_metric/db/performe/topology 的 1m 桶预聚合结果，MUST 计算贴近存储避免查询时实时重聚合。

#### Scenario: 原始 span 派生聚合索引
- **WHEN** 原始 span 写入 trace_npm_jaeger
- **THEN** 聚合管道按 (service, 1m bucket) 派生 npm_online_performe（call_count/p95/error_count 等），查询时直接读聚合索引无需重算
