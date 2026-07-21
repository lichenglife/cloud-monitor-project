## ADDED Requirements

### Requirement: Kafka 消费与分索引写入
ingestion-service MUST 从 Kafka 消费 otel-spans/metrics/logs/infra-metrics 四 Topic，按数据类型分写对应 ES 索引（span->trace_npm_jaeger、metric->npm_online_metric、db->npm_online_db、performe->npm_online_performe、topology->npm_online_topology、event->npm_online_event、infra->npm_online_infra）+ MySQL 元数据 + Redis 实时指标，MUST 支持消费组横向扩展。

#### Scenario: 四 Topic 数据分写对应索引
- **WHEN** Collector 将 span/metric/log/infra 数据写入对应 Kafka Topic
- **THEN** ingestion-service 消费后分别写入对应 ES 索引，数据类型与索引一一对应，无错写

### Requirement: 幂等去重与高吞吐
ingestion-service MUST 对 trace_id+span_id 幂等去重（二次校验防重复消费），MUST 批量写 ES/MySQL/Redis 降低 I/O 往返，MUST 无状态可 HPA（按 Kafka Lag + CPU 扩缩）。

#### Scenario: 重复消费不产生脏数据
- **WHEN** Kafka 因 rebalance 重复投递同一批 span
- **THEN** ingestion 经幂等校验仅落库一次，ES 无重复文档

### Requirement: 存储不可用快速失败不阻塞采集
ES/MySQL 不可用时 ingestion MUST 快速失败并告警，MUST 不阻塞采集侧（Collector 本地磁盘缓冲兜底），恢复后重传不丢。

#### Scenario: ES 故障快速失败
- **WHEN** ES 集群不可用
- **THEN** ingestion 写入快速失败并触发告警，Collector 本地缓冲继续接收 Agent 数据，ES 恢复后重传零丢失
