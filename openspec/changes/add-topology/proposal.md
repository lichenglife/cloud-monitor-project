## Why

M2 已落库 trace_npm_jaeger，但调用关系仍不可见--拓扑缺失是公司现状痛点 P3（依赖关系靠人工梳理）。M4 构建全局链路拓扑：基于真实调用链自动发现依赖、CMDB 元数据补全、强弱依赖识别、纵向下钻，是故障定位与变更影响评估的基础（PRD US-02：拓扑自动发现覆盖率 ≥90%，含强弱依赖标注）。

评审决议 D2 界定全局拓扑产品需求（先系统后工程下钻）+ 后端重新聚合落库 ES + 前端 G6 WebGL 渲染。本 change 落地 M4，对应研发任务 T-M4-1~T-M4-5。

## What Changes

1. **依赖自动发现**：topology-service 消费 otel-spans，按 (source, target_type, target) × 多中心维度重新聚合为 npm_online_topology（1m 桶边），target_type ∈ {service, db, mq, cache}（DD-16 §1.2）。
2. **强弱依赖识别**：边出呼叫占比 <5% 节点总出调用 -> 弱依赖（虚线/淡显）；错误率 >1% 或 p95>SLO -> 异常高亮（红）；阈值可配（DD-03 §9.3）。
3. **CMDB 元数据补全**：节点/边补全机房/系统/负责人（CMDB 接口，先用 mock，Q4 对齐后切真实，PRD REQ-M4-U2）。
4. **全局拓扑 scope 聚合**：按 scope=global/system/center/room 物化，默认全局系统级（跨机房/跨中心/跨系统），点击系统下钻工程级（D2 需求）。
5. **快照合并**：topology.merge.interval 默认 60s（可配），合并快照写 t_topology_snapshot(MySQL) + Redis topology:cache(#3, 1m TTL)，最终一致性。
6. **跨中心边**：同 trace_id 跨中心调用生成跨中心边（特殊样式 + 中心标签，DD-08）。
7. **前端 G6 WebGL 渲染**：@antv/g6 v5 WebGL + 视口 LOD + 按 scope 聚类收起 + Web Worker 布局，万级节点；方案 B 聚类优先作降级兜底（D2）。
8. **纵向下钻**：节点->实例列表->容器->主机资源指标（DD-16 §5）。

## Capabilities

### New Capabilities
- `topology-calc`: 依赖自动发现 + 强弱依赖识别 + scope 聚合 + 跨中心边 + 快照合并（topology-service，SRP 独立）
- `topology-frontend`: 全局拓扑页 G6 WebGL 渲染 + LOD + 聚类 + Worker 布局 + 下钻交互 + 时间回放

### Modified Capabilities
<!-- 无，M4 为新能力；topology 数据源 npm_online_topology 由 M2 聚合管道派生，此处消费 -->

## Impact

- **依赖**：`add-storage-ingestion`（otel-spans Kafka + ES npm_online_topology 聚合索引就绪）+ `chore-foundation`（topology-service Go 脚手架 + 前端 + 契约）。外部：CMDB 元数据接口（Q4 待对齐，先用 mock）。
- **下游解锁**：M5 故障影响分析（读拓扑算爆炸半径）、M14 统一视图（拓扑缩略图）、M7 多中心（跨中心边）。
- **对应 PRD 模块**：M4（全局链路拓扑）。
- **设计依据**：DD-16（拓扑功能详细设计）、DD-03 §3/§9.3（模块二 + 合并频率）、DD-08（跨中心边）、DD-15 §7/§8.1（G6 WebGL+LOD+聚类+Worker）、DD-02（npm_online_topology/t_topology_snapshot/topology:cache）、DD-01 §3.3（/topology 接口）、决议 D2、DD-17 §6.2（M2 拓扑验收）。
