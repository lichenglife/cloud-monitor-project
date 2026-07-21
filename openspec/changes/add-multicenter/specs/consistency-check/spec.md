## ADDED Requirements

### Requirement: 跨中心缺失诚实标注
某中心未上报或网络中断致部分数据暂不可达时 MUST 展示已汇聚部分并标注"跨中心数据缺失（时间窗）"，MUST 不伪造完整链路。

#### Scenario: 跨中心数据缺失标注
- **WHEN** 某 Trace 跨中心但苏州中心部分 Span 未上报
- **THEN** 展示已汇聚的上海部分 + 标注"跨中心数据缺失（时间窗）"，不伪造缺失部分

### Requirement: 时钟偏差因果重排与告警
跨中心时钟不同步致时序错乱时 MUST 基于因果关系（parent-child）重排，偏差超阈值 MUST 告警（接 alert-service）。

#### Scenario: 时钟偏差因果重排
- **WHEN** 跨中心 Span 时序因时钟不同步错乱
- **THEN** 基于 parent-child 因果关系重排（非纯时间），偏差超阈值触发告警

### Requirement: 跨中心链路拼接正确率
跨中心 Trace MUST 100% 拼接为完整链（一期规模，DD-17 §4.2），多中心 testkit 调用校验 trace 完整性。

#### Scenario: 跨中心拼接正确率验收
- **WHEN** testkit 多实例（上海/苏州）跨中心调用
- **THEN** 同 trace_id 100% 拼接为完整链，校验 trace 完整性
