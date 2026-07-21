# add-storage-ingestion（M2）实现任务清单

> **TDD 强制**（CLAUDE.md）：每功能点 RED->GREEN->REFACTOR，覆盖率 ≥80%。
> **涉及组件**：ingestion-service(Go) / trace-query-service(Go) / ES / MySQL / Redis。前置：add-collection 的 Kafka otel-* Topic 有数据。
> **设计依据**：DD-02 数据模型 / DD-04 存储方案 / DD-03 §2-3 数据流 / 决议 D/D9。

## 阶段 1：ES 7 索引与 ILM

### 1.1 索引 Mapping
- [ ] 1.1.1 [RED] 索引 Mapping 单测：7 索引字段类型断言（trace_npm_jaeger trace_id keyword/date_nanos、npm_online_performe p95_duration long、attributes flattened、statement_hash keyword）；UserID/OrderID 不入索引断言
- [ ] 1.1.2 [GREEN] 创建 7 索引 Mapping（DD-02 §3.3，维度 keyword 低基数）
- [ ] 1.1.3 [验收] 7 索引 Mapping 符合基数控制，attributes flattened 不膨胀；覆盖率 ≥80%

### 1.2 按天 Rollover + ILM
- [ ] 1.2.1 [RED] Rollover 单测：跨日写入->新索引 trace_npm_jaeger-YYYY.MM.DD；单日索引 6 主分片+1 副本
- [ ] 1.2.2 [GREEN] 配置 ILM 策略（hot SSD 7d -> warm 降副本 -> cold 本地 HDD D9 -> delete >1年）+ 按天 Rollover
- [ ] 1.2.3 [RED] 降采样单测：聚合索引 180d 后 1m->5m->1h 降采样
- [ ] 1.2.4 [GREEN] 配置聚合索引降采样 ILM
- [ ] 1.2.5 [验收] 跨日滚动正确，冷数据落本地 HDD（D9），1 年留存达标；覆盖率 ≥80%

## 阶段 2：ingestion-service 消费写入（Go）

### 2.1 Kafka 消费与分索引写入
- [ ] 2.1.1 [RED] 消费单测：消费 otel-spans->写 trace_npm_jaeger；otel-metrics->npm_online_metric；otel-logs->关联；otel-infra-metrics->npm_online_infra；分区按 trace_id hash 保序
- [ ] 2.1.2 [GREEN] 实现 Kafka 消费者（kafka-go，四 Topic 消费组，分索引路由）
- [ ] 2.1.3 [RED] 元数据写 MySQL 单测：service/instance/env 元数据写 t_engineering 等表
- [ ] 2.1.4 [GREEN] 实现 MySQL 元数据写入（批量 insert）
- [ ] 2.1.5 [RED] 实时指标写 Redis 单测：metric:rt:{service} ZSet 更新
- [ ] 2.1.6 [GREEN] 实现 Redis 实时指标写入（pipeline）
- [ ] 2.1.7 [验收] 四 Topic 数据分写对应索引 + MySQL + Redis，类型一一对应；覆盖率 ≥80%

### 2.2 幂等去重与高吞吐
- [ ] 2.2.1 [RED] 幂等单测：trace_id+span_id 重复消费仅落库一次（Redis 二次校验 + 本地布隆过滤）
- [ ] 2.2.2 [GREEN] 实现幂等二次校验（Redis trace:dedup + 布隆过滤器预过滤）
- [ ] 2.2.3 [RED] 批量写单测：bulk ES + 批量 MySQL insert + Redis pipeline，I/O 往返降低
- [ ] 2.2.4 [GREEN] 实现批量写入
- [ ] 2.2.5 [RED] HPA 单测：Kafka Lag + CPU 超阈扩缩（模拟）
- [ ] 2.2.6 [GREEN] 配置 HPA 指标
- [ ] 2.2.7 [验收] 重复消费无脏数据，批量写入吞吐达标，HPA 生效；覆盖率 ≥80%

### 2.3 存储不可用快速失败
- [ ] 2.3.1 [RED] 快速失败单测：ES 不可用时写入快速失败+告警，不阻塞消费
- [ ] 2.3.2 [GREEN] 实现存储不可用快速失败 + 告警钩子
- [ ] 2.3.3 [验收] ES 故障 ingestion 快速失败，Collector 缓冲兜底，恢复重传零丢失；覆盖率 ≥80%

## 阶段 3：聚合管道（预聚合 1m 桶）

### 3.1 ES Ingest Pipeline / Transform
- [ ] 3.1.1 [RED] 聚合单测：原始 span->npm_online_performe 1m 桶（call_count/p50/p95/p99/error_count/throughput_tps/apdex）；npm_online_db（db_system/call_count/slow_query_count）；npm_online_topology（source-target 边）
- [ ] 3.1.2 [GREEN] 实现 ES Ingest Pipeline/Transform（1m 桶预聚合，计算贴近存储）
- [ ] 3.1.3 [RED] Collector 流式聚合单测：Collector 侧预聚合后写聚合索引
- [ ] 3.1.4 [GREEN] 实现 Collector 流式聚合 processor
- [ ] 3.1.5 [验收] 原始 span 派生聚合索引，查询无需实时重聚合；覆盖率 ≥80%

## 阶段 4：trace-query-service 检索基座（Go）

### 4.1 TraceID 精确检索 P95<1s
- [ ] 4.1.1 [RED] 检索单测：TraceID 精确查 trace_npm_jaeger，返回 span 列表按 parent_span_id 树形；P95<1s（一期规模压测）
- [ ] 4.1.2 [GREEN] 实现 trace-query ES 检索 gRPC（QueryTrace RPC）+ Repository 抽象
- [ ] 4.1.3 [RED] 多级缓存单测：Redis trace:query:result(300s) + Caffeine 本地缓存命中/失效
- [ ] 4.1.4 [GREEN] 实现多级缓存
- [ ] 4.1.5 [验收] TraceID 查询 P95<1s，热点 Trace 走缓存；覆盖率 ≥80%

### 4.2 慢查询与容量预警
- [ ] 4.2.1 [RED] 慢查询单测：查询 >1s 记录慢查询日志 + 触发容量预警告警
- [ ] 4.2.2 [GREEN] 实现慢查询记录 + 容量预警钩子（接 alert-service DD-06）
- [ ] 4.2.3 [验收] 慢查询记录且不影响其他查询；覆盖率 ≥80%

### 4.3 ES 故障查询降级
- [ ] 4.3.1 [RED] 降级单测：ES 故障返回降级提示 + 已缓存数据仍可返回，不抛 500
- [ ] 4.3.2 [GREEN] 实现查询降级（返回缓存/降级视图，标注'数据暂不可用'）
- [ ] 4.3.3 [验收] ES 故障不雪崩；覆盖率 ≥80%

## 阶段 5：备份与清理

- [ ] 5.1 [RED] 备份单测：ES 每日 snapshot（RPO<1h）；MySQL 全量+binlog（RPO<5min）；Redis AOF+RDB
- [ ] 5.2 [GREEN] 配置备份策略（DD-04 §4）
- [ ] 5.3 [RED] 清理单测：Kafka trace 7d/metric 3d 滚动；ES 冷索引 >30d 迁冷；MySQL 审计 180d 清理
- [ ] 5.4 [GREEN] 配置清理策略（DD-02/系统设计 §2.14.1）
- [ ] 5.5 [验收] 备份可恢复，清理不误删热数据；覆盖率 ≥80%

## 阶段 6：跨组件联调验收（Gate，对齐 DD-17 §6.2 M2）

- [ ] 6.1 [联调] add-collection Kafka otel-spans -> ingestion -> ES trace_npm_jaeger 可查
- [ ] 6.2 [联调] 聚合管道：原始 span -> npm_online_performe/topology 派生正确
- [ ] 6.3 [联调] trace-query TraceID 查询 -> 返回完整 span 树
- [ ] 6.4 [联调] ES 故障 -> ingestion 快速失败 + Collector 缓冲 -> 恢复重传零丢失
- [ ] 6.5 [联调] 多级缓存命中 -> ES 查询量下降
- [ ] 6.6 [验收] DD-17 §6.2 M2：分层存储/≥1年/P95<1s/慢查询预警/降级；MySQL+Redis+ES+Kafka 分层验证
- [ ] 6.7 [验收] 压测 3×（D10）：QPS≈500 下 P95<1s、存储写入不丢、Kafka Lag 可控
- [ ] 6.8 [验收] 覆盖率 ≥80%，全量单测绿

