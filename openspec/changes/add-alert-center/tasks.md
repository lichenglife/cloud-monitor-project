# add-alert-center（M5.1）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%（告警属 ≥80% 红线）。前置：add-storage-ingestion 预聚合索引 + alert-events Kafka 就绪。
> **涉及组件**：alert-service(Java) / 前端 alerts 页(Vue3)。设计依据：DD-06+告警技术决议 / DD-05 / DD-01 §3.8 / 决议 D6 / DD-17 §6.7。

## 阶段 1：alert-engine 流式评估（Java）

### 1.1 规则模型与加载
- [ ] 1.1.1 [RED] 规则 CRUD 单测：t_alert_rule 字段（评估对象/指标源/条件/for/严重级/作用域）；启停 /alert/rules/{id}/enable
- [ ] 1.1.2 [GREEN] 实现规则 CRUD gRPC + MySQL t_alert_rule + Redis 规则缓存
- [ ] 1.1.3 [验收] 规则 CRUD + 启停正确；覆盖率 ≥80%

### 1.2 流式消费与滑动窗口评估（D6 方案 X）
- [ ] 1.2.1 [RED] 流式消费单测：订阅 Kafka 预聚合指标流，不轮询 ES（断言无 ES 查询调用）
- [ ] 1.2.2 [RED] 滑动窗口单测：窗口=规则 for 时长，窗口内持续命中才触发
- [ ] 1.2.3 [RED] 续算单测：评估状态外置 Redis，重启恢复不丢窗口
- [ ] 1.2.4 [GREEN] 实现 alert-engine 流式消费 + 滑动窗口 + 状态外置
- [ ] 1.2.5 [验收] 流式评估不压 ES，重启续算；覆盖率 ≥80%

### 1.3 静态阈值一期 + 检测延迟 SLA（D6 算法①）
- [ ] 1.3.1 [RED] P0 单测：错误率>10% for 1m -> ≤10s 检出（evaluate.interval=10s）
- [ ] 1.3.2 [RED] P1/P2/P3 单测：evaluate.interval=1m -> ≤1-2min 检出
- [ ] 1.3.3 [RED] 缺失类单测：心跳丢失 heartbeat_timeout=60s + for 判定
- [ ] 1.3.4 [GREEN] 实现静态阈值评估 + 分级 interval
- [ ] 1.3.5 [验收] P0≤10s/P1-3≤1-2min 检出，一期全覆盖 P0/P1；覆盖率 ≥80%

### 1.4 无状态 HPA 与分片
- [ ] 1.4.1 [RED] 分片单测：按规则/作用域分片并行评估，无重复评估
- [ ] 1.4.2 [GREEN] 实现分片 + HPA（按规则数/CPU）
- [ ] 1.4.3 [验收] 分片无重复，HPA 生效；覆盖率 ≥80%

## 阶段 2：降噪收敛与事件投递（Java）

### 2.1 静默与收敛
- [ ] 2.1.1 [RED] 静默单测：维护窗口规则不产生事件（alert:silence）
- [ ] 2.1.2 [RED] 收敛单测：同 rule+service+instance group_window=5m 仅 1 事件计数累加
- [ ] 2.1.3 [GREEN] 实现静默 + 收敛
- [ ] 2.1.4 [验收] 告警风暴收敛为 1 事件；覆盖率 ≥80%

### 2.2 关联压缩与升级
- [ ] 2.2.1 [RED] 关联压缩单测：同根因多告警经 M4 拓扑归因为 1 条根因告警+受影响列表
- [ ] 2.2.2 [RED] 升级单测：P0 未确认 ack_timeout=10min 自动升级二线
- [ ] 2.2.3 [GREEN] 实现关联压缩（依赖 add-topology）+ 升级
- [ ] 2.2.4 [验收] 同根因归因正确，升级触发；覆盖率 ≥80%

### 2.3 事件落库投递与状态闭环
- [ ] 2.3.1 [RED] 落库单测：事件落 npm_online_event(ES 365d) 字段完整
- [ ] 2.3.2 [RED] 投递单测：推 alert-events(Kafka) + HMAC 签名鉴权
- [ ] 2.3.3 [RED] Trace 入口单测：事件携带 trace_id
- [ ] 2.3.4 [RED] 状态闭环单测：firing->acknowledged->resolved，恢复反向更新 resolved_time
- [ ] 2.3.5 [GREEN] 实现落库 + 投递 + 状态闭环
- [ ] 2.3.6 [验收] 事件落库投递 + 状态闭环 + Trace 入口；覆盖率 ≥80%

### 2.4 平台自监控告警 Dogfooding
- [ ] 2.4.1 [RED] Dogfooding 单测：Collector 丢弃率/Kafka Lag/ES 写入错误/Agent 心跳丢失/health degraded -> 走同链路 P0 优先
- [ ] 2.4.2 [GREEN] 实现平台自监控告警规则（接 DD-05 黄金指标）
- [ ] 2.4.3 [验收] 平台异常先于业务方暴露；覆盖率 ≥80%

## 阶段 3：前端告警中心页（Vue3）

- [ ] 3.1 [RED] 规则管理页单测：CRUD/启停/静默配置
- [ ] 3.2 [GREEN] 实现 features/alerts 规则管理页
- [ ] 3.3 [RED] 事件列表单测：按级别/时间/服务/状态筛选 + 实时 WS 增量推送
- [ ] 3.4 [GREEN] 实现事件列表 + WS 实时推送（RealtimeService）
- [ ] 3.5 [RED] 告警详情单测：详情 + 关联 Trace 按钮 + 降噪原因标注 + Oncall 值班
- [ ] 3.6 [GREEN] 实现告警详情 + Trace 跳转
- [ ] 3.7 [RED] 降级单测：统一告警平台不可达->'投递通道暂不可用'，事件仍落库
- [ ] 3.8 [GREEN] 实现投递通道降级标注
- [ ] 3.9 [验收] 告警页全功能 + WS 实时 + Trace 跳转；覆盖率 ≥70%

## 阶段 4：跨组件联调验收（Gate，DD-17 §6.7）

- [ ] 4.1 [联调] M2 预聚合流 -> alert-engine 流式消费 -> 评估
- [ ] 4.2 [联调] P0 告警 -> ≤10s 检出 -> 降噪收敛 -> 落库投递
- [ ] 4.3 [联调] 告警 -> 关联 Trace 跳转链路（M3）
- [ ] 4.4 [联调] 同根因告警 -> M4 拓扑归因 -> 关联压缩
- [ ] 4.5 [联调] 平台自监控（Collector/Kafka/ES/Agent）-> Dogfooding 告警
- [ ] 4.6 [联调] 告警 -> M5 影响面跳转
- [ ] 4.7 [联调] 回填 add-collection 自适应采样：alert-engine 派生指标计算 sample_rate 下发（D4 闭环）
- [ ] 4.8 [验收] DD-17 §6.7：告警带Trace入口/动态基线(一期静态)/降噪/通知可达/状态闭环
- [ ] 4.9 [验收] 告警准确率 ≥95%、误报率 ≤5%（DD-17 §4.2）；覆盖率 ≥80%（后端）/≥70%（前端）

