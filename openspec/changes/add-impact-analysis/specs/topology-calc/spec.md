## MODIFIED Requirements

### Requirement: 拓扑服务扩展上下游查询
topology-calc（M4）MUST 扩展提供 GET /topology/upstream-downstream?service={svc}&depth={1|2|3} 查询与影响面计算，复用 npm_online_topology 边数据，结果缓存 Redis topology:upstream:{service}:{depth}（300s TTL）。

#### Scenario: 上下游查询 depth 分级
- **WHEN** 查询 service=A depth=2
- **THEN** 返回 A 的二级上下游依赖，结果缓存命中时直接读 Redis
