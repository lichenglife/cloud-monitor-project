## ADDED Requirements

### Requirement: 流式评估引擎（D6 方案 X）
alert-engine MUST 流式消费 Kafka 预聚合指标流（otel-metrics/聚合结果），内存滑动窗口评估（窗口=规则 for 时长），MUST 不轮询 ES，MUST 无状态可 HPA（按规则/作用域分片），评估状态 MUST 外置 Redis 支持重启续算。

#### Scenario: 流式评估不压 ES
- **WHEN** 指标流入 alert-engine
- **THEN** 内存滑动窗口评估，不发起 ES 聚合查询，ES 仅写不额外查

#### Scenario: 重启续算
- **WHEN** alert-engine 重启
- **THEN** 从 Redis 恢复评估状态，不丢已累积窗口数据

### Requirement: 静态阈值一期 + 检测延迟 SLA（D6 算法①）
MUST 以静态阈值规则一期全覆盖 P0/P1 告警，检出延迟 P0 ≤10s（evaluate.interval=10s）、P1/P2/P3 ≤1-2min（=1m），缺失类（心跳丢失）heartbeat_timeout=60s + for 判定。

#### Scenario: P0 告警快速检出
- **WHEN** 核心服务错误率 >10% 持续 for 时长
- **THEN** P0 规则 ≤10s 检出并生成候选事件，5min 响应 SLA（电话+IM+大屏红闪）

### Requirement: 平台自监控告警 Dogfooding
Collector 丢弃率>Kafka Lag 阈值/ES 写入错误/Agent 心跳丢失/平台服务 /health degraded MUST 走同一条 alert-engine 链路，P0 优先，平台自身异常先于业务方被发现。

#### Scenario: 平台异常先于业务方暴露
- **WHEN** Collector 丢弃率超阈
- **THEN** alert-engine 触发 P0 告警，先于业务方感知采集异常
