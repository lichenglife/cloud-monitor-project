## Why

M1 已做基础多中心管道（各中心 biz Collector + dmz 代理 + 上海 DC1 汇聚），但跨中心 Trace 拼接、一致性校验、时钟偏差处理尚未落地--跨中心调用链可能断裂或时序错乱（PRD REQ-M7-W1/W2）。M7 构建跨中心链路汇聚的高层能力：同 TraceID 跨中心拼接、因果重排、缺失标注，支撑金融多活链路核查（PRD US-07）。

本 change 落地 M7，对应研发任务 T-M7-1~T-M7-3。注意：M1 已落地基础管道与 TraceID 真 128bit（D7），本变更做查询时跨中心拼接/一致性/时钟处理。

## What Changes

1. **跨中心 Trace 拼接**：同一 128bit trace_id 出现多中心时，查询时跨中心拼接为完整调用链，按因果/时间排序（PRD REQ-M7-E1）。各中心 Span 携带 datacenter_id，汇聚后按 trace_id 重组。
2. **一致性校验**：跨中心数据完整性校验，某中心未上报或网络中断致部分数据暂不可达时，展示已汇聚部分并标注"跨中心数据缺失（时间窗）"，不伪造完整链路（PRD REQ-M7-W1）。
3. **时钟偏差因果重排**：跨中心时钟不同步致时序错乱时，基于因果关系（parent-child）重排，偏差超阈值告警（PRD REQ-M7-W2）。
4. **biz/dm 隔离通道**：某中心隔离（biz/dm 不通）时，该中心 Agent 仅经 dmz 代理上报（PRD REQ-M7-S1，M1 已做管道，本变更做查询通道标注）。
5. **多中心查询通道**：用户经上海 DC1 ES/MySQL 查询，跨中心数据已汇聚，前端按 datacenter_id 标注来源。

## Capabilities

### New Capabilities
- `cross-center-splicing`: 跨中心 Trace 拼接（同 trace_id 多中心重组 + 因果/时间排序）
- `consistency-check`: 跨中心一致性校验 + 缺失标注 + 时钟偏差因果重排

### Modified Capabilities
- `trace-waterfall`: M3 瀑布图支持跨中心 Span 拼接展示 + 中心标签 + 缺失标注（CROSS_DC_MISSING）
- `topology-calc`: M4 拓扑跨中心边已生成（M4 已做），本变更高层一致性校验

## Impact

- **依赖**：`add-collection`（M1 多中心管道 + TraceID 128bit D7 + datacenter_id 维度）+ `add-storage-ingestion`（汇聚上海 DC1 ES）+ `add-trace-query`（瀑布图）+ `add-topology`（跨中心边）。外部：专线带宽（决议 A）。
- **下游解锁**：M14 统一视图（多中心健康）。
- **对应 PRD 模块**：M7（多中心链路汇聚）。
- **设计依据**：DD-08（多中心数据方案）、DD-14 §3（TraceID 跨中心拼接）、DD-03 §9.4（跨中心缺失标注）、DD-16 §3（跨中心边）、决议 A/D7/D8、DD-17 §6.2/§6.7。
