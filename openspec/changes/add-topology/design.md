## Context

M2 已落库 otel-spans Kafka + ES。M4 的 topology-service 消费 otel-spans 重新聚合为 npm_online_topology 边，构建全局拓扑。D2 决议：后端重新聚合落库 ES，前端 G6 WebGL 渲染。设计见 DD-16。

数据流（DD-16 §2）：
```
otel-spans(Kafka) -> topology-calc 消费 -> 按 (source,target_type,target)×多中心 聚合 1m 边
   -> npm_online_topology(ES) -> 合并快照(60s) -> t_topology_snapshot(MySQL) + topology:cache(Redis)
   -> 前端 /topology 查询(scope=global/system/center/room)
```

## Goals / Non-Goals

**Goals:**
- 依赖自动发现（不依赖人工录入，PRD REQ-M4-U1）
- 强弱依赖识别 + 异常高亮
- scope 聚合（global 默认系统级 -> 点击下钻工程级，D2）
- 跨中心边 + 快照合并 60s
- 前端 G6 WebGL 万级节点（LOD + 聚类 + Worker 布局）
- 纵向下钻至实例/容器/主机

**Non-Goals:**
- 不实现爆炸半径计算（M5，复用拓扑）
- 不实现 CMDB 真实接口对接（Q4 待对齐，mock 占位）
- 不实现时间回放的历史快照存储优化（DD-16 §6 待评审，本变更用 1m 快照 180d 基线）

## Decisions

### DM1: topology-calc 独立服务 SRP（DD-01 §5）
topology-service 只算拓扑，不收数不查询。消费 otel-spans -> 聚合边 -> 写 ES/npm_online_topology + 快照。计算贴近存储（ES Ingest/Transform 或 topology-calc 内聚合）。

### DM2: scope 聚合物化（D2）
聚合结果同时按 scope=global/system/center/room 物化：global=跨系统调用（系统级，忽略实例/工程）；点击系统 -> 下钻 engineering 级（按 engineering_id）。支撑"先系统后工程"默认视图。

### DM3: 强弱依赖阈值（DD-03 §9.3）
弱依赖：边出呼叫占比 <5% 节点总出调用（虚线/淡显）；异常：错误率 >1% 或 p95>SLO（红高亮）。阈值经 config-center 可配。

### DM4: 快照合并最终一致性
topology.merge.interval 默认 60s（可配），合并写 t_topology_snapshot + Redis topology:cache(#3, 1m TTL)。拓扑允许秒级延迟（追踪特殊原则）。

### DM5: 前端 G6 WebGL + 降级（D2）
方案 A：G6 v5 WebGL + 视口 LOD + 按 scope 聚类收起 + Web Worker 布局；方案 B（默认聚类收起、仅下钻展开）作降级兜底（超阈值自动切聚类）。前端仅检索轻量聚合，不重算。

## Risks / Trade-offs

- **万级节点渲染性能**（D2 PoC 需验证）：缓解--WebGL + LOD + 聚类 + Worker 布局，B 降级兜底；PoC 2 周内出帧率结论。
- **CMDB 数据不全**（R8）：缓解--二次评估 + mock 占位 + 链路自动发现为主、CMDB 补元数据；未知依赖标注"未知依赖"提示人工确认（PRD REQ-M4-W1）。
- **拓扑计算延迟**：60s 合并窗口。缓解--最终一致性可接受，实时模式展示近期调用（PRD REQ-M4-S1）。
- **跨中心同名服务合并**：缓解--按 data_center 区分或合并（可配，DD-16 §6 待评审）。
