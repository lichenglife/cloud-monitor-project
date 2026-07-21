## ADDED Requirements

### Requirement: 依赖自动发现与边聚合
topology-service MUST 消费 otel-spans 按 (source, target_type, target) × 多中心维度重新聚合为 1m 桶边写入 npm_online_topology，target_type MUST ∈ {service, db, mq, cache}，MUST 不依赖人工录入（基于真实调用链自动发现）。

#### Scenario: 调用关系自动发现
- **WHEN** 服务 A 调用服务 B 与 MySQL
- **THEN** topology-calc 自动生成 A->B(service 边)与 A->MySQL(db 边)，无需人工录入

### Requirement: 强弱依赖与异常高亮
MUST 识别强弱依赖（边出呼叫占比 <5% 节点总出调用 -> 弱依赖虚线淡显）与异常高亮（错误率 >1% 或 p95>SLO -> 红高亮），阈值 MUST 经 config-center 可配。

#### Scenario: 弱依赖识别
- **WHEN** 服务 A 对服务 C 的调用占比 <5% 总出调用
- **THEN** A->C 边标注弱依赖（虚线/淡显），C 故障时 A 可降级

#### Scenario: 未知依赖标注
- **WHEN** 依赖数据缺失无法判定强弱
- **THEN** 标注"未知依赖"并提示人工确认，不静默丢弃（PRD REQ-M4-W1）

### Requirement: scope 聚合与全局下钻（D2）
MUST 按 scope=global/system/center/room 物化聚合结果，默认全局系统级（跨机房/跨中心/跨系统调用），点击系统节点 MUST 下钻至该系统工程级（按 engineering_id 重新聚合）。

#### Scenario: 全局拓扑默认系统级
- **WHEN** 用户点击"全局拓扑"菜单
- **THEN** 默认展示跨机房/跨中心/跨系统的调用关系（系统级，忽略实例/工程细节）

#### Scenario: 点击系统下钻工程级
- **WHEN** 用户点击某系统节点
- **THEN** 下钻展示该系统下工程与其他工程的调用关系（engineering 级）

### Requirement: 快照合并与跨中心边
MUST 按 topology.merge.interval（默认 60s 可配）合并快照写 t_topology_snapshot + Redis topology:cache（1m TTL），同 trace_id 跨中心调用 MUST 生成跨中心边（特殊样式 + 中心标签）。

#### Scenario: 跨中心边生成
- **WHEN** 上海服务调用苏州服务（同 trace_id 跨中心）
- **THEN** 生成跨中心边，拓扑图以特殊样式 + 中心标签呈现
