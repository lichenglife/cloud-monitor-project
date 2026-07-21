## ADDED Requirements

### Requirement: Trace×Metric×Log 三域关联
MUST 以 TraceID 为关联主键联动链路、指标、日志：Trace×Metric 同 service+时间窗（[start-5min, end+1min] 可配）展示指标曲线（读 npm_online_metric），Trace×Log 按精确 traceId 匹配，MUST 支持联合下钻上下文（链->指->日志）。

#### Scenario: 三域关联同上下文展示
- **WHEN** 用户在链路详情查看三域关联
- **THEN** 同一上下文展示该 Trace 时间窗内指标曲线 + 按 traceId 关联日志，无需跨系统跳转

### Requirement: 关联超时降级
日志/指标接口超时或无数据时 MUST 标注"关联数据暂不可用"，MUST 不影响链路本身展示。

#### Scenario: 关联平台超时降级
- **WHEN** 日志平台接口超时
- **THEN** 标注"关联数据暂不可用"，链路详情仍正常展示，不抛错中断

### Requirement: 日志 PII 展示脱敏
日志展示侧 MUST 对手机号/身份证/银行卡等 PII 正则掩码（如 138****1234），原始不入本平台。

#### Scenario: 日志 PII 脱敏展示
- **WHEN** 关联日志含手机号
- **THEN** 展示侧掩码为 138****1234，原始手机号不显示
