# add-collection（M1）实现任务清单

> **TDD 强制**（CLAUDE.md）：每个功能点先写失败测试（RED）-> 最小实现（GREEN）-> 重构（REFACTOR），覆盖率 ≥80%，无测试代码视为未完成。
> **开发顺序**：按依赖链从底层到上层，每阶段产出可验收功能点；阶段末跨组件联调验收为 Gate。
> **涉及组件**（chore-foundation 已初始化脚手架）：otel-collector(Go) / config-center(Go) / agent-registry(Go) / Java Agent(SkyWalking) / Go Agent(OTel) / 前端 ingest+agents 页(Vue3)。
> **设计依据**：DD-07 采集采样 / DD-13 Agent 治理 / DD-08 多中心 / DD-03.5 埋点 / DD-14 TraceID+脱敏 / DD-02 数据模型 / 决议 A/C/D3/D4/D5/D7/D8/D11。

## 阶段 1：TraceID 生成器（D7，全局基础，最先做）

> 所有采集依赖 TraceID，先落地 128bit 生成器，Java/Go 双实现 + 一致性测试。

- [ ] 1.1 [RED] TraceID 128bit 生成器单测：验证格式 16 字节/32 hex、低 64bit 字段位（时间低 41+中心 3+节点 10+序列 10）、高 64bit（续位 23 恒 0+随机 41）、同毫秒防碰撞、时间有序
- [ ] 1.2 [GREEN] Go TraceID 生成器实现（github.com/lichenglife/cloud-monitor/pkg/traceid，crypto/rand + 时间戳）
- [ ] 1.3 [GREEN] Java TraceID 生成器实现（com.lichenglife.npm.trace.common.traceid，SecureRandom）
- [ ] 1.4 [REFACTOR] 抽取位运算常量与测试夹具，双语言共享测试用例集
- [ ] 1.5 [验收] Go/Java 生成器各 10 万次无碰撞；跨语言同参数生成可解析同结构；覆盖率 ≥80%

## 阶段 2：otel-collector 核心 pipeline（Go）

### 2.1 OTLP Receiver
- [ ] 2.1.1 [RED] OTLP gRPC Receiver 单测：接收 ExportTraceServiceRequest，解析为内部 Span 模型，断言字段映射
- [ ] 2.1.2 [RED] OTLP HTTP Receiver 单测：:4318 /v1/traces 接收，同上
- [ ] 2.1.3 [GREEN] 实现 otlp receiver（gRPC:4317 + HTTP:4318，基于 OTel collector builder）
- [ ] 2.1.4 [验收] testkit 上报 OTLP -> receiver 收到 span，字段完整；覆盖率 ≥80%

### 2.2 SkyWalking->OTel 协议转换 processor（R1 高危，需 PoC）
- [ ] 2.2.1 [RED] 字段映射表单测：给定 SkyWalking v3 SegmentSpan，转换后 OTel Span 的 trace_id/span_id/parent/operation/kind/status/duration/tags 逐字段断言（覆盖 HTTP/SQL/Redis/MQ 五类）
- [ ] 2.2.2 [GREEN] 实现 skywalking2otel processor（字段映射表 DD-14 §5）
- [ ] 2.2.3 [RED] 映射缺失/有损字段单测：断言无法映射字段记入 dropped_fields 清单并告警
- [ ] 2.2.4 [GREEN] 实现有损字段记录与告警钩子
- [ ] 2.2.5 [REFACTOR] 映射表抽配置化（支持后续扩字段）
- [ ] 2.2.6 [验收] SkyWalking Agent 真实上报 -> 转换后 OTel Span 关键字段无损，有损清单可查；PoC 报告归档

### 2.3 PII 脱敏 processor
- [ ] 2.3.1 [RED] 脱敏单测：db.statement 含手机号->`138****1234`、http.url query 参数、身份证/银行卡/密码正则掩码；非命中字段不变
- [ ] 2.3.2 [GREEN] 实现 piimask processor（正则清单 DD-14 §4.2，可配白名单 Attribute 键放行）
- [ ] 2.3.3 [RED] 脱敏误伤单测：业务字段误命中身份证正则时经白名单放行
- [ ] 2.3.4 [GREEN] 白名单放行逻辑
- [ ] 2.3.5 [验收] 含 PII 的 span 上报后 ES 不含原始 PII；覆盖率 ≥80%

### 2.4 幂等去重 processor
- [ ] 2.4.1 [RED] 去重单测：同一 trace_id+span_id 重复上报仅通过一次；不同 span_id 并存
- [ ] 2.4.2 [GREEN] 实现 dedup processor（Redis trace:dedup:{traceId} TTL 10m，SETNX）
- [ ] 2.4.3 [RED] Redis 不可用降级单测：去重失败时不阻塞 pipeline（放行+告警）
- [ ] 2.4.4 [GREEN] Redis 不可用降级路径
- [ ] 2.4.5 [验收] 重复上报落库无重复；覆盖率 ≥80%

### 2.5 多中心维度补全 processor
- [ ] 2.5.1 [RED] 维度补全单测：缺 data_center/network_area/room/cluster 的 span 按启动配置补全
- [ ] 2.5.2 [GREEN] 实现 dimension processor（DD-03.5）
- [ ] 2.5.3 [验收] 补全后维度齐全可切片；覆盖率 ≥80%

### 2.6 Kafka Exporter
- [ ] 2.6.1 [RED] Exporter 单测：span->otel-spans、metric->otel-metrics、log->otel-logs、infra->otel-infra-metrics 分 Topic；分区按 trace_id hash 保序；Avro 序列化
- [ ] 2.6.2 [GREEN] 实现 kafka exporter（kafka-go，Avro + Schema Registry v1）
- [ ] 2.6.3 [验收] testkit 上报 -> Kafka 4 Topic 各有数据，分区保序；覆盖率 ≥80%

### 2.7 入口限流与背压
- [ ] 2.7.1 [RED] 限流单测：>1500 QPS 触发背压（429/降级），不爆内存
- [ ] 2.7.2 [GREEN] 实现 ingress limiter（1500 QPS 预留，DD-01 §9）
- [ ] 2.7.3 [验收] 压测 1500 QPS 稳定，超阈背压生效；覆盖率 ≥80%

### 2.8 ES/Kafka 不可用降级本地磁盘缓冲
- [ ] 2.8.1 [RED] 降级单测：Kafka 不可用时 span 写本地有界磁盘队列（不 OOM）；恢复后重传不丢；队列满丢最旧+计数
- [ ] 2.8.2 [GREEN] 实现 disk buffering（有界队列，满丢最旧计数告警）
- [ ] 2.8.3 [验收] 断 Kafka 注入 -> 缓冲有界 -> 恢复重传零丢失（DD-11 C1）；覆盖率 ≥80%

### 2.9 otel-collector 联调与容器化
- [ ] 2.9.1 [验收] collector 端到端：OTLP->转换->脱敏->去重->维度->Kafka，testkit 驱动
- [ ] 2.9.2 [验收] 无状态 HPA + /health + /metrics（黄金指标：span/s、P95、丢弃率、队列深度）
- [ ] 2.9.3 [验收] Helm 部署预发，dmz 代理转发至 biz collector（决议 A）

## 阶段 3：Java Agent（SkyWalking，注入+埋点+MDC）

### 3.1 Agent 注入（K8S initContainer / VM）
- [ ] 3.1.1 [RED] 注入脚本单测：initContainer 正确生成 `java -javaagent:skywalking.jar -Dskywalking.collector.backend_service=... -Dservice_name=... -Dsample_rate=... -jar app.jar`
- [ ] 3.1.2 [GREEN] 实现 K8S initContainer 注入脚本 + VM 手动注入文档
- [ ] 3.1.3 [验收] SpringBoot 应用注入后启动正常，Agent 加载日志可见；ClassLoader 隔离无类冲突（DD-13 §3）

### 3.2 自动埋点（Web/RPC/SQL/Redis/MQ/运行环境）
- [ ] 3.2.1 [RED] 自动埋点单测：SpringBoot Web 请求->生成 http.server.request span；JDBC->mysql.query span；Redis->redis.command span；Kafka produce/consume->kafka.* span；JVM 指标采集
- [ ] 3.2.2 [GREEN] 集成 SkyWalking Agent 插件（Web/RPC/SQL/Redis/MQ/运行环境）
- [ ] 3.2.3 [RED] 跨 MQ 上下文透传单测：生产者与消费者 span 共享 trace_id
- [ ] 3.2.4 [GREEN] 配置 MQ 上下文透传
- [ ] 3.2.5 [RED] 插件冲突禁用单测：冲突插件默认禁用并告警
- [ ] 3.2.6 [GREEN] 插件热插拔 + 冲突禁用告警
- [ ] 3.2.7 [验收] testkit scenario S1(db)/S2(mq) 产出对应 span 经 collector 入 Kafka；覆盖率 ≥80%

### 3.3 日志 MDC 注入 TraceID（决议 C）
- [ ] 3.3.1 [RED] MDC 注入单测：存在 TraceContext 时日志文本含 traceId 且 = 链路 trace_id；无 Context 时降级（不报错）
- [ ] 3.3.2 [GREEN] 实现 SkyWalking Log 插件 MDC 注入（MDC.put("traceId",...)）+ Logback Pattern `%X{traceId}`
- [ ] 3.3.3 [验收] testkit scenario S5(log) 日志 traceId = 链路 traceId；覆盖率 ≥80%

### 3.4 全栈指标采集
- [ ] 3.4.1 [RED] 指标采集单测：JVM GC/线程/死锁、接口 QPS/耗时/错误码/状态码、CPU/内存/磁盘/网络、中间件客户端吞吐/耗时/错误率
- [ ] 3.4.2 [GREEN] 配置 SkyWalking 指标插件 + 资源/中间件采集
- [ ] 3.4.3 [验收] 指标经 OTLP 入 otel-metrics；覆盖率 ≥80%

### 3.5 代码级链路开关
- [ ] 3.5.1 [RED] 开关单测：开启->方法级明细+堆栈采集；关闭->停止；≤30s 生效
- [ ] 3.5.2 [GREEN] 实现代码级开关响应 config-center 下发
- [ ] 3.5.3 [验收] 开关热切换无重启生效；覆盖率 ≥80%

## 阶段 4：Go Agent（OTel SDK 手动埋点）

- [ ] 4.1 [RED] Go 埋点单测：gin HTTP handler->http.server.request span；grpc-go->rpc span；net/http client->http.client.request span；遵循 DD-03.5 命名/tag
- [ ] 4.2 [GREEN] 实现 Go OTel SDK 埋点封装（gin/grpc-go/net/http + DB/Redis/MQ）
- [ ] 4.3 [RED] Go 跨 MQ 透传单测：produce/consume 共享 trace_id
- [ ] 4.4 [GREEN] MQ 上下文透传
- [ ] 4.5 [RED] Go MDC 等价单测：日志含 trace_id（context.Logger）
- [ ] 4.6 [GREEN] 日志 trace_id 注入（zap + context）
- [ ] 4.7 [验收] Go 应用上报链路经 collector 入 Kafka；覆盖率 ≥80%

## 阶段 5：采样双模 + 自适应（D3/D4/D5）

### 5.1 头部采样（Agent 侧，D3 试点 100%）
- [ ] 5.1.1 [RED] 头部采样单测：sample_rate=100%->全采；=50%->约半采（统计偏差内）；错误/慢不在此层决策
- [ ] 5.1.2 [GREEN] 实现 head sampler（Java+Go，读 agent:config sample_rate，低基数键决策）
- [ ] 5.1.3 [验收] 试点 100% 落库率 100%（D3）；覆盖率 ≥80%

### 5.2 尾部采样（Collector 侧，D5）
- [ ] 5.2.1 [RED] 尾部采样单测：错误 span(status.code=1)->100% 采；慢请求(>2s)->100% 采；普通->按头部结果；超长(>50000 spans)->硬截断计数仍保留
- [ ] 5.2.2 [RED] 缓冲单测：window=10s 内同 trace_id span 收齐后决策；in-flight=20000 超限丢最旧+计数告警；内存 ≤300MB
- [ ] 5.2.3 [GREEN] 实现 tail sampler（trace_id 分组缓冲 window=10s/in-flight=20000，DD-07 §3.2）
- [ ] 5.2.4 [REFACTOR] 缓冲池与淘汰策略优化
- [ ] 5.2.5 [验收] 错误/慢必采，普通按比例；超限不 OOM；覆盖率 ≥80%

### 5.3 自适应采样（D4，依赖 config-center + 派生指标）
- [ ] 5.3.1 [RED] 自适应单测：错误率↑->采样↑、流量↑->采样↓、耗时异常->↑慢请求采样，步进 ±10%，有下限（保错误/慢必采+最小样本）
- [ ] 5.3.2 [GREEN] 实现规则触发调档逻辑（config-center 侧计算，读 npm_online_performe 派生指标）
- [ ] 5.3.3 [验收] 流量/错误率突变采样率 ±10% 平滑无抖动（D4）；覆盖率 ≥80%
- [ ] 5.3.4 [说明] alert-engine 派生指标计算在 add-alert-center(M5.1) 落地，M1 阶段先用静态配置兜底（D4 方案 C），M5.1 后切规则触发

## 阶段 6：config-center（Go，配置下发+热更新）

### 6.1 配置存储与 CRUD
- [ ] 6.1.1 [RED] t_agent_config CRUD 单测：scope_type(cluster/engineering/agent)+scope_id 唯一；sample_rate/max_tps/sampling_mode/enabled/config_version 字段
- [ ] 6.1.2 [GREEN] 实现 config-center 配置 CRUD gRPC（Watch/Publish RPC）+ MySQL t_agent_config/t_agent_config_history
- [ ] 6.1.3 [验收] 配置变更记历史表；覆盖率 ≥80%

### 6.2 热下发 ≤30s
- [ ] 6.2.1 [RED] 热下发单测：Publish 配置->Redis agent:config:{clusterId}+config:version 更新->Agent Watch ≤30s 拉取生效
- [ ] 6.2.2 [GREEN] 实现 config-center 发布 + Redis 写 + Agent Watch（长轮询/gRPC stream）
- [ ] 6.2.3 [验收] 采样率变更 ≤30s 生效无重启（DD-07 §6）；覆盖率 ≥80%

### 6.3 灰度分组下发
- [ ] 6.3.1 [RED] 灰度单测：仅命中灰度分组 Agent 应用新配置，未命中保持原配置
- [ ] 6.3.2 [GREEN] 实现灰度分组下发
- [ ] 6.3.3 [验收] 灰度隔离正确；覆盖率 ≥80%

## 阶段 7：agent-registry（Go，注册+心跳+台账+升级+红线）

### 7.1 注册与令牌
- [ ] 7.1.1 [RED] 注册单测：Agent 注册->签发 token（AES-256-GCM 加密存 t_agent_registration）；重复注册 idempotent；非法字段拒绝
- [ ] 7.1.2 [GREEN] 实现 agent-registry 注册 gRPC + 令牌签发
- [ ] 7.1.3 [验收] 注册令牌加密存储；覆盖率 ≥80%

### 7.2 心跳与存活
- [ ] 7.2.1 [RED] 心跳单测：周期心跳更新 Redis agent:heartbeat + t_agent_instance.last_heartbeat；超 60s 标离线
- [ ] 7.2.2 [GREEN] 实现心跳 gRPC + 离线判定
- [ ] 7.2.3 [验收] 离线 Agent 台账标记+P1 告警（DD-06）；覆盖率 ≥80%

### 7.3 Agent 台账查询
- [ ] 7.3.1 [RED] 台账单测：按集群/机房/zone/IP/服务/健康/版本筛选查询
- [ ] 7.3.2 [GREEN] 实现台账查询 gRPC + Redis agent:heartbeat 读
- [ ] 7.3.3 [验收] 台账数据准确；覆盖率 ≥80%

### 7.4 批量灰度升级与回滚
- [ ] 7.4.1 [RED] 升级单测：按 cluster_id/engineering_id 推送目标版本，金丝雀 1%->10%->100%；异常（心跳丢失率突增）自动回滚
- [ ] 7.4.2 [GREEN] 实现 /agents/upgrade 推送 + 自动回滚判定
- [ ] 7.4.3 [验收] 升级异常自动回滚，期间数据经缓冲不丢；覆盖率 ≥80%

### 7.5 兼容性矩阵
- [ ] 7.5.1 [RED] 矩阵单测：查询 Java/Go 版本+框架兼容性；EOL（JDK6/7）拒绝
- [ ] 7.5.2 [GREEN] 实现 /agents/compatibility + CI 兼容性测试卡点
- [ ] 7.5.3 [验收] 新版本入矩阵须过 CI；覆盖率 ≥80%

### 7.6 资源红线与自动熔断（D11）
- [ ] 7.6.1 [RED] 红线单测：CPU<1%/内存<128M/线程≤业务5%/丢包<0.1% 监控；超阈告警
- [ ] 7.6.2 [RED] 熔断单测：业务 CPU>95% 持续 N 秒->自动熔断/降采样；内存逼近红线->丢缓冲降采样；Collector 不可达->本地缓冲+心跳异常
- [ ] 7.6.3 [RED] 白名单单测：重负载应用经 t_agent_config 白名单调高上限
- [ ] 7.6.4 [GREEN] 实现资源红线监控 + 自动熔断 + 白名单（Agent 侧 + registry 侧）
- [ ] 7.6.5 [验收] CPU 打满自动熔断保业务（D11）；覆盖率 ≥80%

### 7.7 Agent 自身指标回流（Dogfooding）
- [ ] 7.7.1 [RED] 回流单测：Agent CPU/内存/线程/丢包指标经 OTLP 回流本平台
- [ ] 7.7.2 [GREEN] 实现 Agent 自监控指标上报
- [ ] 7.7.3 [验收] Agent 异常先于业务方暴露（DD-05）；覆盖率 ≥80%

## 阶段 8：前端采集治理页（Vue3）

- [ ] 8.1 [RED] Collector 实例列表页单测：/collectors 列表渲染+健康/QPS/丢包；/collectors/status 总览
- [ ] 8.2 [GREEN] 实现 ingest 治理页（Collector 列表/状态/管道指标 /ingest/metrics + /ingest/health）
- [ ] 8.3 [RED] Agent 管理页单测：台账列表+筛选、配置下发面板（采样率/开关/灰度）、升级操作、兼容矩阵
- [ ] 8.4 [GREEN] 实现 admin/agents 页（配置下发 /agents/config、升级 /agents/upgrade、兼容性 /agents/compatibility）
- [ ] 8.5 [验收] 页面状态覆盖（加载/空/异常/无权限/延迟/降级）；Playwright E2E 配置下发->≤30s 生效

## 阶段 9：testkit M1 场景填充

- [ ] 9.1 [RED] S1 数据库访问场景：产出 db span（正常/慢SQL/异常SQL）
- [ ] 9.2 [GREEN] 实现 testkit scenario-db（随 Java Agent 3.2）
- [ ] 9.3 [RED] S2 Kafka/中间件场景：产出 mq/cache span + 跨服务调用
- [ ] 9.4 [GREEN] 实现 scenario-mq
- [ ] 9.5 [RED] S4 error 请求场景：status=ERROR span
- [ ] 9.6 [GREEN] 实现 scenario-error/scenario-fault
- [ ] 9.7 [RED] S8 混沌联动场景：Kafka/ES 断连下持续上报
- [ ] 9.8 [GREEN] 实现 scenario-chaos 联动
- [ ] 9.9 [验收] S1/S2/S4/S8 场景稳定产出对应链路，可被 ES 检索

## 阶段 10：跨组件联调验收（Gate，对齐 DD-17 §6.1 M1）

> 全链路联调：Agent -> Collector -> Kafka -> ingestion(占位/M2) -> 可观测。

- [ ] 10.1 [联调] Java Agent -> otel-collector：SkyWalking 数据经转换入 Kafka otel-spans，字段无损
- [ ] 10.2 [联调] Go Agent -> otel-collector：OTel 数据透传入 Kafka
- [ ] 10.3 [联调] dmz Agent -> dmz 代理 -> biz collector -> 上海 DC1 Kafka（决议 A/D8）
- [ ] 10.4 [联调] config-center 下发采样率 -> Agent ≤30s 生效 -> collector 尾部采样行为变化
- [ ] 10.5 [联调] agent-registry 心跳 -> 台账在线率 -> 离线告警（DD-06）
- [ ] 10.6 [联调] 资源红线超限 -> Agent 熔断 -> registry 标记 -> 前端台账红显（D11）
- [ ] 10.7 [联调] Kafka 断连 -> collector 本地缓冲 -> 恢复重传零丢失（DD-11 C1）
- [ ] 10.8 [联调] 跨中心 Trace 拼接：上海服务->苏州服务同一 128bit trace_id 重组完整链（D7）
- [ ] 10.9 [验收] DD-17 §6.1 M1 全项：OTLP 双协议/头部 100%(D3)/尾部错误慢必采+window10s-inflight20000(D5)/自适应±10%(D4)/幂等去重/biz-dmz 双路(D8)/断连不丢(C1)/红线熔断(D11)/非法 Agent 拒绝/SW->OTel 无损
- [ ] 10.10 [验收] 覆盖率报告：采集/采样/告警相关 ≥80%（DD-11 §3），全量单测绿
- [ ] 10.11 [验收] 压测 3× 流量验证背压/降级/不丢（D10 试点阶段），报告归档
- [ ] 10.12 [验收] 混沌 C1(Kafka 断)/C3(Collecter 杀)/C5(Agent CPU 打满) 用例通过（DD-11 §5）

