## ADDED Requirements

### Requirement: Java SkyWalking Agent 自动埋点
Java 服务 MUST 使用 SkyWalking Agent 字节码增强自动埋点（Web/RPC/SQL/Redis/MQ/运行环境），Agent 依赖 MUST 经 ClassLoader 隔离（AgentClassLoader）避免与业务类冲突，插件热插拔，冲突插件默认禁用并告警。

#### Scenario: SpringBoot 应用无侵入采集
- **WHEN** SpringBoot 应用经 K8S initContainer 注入 `-javaagent:skywalking-agent.jar` 启动
- **THEN** Agent 自动拦截 Web/RPC/SQL/Redis/MQ 调用生成 Span，业务源码零修改，依赖与业务 AppClassLoader 隔离无类冲突

### Requirement: Go OTel SDK 手动埋点
Go 服务 MUST 使用 OTel SDK 手动埋点，原生 OTLP 导出，遵循 DD-03.5 语义约定。

#### Scenario: Go gin 应用手动埋点
- **WHEN** Go gin 应用引入 OTel SDK 并在 HTTP handler 与下游调用处埋点
- **THEN** 产生 OTel Span 经 OTLP 导出至 Collector，span 命名与 tag 遵循 DD-03.5

### Requirement: Agent 注入与上下文透传
容器应用 MUST 经 K8S initContainer 注入 Agent 启动命令（含 Collector ip+port、service_name、instance_name、sample_rate），VM 应用 MUST 由运维手动注入；跨线程/跨 MQ（Kafka/RabbitMQ）MUST 透传 Trace 上下文保证异步链路连续。

#### Scenario: 跨 MQ 链路连续
- **WHEN** 应用经 Kafka 收发消息
- **THEN** Agent 自动注入/提取 Trace 上下文，生产者与消费者 Span 共享 trace_id，异步链路连续可下钻

### Requirement: 日志 MDC 注入 Agent TraceID
Agent MUST 在应用打印日志时无感注入 TraceID 到 MDC（`MDC.put("traceId", currentTraceId)`），日志框架 Pattern `%X{traceId}` 输出，使日志链路 ID = 链路 trace_id（决议 C）。

#### Scenario: 日志与链路 trace_id 一致
- **WHEN** 应用执行 logger.info("订单完成") 时存在 TraceContext
- **THEN** 日志文本含当前 traceId，与该请求链路 trace_id 一致，可在链路详情按 traceId 查关联日志

### Requirement: 全栈指标采集
Agent MUST 采集全栈指标：应用层（JVM GC/线程/死锁、接口 QPS/耗时/错误码/状态码）、资源层（CPU/内存/磁盘/网络）、中间件（MySQL/Oracle/Redis/Kafka/RabbitMQ 客户端吞吐/耗时/错误率）、运行环境（OS/JDK/框架/Tomcat）。

#### Scenario: 中间件客户端指标采集
- **WHEN** 应用经 JDBC 访问 MySQL
- **THEN** Agent 采集 MySQL 客户端吞吐量/耗时/错误率，经 OTLP 上报，落入 npm_online_db 聚合索引

### Requirement: 代码级链路开关
管理端 MUST 可下发"代码级调用链采集开关"，Agent 据此开启/关闭方法级明细与堆栈采集。

#### Scenario: 代码级开关热切换
- **WHEN** 管理端关闭某服务代码级采集开关
- **THEN** Agent ≤30s 停止方法级明细与堆栈采集，降低开销，业务无感知
