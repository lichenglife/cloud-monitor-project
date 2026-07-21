## ADDED Requirements

### Requirement: 规则模型与配置
MUST 支持 t_alert_rule 规则 CRUD（评估对象 service/instance/endpoint/cluster/room，指标源 npm_online_performe/metric/topology/event + DD-05 平台指标，条件阈值/同环比/缺失，for 1m/5m，严重级 P0-P3，作用域全局/项目/集群/工程），MUST 支持规则启停（/alert/rules/{id}/enable）。

#### Scenario: 规则 CRUD 与启停
- **WHEN** 运维创建 P0 规则"核心服务错误率>10% for 1m"
- **THEN** 规则存 t_alert_rule 并加载至 alert-engine，可启停

### Requirement: 降噪收敛
MUST 支持静默（维护窗口 alert:silence）、收敛（同 rule+service+instance group_window=5m 仅 1 事件计数累加）、关联压缩（同根因经拓扑归因为 1 条根因告警+受影响列表）、升级（P0 未确认 ack_timeout=10min 自动升级二线）。

#### Scenario: 告警风暴收敛
- **WHEN** 同一 rule+service+instance 在 5min 内多次触发
- **THEN** 收敛为 1 事件计数累加，不产生告警风暴

#### Scenario: 同根因关联压缩
- **WHEN** 某 DB 故障触发多条服务告警
- **THEN** 经拓扑归因为 1 条根因告警 + 受影响服务列表

### Requirement: 事件落库投递与状态闭环
事件 MUST 落 npm_online_event（ES 365d）+ 推 alert-events(Kafka) -> 统一告警平台投递，MUST 携带关联 Trace 入口，状态 MUST 闭环 firing/acknowledged/resolved（恢复事件反向更新 resolved_time），投递鉴权 HMAC 签名。

#### Scenario: 告警携带 Trace 入口
- **WHEN** 来自 Trace/Metric 的告警触发
- **THEN** 事件携带 trace_id，前端可一键跳转链路视图（PRD REQ-M5.1-U1）

#### Scenario: 恢复状态闭环
- **WHEN** 告警条件解除
- **THEN** 反向更新 resolved_time，状态转 resolved，避免"已恢复仍红"
