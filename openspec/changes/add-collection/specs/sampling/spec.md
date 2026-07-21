## ADDED Requirements

### Requirement: 头部采样（Agent 侧）
Agent MUST 在请求入口即时做头部采样决策，试点期默认 100% 全量采样（D3），普通流量按 sample_rate 比例，决策依据低基数键（http.status_code/db.operation），sample_rate 由 t_agent_config 热配置下发。

#### Scenario: 试点全量采样
- **WHEN** 试点期某服务 sample_rate 配置为 100%
- **THEN** 该服务所有请求 Span 均上报，落库率 100%，验证全链路

### Requirement: 尾部采样（Collector 侧）
Collector MUST 做尾部采样，错误 Span（status.code=1/error=true）100% 采，慢请求（Trace 总耗时 >2s）100% 采，超长 Trace（>50000 spans）硬截断并计数仍保留；缓冲 window=10s / in-flight=20000（D5），内存占 100-300MB/实例，超限丢最旧在途 Trace 并告警。

#### Scenario: 错误请求必采
- **WHEN** 某 Trace 含错误 Span（status.code=1）
- **THEN** 该 Trace 100% 被采样上报，即使头部采样未命中

#### Scenario: 尾部缓冲超限保护
- **WHEN** 在途 Trace 数超 in-flight=20000
- **THEN** Collector 丢弃最旧在途 Trace 的缓冲并计数告警（DD-06），避免 OOM，不阻塞业务

### Requirement: 自适应采样（D4）
自适应采样 MUST 以规则触发调档：错误率↑->采样↑、流量↑->采样↓、耗时百分位异常->↑慢请求采样，步进 ±10%，下调有下限（保错误/慢必采 + 最小可观测样本）。规则触发逻辑 MUST 在本变更实现（读派生指标，经 config-center 下发 agent:config）；派生指标计算的完整闭环依赖 add-alert-center(M5.1) 的 alert-engine，本变更先用静态配置兜底（D4 方案 C），M5.1 落地后切规则触发。

#### Scenario: 流量洪峰自动降采样
- **WHEN** 入口 QPS 超阈值持续
- **THEN** 自适应规则将 sample_rate 下调（步进 ±10%，有下限），控成本同时保错误/慢必采

### Requirement: 采样热配置 ≤30s 生效
采样率/最大 TPS/采样模式变更 MUST 经 config-center 下发至 Redis agent:config + config:version，Agent Watch 拉取 ≤30s 生效，无需重启业务进程。

#### Scenario: 采样率热更新无重启
- **WHEN** 管理端修改某集群 sample_rate 从 100% 调为 50%
- **THEN** Agent ≤30s 生效新采样率，业务进程无重启，心跳回报新配置版本
