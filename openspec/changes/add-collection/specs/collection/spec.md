## ADDED Requirements

### Requirement: OTLP 接收与协议转换
Collector MUST 通过 OTLP 协议（gRPC :4317 / HTTP :4318）接收 Agent 上报数据，并 MUST 将 SkyWalking v3 格式与 Go OTel 格式统一转换为 OTel 协议统一格式，字段映射遵循 DD-14 §5。

#### Scenario: Java SkyWalking 数据统一为 OTel
- **WHEN** Java 应用经 SkyWalking Agent 上报 Span 至 Collector
- **THEN** Collector 自定义 processor 将 SkyWalking v3 格式转换为 OTel Span，关键字段（trace_id/span_id/operation/tags/status）无损，导出至 Kafka otel-spans

#### Scenario: Go OTel 数据透传
- **WHEN** Go 应用经 OTel SDK 手动埋点上报数据
- **THEN** 数据已是 OTel 格式，Collector 直接透传至 Kafka，无需协议转换

### Requirement: 幂等去重与背压降级
Collector MUST 按 trace_id+span_id 幂等去重（Redis trace:dedup TTL 10m），入口限流 1500 QPS 预留超阈背压，ES/Kafka 不可用时 MUST 降级本地磁盘缓冲（有界队列）不反向压垮 Agent。

#### Scenario: 重复 span 去重
- **WHEN** 同一 trace_id+span_id 被重复上报（Agent 重传）
- **THEN** Collector 经 Redis 去重仅落库一次，Kafka 与 ES 无重复数据

#### Scenario: 存储不可用降级不压垮业务
- **WHEN** ES 或 Kafka 不可用
- **THEN** Collector 降级本地磁盘缓冲（有界队列），不反向压垮 Agent，业务线程零阻塞；恢复后重传不丢

### Requirement: 多中心管道与数据集中
各中心 biz 区 MUST 部署 Collector 集群，dmz 区 MUST 部署代理转发至本中心 biz Collector，所有 Collector 数据 MUST 汇聚至上海 DC1 Kafka（RF≥3 跨 AZ），跨中心仅传已采样/已处理数据。

#### Scenario: dmz 区 Agent 经代理上报
- **WHEN** dmz 区应用 Agent 上报数据
- **THEN** 经本中心 dmz 代理（HAProxy/Nginx）转发至本中心 biz Collector，再汇聚至上海 DC1 Kafka，符合决议 A

#### Scenario: 跨中心 Trace 拼接
- **WHEN** 同一请求跨中心调用（上海服务->苏州服务）
- **THEN** 同一 128bit trace_id 的 Span 由上海 DC1 汇聚后按 trace_id 重组完整链路，Kafka 分区按 trace_id hash 保序

### Requirement: PII 源头脱敏与多中心维度补全
Collector MUST 在转换阶段对 PII（db.statement/http.url/自定义 Attribute 中手机号/身份证/密码/银行卡）正则掩码，并 MUST 补全多中心维度（data_center/network_area/room/cluster）。

#### Scenario: SQL 参数脱敏
- **WHEN** Span 含 db.statement "SELECT * FROM user WHERE phone='13812345678'"
- **THEN** Collector 上报前掩码为 "138****5678"，原始手机号不落 ES
