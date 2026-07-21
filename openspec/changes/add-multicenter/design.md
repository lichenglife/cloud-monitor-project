## Context

M1 已落地基础多中心管道（各中心 biz Collector + dmz 代理 + 上海 DC1 Kafka 汇聚，D7 真 128bit TraceID 携 datacenter_id）。M7 做查询时跨中心拼接/一致性/时钟处理高层能力。设计见 DD-08。

## Goals / Non-Goals

**Goals:**
- 跨中心 Trace 拼接（同 trace_id 多中心重组，因果/时间排序）
- 一致性校验 + 缺失标注（不伪造）
- 时钟偏差因果重排 + 偏差告警
- 多中心查询通道（上海 DC1 汇聚，前端标注来源）

**Non-Goals:**
- 不重建多中心管道（M1 已做）
- 不做异地灾备/苏州只读副本（DD-08 §7 待二期评估，F7 flag）
- 不做跨中心数据不出中心（决议 A 已定集中上海 DC1）

## Decisions

### DM1: 查询时跨中心拼接
各中心 Span 携 datacenter_id 汇聚至上海 DC1 ES。查询时按 128bit trace_id 聚合所有 Span，按 parent-child 因果关系 + start_time 排序重组完整链路。Kafka 分区按 trace_id hash 保序（M1 已做）保证同 trace 进同分区。

### DM2: 缺失诚实标注（PRD REQ-M7-W1）
某中心未上报或网络中断致部分 Span 不可达 -> 展示已汇聚部分 + 标注"跨中心数据缺失（时间窗）"，不伪造完整链路。复用 M3 的 CROSS_DC_MISSING 标注。

### DM3: 时钟偏差因果重排（PRD REQ-M7-W2）
跨中心时钟不同步致时序错乱 -> 基于 parent-child 因果关系重排（非纯时间排序），偏差超阈值（可配）告警（接 M5.1 alert-service）。

### DM4: 多中心查询通道
用户查询经上海 DC1 ES/MySQL（决议 A），跨中心数据已汇聚。前端按 datacenter_id 标注 Span/节点来源中心。

## Risks / Trade-offs

- **上海 DC1 单点查询风险**（F7）：DC1 整体故障->全局查询不可用。缓解--一期接受集中风险（决议 A），异地灾备/苏州只读副本待二期评估（DD-08 §4/§7）。
- **时钟同步依赖 NTP**：偏差超阈值影响因果重排。缓解--因果重排为主、时间为辅 + 偏差告警。
- **专线带宽**：跨中心传输。缓解--M1 已做本地预处理+压缩+限流（DD-08 §3）。
