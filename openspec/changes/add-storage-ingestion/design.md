## Context

M1 已将数据汇聚至上海 DC1 Kafka（otel-spans/metrics/logs/infra-metrics）。M2 负责消费落库，是 M3~M14 的数据底座。设计已由 DD-02（数据模型）、DD-04（存储方案）详定，本设计固化关键决策。

数据流（DD-03 §1）：
```
Kafka(上海 DC1: otel-spans/metrics/logs/infra-metrics)
   │ ingestion-service 消费（Go，无状态 HPA）
   ▼
幂等去重(trace_id+span_id) -> 写 ES 6 索引 + MySQL 元数据 + Redis 实时
   │ 聚合管道（Collector 流式聚合 + ES Ingest/Transform）
   ▼
trace_npm_jaeger(按天Rollover) + npm_online_metric/db/performe/topology/event/infra(1m桶预聚合)
```

## Goals / Non-Goals

**Goals:**
- ingestion-service 从 Kafka 消费写 ES/MySQL/Redis，高吞吐、幂等、降级
- ES 7 大索引 + ILM（按天 Rollover，D9 本地 HDD 冷层，1 年留存）
- 聚合管道预聚合 1m 桶（计算贴近存储）
- trace-query-service ES 检索 P95<1s（TraceID 精确查询）
- 存储不可用快速失败不阻塞采集

**Non-Goals:**
- 不实现瀑布图/代码栈 UI（M3）
- 不实现拓扑计算（M4，topology-service 消费 otel-spans）
- 不实现告警评估（M5.1）
- 不实现三域关联（M6）
- 不做容量精确测算（DD-04 §6 待评审，本变更用 DD-02 基线值）

## Decisions

### DM1: ingestion-service 高吞吐消费（Go）
- 消费 otel-spans/metrics/logs/infra-metrics 四 Topic，分区按 trace_id hash 保序，消费组横向扩展（X 轴）。
- 幂等：trace_id+span_id 去重（Redis trace:dedup 已在 Collector 做，ingestion 二次校验防重复消费）。
- 批量写 ES（bulk API）+ MySQL（批量 insert）+ Redis（pipeline），降低 I/O 往返。
- 无状态，K8S HPA 按 Kafka Lag + CPU 扩缩。

### DM2: ES 7 索引 + 按天 Rollover + ILM（DD-02 §3 / DD-04 §3）
- trace_npm_jaeger：按天 Rollover（`-YYYY.MM.DD`），单日索引 6 主分片+1 副本（单分片 <50GB）。
- 6 聚合索引：3 主分片+1 副本，1m 桶预聚合。
- ILM：hot(SSD,7d)->warm(降副本)->cold(本地大容量 HDD,D9)->delete(>1年)；聚合索引 180d 降采样（1m->5m->1h）。
- 维度字段全 keyword 低基数；attributes 用 flattened；statement_hash 替代 SQL 原文。

### DM3: 聚合管道（DD-02 §5 定稿）
- 一期：Collector 内部流式聚合 + ES Ingest Pipeline/Transform，原始 Span 派生 npm_online_metric/db/performe/topology（1m 桶）。
- Flink 作二期高吞吐扩展（待评估），本变更不引入。

### DM4: 查询 P95<1s 保障
- TraceID 精确查询走 trace_npm_jaeger keyword 字段（高基数 hash 但精确查询快）。
- 聚合查询走预聚合索引（轻量 term 聚合），不实时重聚合。
- 多级缓存：Redis trace:query:result 缓存热点 Trace 详情（DD-02 #trace:query:result，300s TTL）+ Caffeine 本地缓存。
- 慢查询（>1s）记录 + 容量预警（PRD REQ-M2-W2）。

### DM5: 存储不可用快速失败（可靠性）
- ES/MySQL 不可用时 ingestion 快速失败并告警，不阻塞采集（Collector 本地磁盘缓冲兜底，DD-07）。
- 降级优先级：保采集 > 保聚合展示（DD-07 §5）。

## Risks / Trade-offs

- **ES P95<1s 达标风险**（R3 高危）：数据量/索引设计影响。缓解：按天 Rollover 控单索引规模、预聚合、多级缓存、冷热分层；Gate2.5 压测验证（D10 3×/10×）。
- **聚合管道复杂度**：Collector 流式聚合 + ES Transform 双路。缓解：一期数据量可控（试点 300 工程），先 ES Ingest Pipeline 轻量方案。
- **存储成本**（F8 隐性依赖）：试点 100% 采样（D3）致 ~315TB/年原始。缓解：ILM 冷热分层 + 降采样 + 自适应采样（D4，M1 已落地）。
- **幂等二次校验开销**：ingestion 再校验去重增加 Redis 调用。缓解：批量 pipeline + 本地布隆过滤器预过滤。
