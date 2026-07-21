## ADDED Requirements

### Requirement: 上下游影响面计算
topology-service MUST 沿 npm_online_topology 拓扑计算某故障服务的上下游影响面（depth=1/2/3，直接/二级/三级依赖），MUST 区分强依赖（阻断）与弱依赖（可降级）影响，MUST 输出爆炸半径（受影响服务数/实例数/层级深度/强弱依赖影响数）。

#### Scenario: 故障影响面自动计算
- **WHEN** 服务 A 被标记故障
- **THEN** 沿拓扑计算上游（依赖 A 的服务）与下游（A 依赖的服务），depth 可配，输出爆炸半径

#### Scenario: 强弱依赖区分影响
- **WHEN** A 强依赖 B（B 故障致 A 不可用）、A 弱依赖 C（可降级）
- **THEN** 影响面标注 B 阻断影响、C 可降级影响，爆炸半径分别统计

### Requirement: 拓扑盲区诚实标注
影响面因拓扑不完整有盲区时 MUST 提示"影响分析可能存在遗漏"，MUST 不绝对结论。

#### Scenario: 拓扑不全提示遗漏
- **WHEN** 影响面计算遇到未知依赖或缺失边
- **THEN** 标注"影响分析可能存在遗漏"，不给出绝对完整结论（PRD REQ-M5-W1）
