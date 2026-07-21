## Why

应用链路层（Trace）监控是平台核心缺口（需求规划 P1：现状无任何应用链路层监控，调用链完全不可见）。M1 是生命周期第一步，为 M2~M17 全部模块提供数据源头--没有采集与 Agent 治理，后续存储/检索/拓扑/关联均无数据可用。

技术路线已决议（议题 E）：Collector 基于 OTel 定制开发，统一转换 SkyWalking(Java)/OTel(Go) 数据为 OTel 协议；Java=SkyWalking Agent 自动埋点、Go=OTel SDK 手动埋点（PRD REQ-M1-U3）；K8S initContainer 注入 + VM 手动注入；biz 直连 / dmz 代理转发，数据集中上海 DC1 Kafka（决议 A）。采样双模（最高 TPS / 比率）热配置 ≤30s 生效（决议 D3 试点 100% / D5 尾部 window=10s in-flight=20000 / D4 自适应 ±10%）。

本 change 落地 M1 采集层 + Agent 管理，对应研发任务 T-M1-1~T-M1-12（52 条任务中关键路径起点）。

## What Changes

1. **OTel 定制 Collector**：基于 OTel 0.90+ 定制，Receiver(OTLP gRPC:4317/HTTP:4318) -> Processor(协议转换 SkyWalking v3->OTel / Go OTel 透传 + PII 脱敏 + 多中心维度补全 + 幂等去重) -> Exporter(Kafka 上海 DC1)。无状态可 HPA，入口背压 1500 QPS 预留，ES 不可用降级本地磁盘缓冲。
2. **Java/Go Agent 接入**：Java=SkyWalking Agent（字节码增强，ClassLoader 隔离，自动埋点 Web/RPC/SQL/Redis/MQ/运行环境）；Go=OTel SDK 手动埋点。K8S initContainer 注入 `java -jar -agent:xxx`，VM 手动注入。跨 MQ 上下文透传，日志 MDC 注入 Agent TraceID（决议 C）。
3. **采样双模 + 自适应**：头部采样（Agent 侧，试点 100% D3）+ 尾部采样（Collector 侧，错误/慢>2s 必采，window=10s/in-flight=20000 D5）+ 自适应规则触发调档 ±10%（D4）。热配置 ≤30s 生效无需重启。
4. **全栈指标采集**：应用层（JVM GC/线程/死锁、接口 QPS/耗时/错误码/状态码）、资源层（CPU/内存/磁盘/网络）、中间件（MySQL/Oracle/Redis/Kafka/RabbitMQ 客户端吞吐/耗时/错误率）、运行环境（OS/JDK/框架/Tomcat）。
5. **Agent 管理**：配置下发/版本/心跳/灰度（config-center + agent-registry），Agent 台账（集群/机房/zone/IP/实例名/健康/版本/注册时间），兼容性矩阵，批量灰度升级与回滚，资源红线（D11 CPU<1%/内存<128M + 重负载白名单），自动熔断降级。
6. **多中心管道**：各中心 biz Collector + dmz 代理转发，汇聚上海 DC1 Kafka（决议 A/D8 RF≥3 跨 AZ）；TraceID 真 128bit 跨中心拼接（D7）。
7. **代码级链路开关**：方法级明细/堆栈采集开关可配。

## Capabilities

### New Capabilities
- `collection`: OTel 定制 Collector 数据采集与协议转换管道（Receiver/Processor/Exporter），OTLP 接收、SkyWalking/OTel->OTel 统一、PII 脱敏、幂等去重、背压降级、多中心汇聚
- `agent-instrumentation`: Java(SkyWalking)/Go(OTel) Agent 注入与埋点（K8S initContainer/VM 手动、自动/手动埋点、跨 MQ 透传、日志 MDC 注入 TraceID、全栈指标采集、代码级开关）
- `sampling`: 采样双模（头部+尾部）+ 自适应采样（D3 试点 100%、D5 尾部 window=10s/in-flight=20000、D4 ±10% 规则调档、错误/慢必采、热配置 ≤30s）
- `agent-governance`: Agent 治理（配置下发/版本/心跳/灰度/台账/兼容矩阵/批量升级回滚/资源红线 D11/自动熔断）

### Modified Capabilities
<!-- 无，M1 为新能力；testkit 铺底组件的 db/mq 场景填充在 chore-foundation test-kit capability 的后续迭代，此处不修改 -->

## Impact

- **依赖**：`chore-foundation`（脚手架+契约+testkit 骨架）为前置；本 change 引用其 api-contracts（OTLP/Agent 管理 API）、app-scaffold、test-kit。
- **下游解锁**：M2 存储/检索、M4 拓扑（消费 otel-spans）、M6 日志关联（MDC TraceID）、M7 多中心、M14 APM（全栈指标）、M16 Agent 台账。
- **外部依赖**：议题 A 部署架构（biz Collector + dmz 代理，✅已定）；CMDB 权限接口（M4/M15 才需，本 change 不阻塞，Agent 注册暂用本地令牌）。
- **对应 PRD 模块**：M1（统一数据采集层）。本变更同时落地 Agent 台账的后端数据源（agent-registry 台账查询 gRPC，T-M1-8 衍生）+ 前端 Agent 管理 UI（/agents 页，含配置下发/升级/兼容矩阵）；add-ops-console(M16) 不再重复 Agent 台账，仅做用户/项目/集群/系统字典/审计。
- **设计依据**：DD-07（采集与采样）、DD-13（Agent 治理）、DD-08（多中心）、DD-03.5（埋点规范）、DD-14（TraceID/脱敏/采样自适应）、DD-02（Kafka Topic/Redis Key/MySQL t_agent_*）、DD-01（/agents/* /collectors/* /ingest/* 接口、OTLP 协议、限流 1500 QPS）、系统设计说明书 §3.1（数据采集模块）、决议 A/C/D3/D4/D5/D7/D8/D11。
