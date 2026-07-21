## ADDED Requirements

### Requirement: Agent 注册与心跳
Agent 启动 MUST 向 agent-registry 注册（签发 t_agent_registration.token 鉴权），周期心跳 PushAgentHeartbeat，last_heartbeat 超 60s 标离线，台账展示在线率/版本/资源/丢包。

#### Scenario: Agent 离线识别
- **WHEN** 某 Agent 超过 60s 未上报心跳
- **THEN** 台账标记该实例离线并触发 P1 告警（DD-06），不阻断其他 Agent

### Requirement: 配置下发与灰度
config-center MUST 将 t_agent_config 配置（采样率/最大TPS/开关/灰度分组）发布至 Redis agent:config:{clusterId}，支持全量与灰度分组下发，Agent Watch ≤30s 生效；配置变更 MUST 记录 t_agent_config_history。

#### Scenario: 灰度配置下发
- **WHEN** 管理端按灰度分组下发新采样率
- **THEN** 仅命中灰度分组的 Agent 应用新配置，未命中保持原配置，变更记录入历史表

### Requirement: 批量升级与回滚
/agents/upgrade MUST 支持按 cluster_id/engineering_id 推送目标版本，金丝雀（1%->10%->100%）分批验证，升级异常（心跳丢失率突增/资源超红线/业务报错）MUST 自动回滚至稳定版，支持一键回滚，升级期间不丢数据。

#### Scenario: 灰度升级异常自动回滚
- **WHEN** 金丝雀阶段某批次 Agent 心跳丢失率突增
- **THEN** 自动回滚该批次至稳定版，上报恢复，升级期间数据经缓冲不丢

### Requirement: 兼容性矩阵
MUST 维护语言/框架兼容矩阵（Java 8/11/17 + SpringBoot 1.x/2.x/3.x + Dubbo/gRPC；Go 1.18+ + gin/grpc-go），新版本上线前 MUST 入矩阵并通过兼容性测试（CI 卡点），明确拒绝 EOL 版本（JDK 6/7）。

#### Scenario: 新 JDK 版本入矩阵
- **WHEN** 需支持 JDK 21
- **THEN** 入兼容矩阵前须通过兼容性测试（CI 卡点），未通过则不支持，治理前置

### Requirement: 资源红线与自动熔断（D11）
Agent 资源 MUST 遵循统一红线（CPU<1%/内存<128M/线程≤业务5%/丢包<0.1%），重负载应用经 t_agent_config 白名单调高；业务 CPU>95% 持续 N 秒 MUST 自动熔断/降采样优先保业务，内存逼近红线 MUST 丢缓冲降采样，Collector 不可达 MUST 本地磁盘缓冲（有界）+ 心跳异常标记。

#### Scenario: 业务 CPU 打满自动熔断
- **WHEN** 宿主 CPU >95% 持续 N 秒
- **THEN** Agent 自动熔断暂停采集或降采样，优先保业务，触发告警（DD-06），业务不受影响

#### Scenario: 重负载应用白名单
- **WHEN** 批处理高吞吐应用统一红线内存<128M 不足
- **THEN** 经 t_agent_config 白名单调高该应用上限，不影响轻量应用红线
