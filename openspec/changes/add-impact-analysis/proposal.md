## Why

M4 已构建全局拓扑，但故障发生时仍需人工沿拓扑逐跳判断影响范围--排障低效（痛点 P4）。M5 沿拓扑自动计算上下游影响面（直接/间接）与爆炸半径，输出可量化影响范围，支撑故障应急与变更影响评估（PRD US-02/REQ-M5-E1）。

本 change 落地 M5，对应研发任务 T-M5-1。

## What Changes

1. **上下游影响面计算**：某服务被标记故障/告警时，沿 npm_online_topology 拓扑自动计算上游（依赖该服务的服务）与下游（该服务依赖的服务）影响面，支持 depth=1/2/3（直接/二级/三级依赖，DD-01 §3.4 /topology/upstream-downstream）。
2. **爆炸半径输出**：聚合影响面为爆炸半径（受影响服务数/实例数/层级深度），区分强依赖（阻断）与弱依赖（可降级）影响。
3. **拓扑盲区提示**：影响面因拓扑不完整有盲区时，提示"影响分析可能存在遗漏"，不绝对结论（PRD REQ-M5-W1）。
4. **告警联动入口**：告警事件（M5.1）携带 Trace 入口时，可一键跳转影响面视图（与 M5.1 联调）。

## Capabilities

### New Capabilities
- `impact-analysis`: 上下游影响面计算 + 爆炸半径 + 拓扑盲区提示（topology-service 扩展，复用 M4 拓扑）

### Modified Capabilities
- `topology-calc`: M4 拓扑服务扩展提供 upstream-downstream 查询与影响面计算 RPC

## Impact

- **依赖**：`add-topology`（npm_online_topology 边 + t_topology_snapshot 就绪）+ `chore-foundation`。与 `add-alert-center`(M5.1) 联调（告警->影响面跳转）。
- **下游解锁**：M14 统一视图（影响面卡片）、告警中心（爆炸半径标注）。
- **对应 PRD 模块**：M5（故障影响分析）。
- **设计依据**：DD-03 §3（模块二拓扑）、DD-16 §3-4（上下游拓扑）、DD-01 §3.4（/topology/upstream-downstream）、PRD REQ-M5-E1/W1、DD-17 §6.2。
