## Why

故障需主动告警而非人工巡检--统一告警中心是 L1->L2 过渡的关键（PRD M5.1）。alert-service 消费预聚合指标流，规则评估、降噪收敛、携带 Trace 入口通知 Oncall，实现"告警后 ≤3 次点击进入链路"（PRD US-01）。

决议 D6：告警引擎=流式消费预聚合流 + 滑动窗口；异常检测算法 ①静态阈值一期 -> ②同环比/EWMA 二期 -> ③ML 远期。本 change 落地 M5.1 一期，对应研发任务 T-M5.1-1~T-M5.1-3。

## What Changes

1. **alert-engine 流式评估**：alert-service（Java）流式消费 Kafka 预聚合指标流（otel-metrics/聚合结果），内存滑动窗口评估（窗口=规则 for 时长），不压 ES（D6 方案 X）。无状态可 HPA，评估状态外置 Redis 支持重启续算。
2. **规则模型**：t_alert_rule（MySQL 热配置源），评估对象 service/instance/endpoint/cluster/room，指标源 npm_online_performe/metric/topology/event + DD-05 平台指标，条件类型阈值/同环比/缺失（心跳丢失），for 持续时间 1m/5m，严重级 P0-P3。
3. **静态阈值一期（D6 算法①）**：阈值规则全覆盖 P0/P1，检出延迟 P0≤10s、P1/P2/P3≤1-2min。
4. **降噪收敛**：静默（维护窗口 alert:silence）、收敛（同 rule+service+instance group_window=5m 仅 1 事件计数累加）、关联压缩（同根因多告警经拓扑归因为 1 条根因告警+受影响列表）、升级（P0 未确认 ack_timeout=10min 自动升级二线）。
5. **事件落库与投递**：事件落 npm_online_event（ES 365d）+ 推 alert-events(Kafka) -> 统一告警平台投递/值班/升级（Y 轴解耦）。状态闭环 firing/acknowledged/resolved。
6. **Trace 入口**：告警携带关联 Trace 入口，一键进入链路视图（PRD REQ-M5.1-U1）。
7. **平台自监控告警（Dogfooding）**：Collector 丢弃率/Kafka Lag/ES 写入错误/Agent 心跳丢失 -> 走同链路，P0 优先（DD-06 §7）。
8. **前端告警中心页**：规则 CRUD、事件列表、静默、Oncall 值班、Trace 入口跳转。

## Capabilities

### New Capabilities
- `alert-engine`: 流式评估引擎（消费预聚合流 + 滑动窗口 + 静态阈值 D6① + 评估状态外置续算）
- `alert-rules-noise`: 规则模型 + 降噪收敛（静默/收敛/关联压缩/升级）+ 事件落库投递 + 状态闭环
- `alert-frontend`: 告警中心页（规则 CRUD/事件列表/静默/Oncall/Trace 入口）

### Modified Capabilities
<!-- 无，M5.1 为新能力；alert-events Kafka Topic 已在 M2 定义 -->

## Impact

- **依赖**：`add-storage-ingestion`（npm_online_performe/metric/topology/event 聚合索引 + alert-events Kafka Topic 就绪）+ `add-topology`（关联压缩归因用拓扑）+ `chore-foundation`（alert-service Java 脚手架 + 前端 + 契约）。外部：统一告警平台投递通道（先用 mock，Q5 对齐后切）。
- **下游解锁**：M5 影响分析（告警->影响面跳转）、M14 统一视图（活跃告警卡片）、add-collection 自适应采样（M5.1 alert-engine 派生指标计算 sample_rate，回填 M1 的 D4 规则触发闭环）。
- **对应 PRD 模块**：M5.1（统一告警中心）。
- **设计依据**：DD-06（告警方案 v0.3 + 告警技术决议 D6）、DD-05（平台自监控指标）、DD-02（npm_online_event/t_alert_rule/alert:silence）、DD-01 §3.8（/alert/* 接口）、DD-15 §2（告警页）、DD-17 §6.7（横切验收）、决议 D6。
