## ADDED Requirements

### Requirement: 瀑布图组装与下钻
trace-query-service MUST 按 parent_span_id 组装 span 树返回瀑布图数据（每 Span 含 service/operation/duration/status/耗时占比/颜色编码），MUST 支持点击 Span 下钻方法级/SQL 级明细与代码堆栈（异常自动记录堆栈）。

#### Scenario: Span 下钻方法/SQL
- **WHEN** 用户点击瀑布图中某 SQL Span
- **THEN** 展示 SQL 语句 + 执行计划 + 耗时；点击方法 Span 展示方法逐层调用耗时与代码栈

### Requirement: 链路完整性标注
MUST 检查 parent-child 时间差异常/Span 缺失/跨中心 datacenter_id 分布，标注 COMPLETE/PARTIAL_BROKEN/CROSS_DC_MISSING，MUST 不静默丢弃断裂链路。

#### Scenario: 跨中心链路断裂标注
- **WHEN** 某 Trace 跨中心调用但部分 Span 缺失
- **THEN** 标注"链路断裂"缺口位置，不静默丢弃，前端显式提示

### Requirement: 超长 Trace 截断
采集侧 MUST 硬上限 50000 spans/Trace（超限丢弃并计数），展示侧 MUST 软上限 10000 spans（超出保留错误/慢 span + 采样子集，标记 truncated=true）。

#### Scenario: 超长 Trace 展示截断
- **WHEN** 某 Trace 含 8 万 spans
- **THEN** 采集侧硬截断至 5 万并计数，展示侧展示 1 万（含错误/慢 span + 采样子集），标记 truncated=true
