## Context

M2 已落库预聚合索引（npm_online_performe/metric/topology/event）+ alert-events Kafka Topic。M5.1 的 alert-service 流式消费评估。D6 决议：方案 X 流式 + 算法①静态阈值一期。设计见 DD-06 + 告警技术决议。

数据流（DD-06 告警技术决议 §3.2）：
```
预聚合指标流(Kafka otel-metrics/聚合结果) -> alert-engine 订阅(按规则/作用域分片 HPA)
  -> 规则加载(t_alert_rule+Redis) -> 滑动窗口评估(窗口=for 时长) -> 命中候选事件
  -> 收敛(group_window=5m) -> 落 npm_online_event(ES 365d) + 推 alert-events(Kafka)
  -> 统一告警平台投递/值班/升级 -> 用户(告警页/大屏)
```

## Goals / Non-Goals

**Goals:**
- alert-engine 流式消费预聚合流 + 滑动窗口评估（D6 方案 X），不压 ES
- 静态阈值一期全覆盖 P0/P1（D6 算法①），检出延迟 P0≤10s
- 降噪（静默/收敛/关联压缩/升级）
- 事件落库 + 投递 + 状态闭环 + Trace 入口
- 平台自监控 Dogfooding 告警
- 前端告警中心页

**Non-Goals:**
- 不实现 ②同环比/EWMA（二期）③ML（远期）
- 不实现统一告警平台侧投递通道（外部系统，本平台只推事件）
- 不做深度 RCA（M10 二期）

## Decisions

### DM1: 流式消费预聚合流（D6 方案 X）
alert-engine 订阅 Kafka 预聚合 1m 指标流（或聚合结果），内存滑动窗口评估（窗口=规则 for 时长），不轮询 ES。评估状态外置 Redis 支持重启续算。无状态按规则/作用域分片 HPA。

### DM2: 静态阈值一期 + 检测延迟 SLA（D6 算法①）
P0 evaluate.interval=10s（检出 ≤10s + 收敛 5m 内 1 事件）；P1/P2/P3=1m（≤1-2min）；缺失类 heartbeat_timeout=60s + for 判定。一期阈值规则全覆盖 P0/P1 保"不漏"。

### DM3: 降噪四件套
静默（alert:silence 维护窗口）、收敛（rule+service+instance group_window=5m 仅 1 事件计数累加）、关联压缩（同根因经 M4 拓扑归因为 1 条根因告警+受影响列表）、升级（P0 ack_timeout=10min 自动升级二线）。

### DM4: 评估与投递解耦（Y 轴）
alert-engine 只"判定+落库+推事件"（npm_online_event + alert-events Kafka），统一告警平台负责投递/值班/升级。Webhook 鉴权 HMAC 签名。状态闭环 firing/acknowledged/resolved（恢复事件反向更新 resolved_time）。

### DM5: Trace 入口
告警事件携带关联 trace_id（若来自 Trace/Metric 告警），前端一键跳转链路视图（PRD REQ-M5.1-U1）。

## Risks / Trade-offs

- **静态阈值误告率**（D6 算法①中误告率）：缓解--一期保"不漏"，阈值可配；二期 ②同环比/EWMA 降误告。
- **预聚合流中断**：缓解--降级短期回退定时拉取 ES 最近窗口（保底不漏），恢复切回流式（DD-06 告警技术决议 §8）。
- **统一告警平台接口未对齐**（Q5）：缓解--mock 占位，alert-events Kafka 先就绪，对齐后切真实投递。
- **关联压缩依赖拓扑完整**：缓解--拓扑不全时降级为按 rule+service 收敛（不归因）。
