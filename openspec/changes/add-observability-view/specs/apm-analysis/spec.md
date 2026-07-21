## ADDED Requirements

### Requirement: 接口/中间件 topN
MUST 支持 GET /services/{name}/top-apis 按 throughput/error/latency topN 排行（读 npm_online_performe），MUST 支持中间件 topN（读 npm_online_db，SQL 错误率/耗时 topN）。

#### Scenario: 接口吞吐 topN
- **WHEN** 用户查看某服务接口 topN 按吞吐量排序
- **THEN** 返回 top N 接口排行（读 npm_online_performe endpoint 聚合）

### Requirement: 热点方法分析（议题 B）
MUST 支持 GET /services/{name}/hotspots 按 operation_name/代码级 span 聚合 self_time（duration-Σchildren）+ 调用次数 TopN，MUST 仅对开启 code_level 采集的服务生效。

#### Scenario: 热点方法 TopN
- **WHEN** 用户对开启代码级采集的服务查询热点方法
- **THEN** 按 self_time+调用次数排序返回 TopN 方法，未开启则提示"未开启代码级采集"

### Requirement: 火焰图（议题 B）
MUST 支持 GET /services/{name}/flamegraph 以 trace_npm_jaeger 为源，按 parent_span_id 在 (service,1m bucket) 构建调用层级，MUST 选代表 Trace（桶内 Top-K 慢 Trace，K=100 默认可配）聚合为火焰图，MUST 不呈现每条 Trace（避免全量渲染），MUST 输出缓存（Redis 或 npm_online_hotspot）保障 P95<1s。

#### Scenario: 火焰图代表 Trace 聚合
- **WHEN** 用户查看某服务火焰图
- **THEN** 选桶内 Top-K 慢 Trace 聚合为层级火焰图（self_time），不渲染每条 Trace，P95<1s
