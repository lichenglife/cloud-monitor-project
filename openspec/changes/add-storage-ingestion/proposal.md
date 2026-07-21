## Why

M1 采集层已将数据汇聚至上海 DC1 Kafka（otel-spans/metrics/logs/infra-metrics），但尚未落库--没有存储与接入层，M3 检索、M4 拓扑、M14 APM 均无数据可读。M2 是数据落地的关键环节：从 Kafka 消费、写入 ES 6 索引 + MySQL 元数据 + Redis 实时，满足存储 1 年、查询 P95<1s（PRD REQ-M2-U1/E1）。

技术选型已决议（议题 D）：MySQL+Redis+ES+Kafka 分层；ES 按天 Rollover 索引 + ILM 冷热分层（D9 本地 HDD 冷层）；聚合引擎一期用 Collector 内部流式聚合 + ES Ingest Pipeline/Transform（DD-02 §5 定稿）。本 change 落地 M2，对应研发任务 T-M2-1~T-M2-4。

## What Changes

1. **ingestion-service 消费写入**：从 Kafka 消费 otel-spans/metrics/logs/infra-metrics，经幂等去重后写入 ES 6 索引（trace_npm_jaeger 按天 Rollover + npm_online_metric/db/performe/topology/event/infra 聚合）+ MySQL 元数据 + Redis 实时指标。高吞吐 Go 实现，无状态可 HPA。
2. **ES 索引与 ILM**：创建 7 大索引 Mapping（DD-02 §3.3），trace_npm_jaeger 按天 Rollover（单日 6 主分片+1 副本），ILM 热(SSD,7d)->温->冷(本地 HDD,D9)->超 1 年删除；聚合索引 180d 降采样（1m->5m->1h）。
3. **聚合管道**：Collector 内部流式聚合 + ES Ingest Pipeline/Transform，原始 Span 派生 npm_online_metric/db/performe/topology（1m 桶预聚合，计算贴近存储）。
4. **查询服务基座**：trace-query-service 的 ES 检索能力（TraceID 精确查询 <1s P95），为 M3 瀑布图提供数据底座。
5. **慢查询/容量预警**：查询延迟 >1s 记录慢查询并触发容量预警（PRD REQ-M2-W2）。
6. **存储不可用快速失败**：存储节点不可用时写入快速失败并告警，不阻塞采集侧（Agent 缓冲兜底，PRD REQ-M2-W1）。

## Capabilities

### New Capabilities
- `ingestion`: Kafka 消费 -> ES/MySQL/Redis 写入管道（幂等去重、分 Topic 写对应索引、降级缓冲、高吞吐）
- `storage-indexing`: ES 7 大索引 + ILM 冷热分层 + 按天 Rollover + 聚合管道（预聚合 1m 桶）
- `trace-query-base`: TraceID 精确检索基座（ES trace_npm_jaeger 查询 P95<1s，为 M3 瀑布图供数）

### Modified Capabilities
<!-- 无，M2 为新能力 -->

## Impact

- **依赖**：`chore-foundation`（ingestion/trace-query 脚手架 + 契约 + ES/MySQL/Redis Repository 适配）+ `add-collection`（Kafka 数据源 otel-* Topic 已就绪）。前置硬依赖：M1 的 Kafka Topic 有数据。
- **下游解锁**：M3 链路检索（读 trace_npm_jaeger）、M4 拓扑（读 npm_online_topology）、M5.1 告警（读 npm_online_performe/metric）、M6 三域关联（读 npm_online_metric）、M14 APM（读 npm_online_metric/performe）、M15 权限（MySQL t_user/role/permission）。
- **外部依赖**：无（存储组件均在上海 DC1 biz 区内部）。
- **对应 PRD 模块**：M2（统一接入与存储）。
- **设计依据**：DD-02（数据模型/ES 7 索引 Mapping/Kafka Topic/Redis Key）、DD-04（存储方案/容量/ILM/D9 冷层/备份 RPO RTO）、DD-03（模块二/三 数据流）、系统设计说明书 §2.12（存储资源需求）、§2.14（清理/备份）、§3.1（ingestion 数据流）、决议 D/D9。
