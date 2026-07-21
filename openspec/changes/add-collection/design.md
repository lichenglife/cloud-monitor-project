## Context

M1 为平台数据源头，设计已由 DD-07（采集与采样）、DD-13（Agent 治理）、DD-08（多中心）详定，本设计不重复，仅固化关键架构决策与实现要点。技术选型经议题 E 决议：Collector 基于 OTel 0.90+ 定制，Java=SkyWalking Agent 自动埋点、Go=OTel SDK 手动埋点（PRD REQ-M1-U3），数据集中上海 DC1 Kafka（决议 A）。

采集架构（DD-07 §1）：
```
业务应用（Java=SkyWalking 自动 / Go=OTel SDK 手动）
   │ OTLP gRPC:4317 / HTTP:4318（biz 直连；dmz 经代理转发）
   ▼
Collector（接收->解析->采样->转换->导出，无状态可 HPA）
   │ 幂等去重(trace_id+span_id) + 背压
   ▼
Kafka（上海 DC1：otel-spans/metrics/logs/infra-metrics）
   ▼
消费侧（topology-calc / 聚合服务）-> ES 6 索引 + MySQL + Redis
```

## Goals / Non-Goals

**Goals:**
- OTel 定制 Collector 接收 OTLP，统一转换 SkyWalking/OTel->OTel，导出 Kafka（上海 DC1），无状态可 HPA
- Java/Go Agent 注入与埋点，全栈指标采集，跨 MQ 透传，日志 MDC 注入 TraceID
- 采样双模 + 自适应（D3/D4/D5），热配置 ≤30s 生效
- Agent 治理（配置/版本/心跳/灰度/台账/资源红线 D11/熔断）
- 多中心管道（决议 A：各中心 biz Collector + dmz 代理 + 上海 DC1 汇聚），TraceID 真 128bit（D7）

**Non-Goals:**
- 不实现 ES 聚合索引写入与查询（M2/M3）
- 不实现拓扑计算（M4，消费 otel-spans）
- 不实现日志平台 SDK 改造（M6，本 change 仅在 Agent 侧 MDC 注入 TraceID，日志平台侧改造归 M6/议题 C）
- 不对齐 CMDB 真实权限接口（M15，Agent 注册暂用本地令牌）
- 不做 eBPF 无侵入采集（PRD REQ-M1-O1，后续 PoC）

## Decisions

### DM1: Collector 定制开发基于 OTel 0.90+，自定义 Processor 做协议转换
- Receiver：OTLP gRPC:4317 / HTTP:4318（标准）。
- Processor：① SkyWalking v3 -> OTel Span 自定义 processor（字段映射表见 DD-14 §5，需 PoC 验证无损）；Go OTel 已是 OTel 格式直接透传；② PII 脱敏（DD-14 §4.2，db.statement/http.url/自定义 Attribute 正则掩码）；③ 多中心维度补全（data_center/network_area/room/cluster，DD-03.5）；④ 幂等去重（trace_id+span_id -> Redis `trace:dedup:{traceId}` TTL 10m）。
- Exporter：Span/Metric/Log/Infra 分 Topic 写 Kafka（上海 DC1），分区按 trace_id hash 保序。
- 无状态，K8S HPA 按 CPU/吞吐；ES 不可用时降级本地磁盘缓冲（有界队列，不反向压垮 Agent）。

### DM2: 采样双模 + 自适应（D3/D4/D5）
- 头部采样（Agent 侧）：请求入口即时决策，试点 100% 全量（D3），普通流量按 sample_rate 比例，决策依据低基数键（http.status_code/db.operation，DD-03.5）。
- 尾部采样（Collector 侧）：持有完整 Trace 后决策，错误 Span（status.code=1/error=true）100% 采，慢请求（Trace 总耗时 >2s）100% 采，超长 Trace（>50000 spans，DD-03 §9.4）硬截断并计数仍保留。缓冲 window=10s / in-flight=20000（D5），内存量化 100-300MB/Collector 实例（DD-07 §3.2）。
- 自适应（D4）：规则触发调档--错误率↑->采样↑、流量↑->采样↓、耗时百分位异常->↑慢请求采样，步进 ±10%；下限保错误/慢必采 + 最小可观测样本。由 alert-engine/config-center 基于 npm_online_performe 派生指标计算，经 config-center 下发 agent:config，Agent ≤30s 生效。

### DM3: Agent 注入与埋点（DD-13）
- Java：SkyWalking Agent 字节码增强，ClassLoader 隔离（AgentClassLoader 独立于业务 AppClassLoader），自动埋点 Web/RPC/SQL/Redis/MQ/运行环境；插件热插拔，冲突插件禁用告警。
- Go：OTel SDK 手动埋点，原生 OTLP 导出。
- 注入：K8S initContainer 注入 `java -jar -agent:xxx`（含 Collector ip+port、service_name、instance_name、sample_rate 启动参数）；VM 由运维手动改启动命令。
- 日志 MDC：Agent 在日志打印时无感注入 TraceID 到 MDC（决议 C），日志平台 SDK 改造复用此 TraceID（M6 落地日志平台侧）。

### DM4: Agent 治理（DD-13）
- 注册/心跳：Agent 启动向 agent-registry 注册（签发 t_agent_registration.token），周期心跳 PushAgentHeartbeat，last_heartbeat 超 60s 标离线。
- 配置下发：t_agent_config（源）-> config-center -> Redis agent:config:{clusterId} + config:version -> Agent Watch ≤30s 生效。
- 灰度升级：/agents/upgrade 按 cluster_id/engineering_id 推送目标版本，金丝雀 1%->10%->100%，异常（心跳丢失/资源超红线/业务报错）自动回滚。
- 资源红线（D11）：统一 CPU<1%/内存<128M/线程≤业务5%/丢包<0.1%，不按应用分级；重负载应用白名单调高（t_agent_config 维度管控）。
- 自动熔断：业务 CPU>95% 持续 N 秒 -> Agent 自动熔断/降采样，优先保业务；内存逼近红线 -> 丢缓冲降采样；Collector 不可达 -> 本地磁盘缓冲（有界）+ 心跳异常标记。

### DM5: 多中心管道（决议 A/D8/D7）
- 各中心 biz 区部署 Collector 集群（本地接收+一级处理），dmz 区部署代理（HAProxy/Nginx）转发至本中心 biz Collector。
- 所有 Collector 数据汇聚上海 DC1 Kafka（RF≥3 跨 AZ，D8）；跨中心仅传已采样/已处理数据，节省专线带宽。
- TraceID 真 128bit（D7：高64[续位23+随机41] ‖ 低64[时间低41+中心3+节点10+序列10]），生成点 Agent 入口，跨中心以完整 128bit trace_id 拼接，Kafka 分区按 trace_id hash 保序。

### DM6: 幂等与背压（可靠性红线）
- Collector 入口限流 1500 QPS 预留（试点 500×3，DD-01 §9），超阈背压。
- 幂等去重 trace_id+span_id（Redis #4 TTL 10m）。
- 降级优先级：保采集 > 保聚合展示；存储压力大时允许丢非错误/非慢链路 Trace（DD-04 成本原则）。

## Risks / Trade-offs

- **SkyWalking->OTel 字段映射有损风险**（R1 高危）：映射表待 PoC 验证（DD-14 §5）。缓解：M0 阶段优先 PoC，验证关键字段（trace_id/span_id/operation/tags/status）无损，有损字段记映射缺失清单。
- **Agent 资源红线超限致业务方拒接入**：缓解：D11 统一基线 + 重负载白名单 + 自动熔断，Agent 自身指标回流 Dogfooding（DD-05）先于业务方暴露异常。
- **尾部采样内存压力**：window=10s/in-flight=20000 在峰值可能 100-300MB/实例。缓解：Collector 堆预留 ≥1GB，尾部缓冲占 ≤30%，in-flight 超限丢最旧在途 Trace 并计数告警（DD-06）。
- **海量 Agent 配置下发 Redis 压力**（终态 5000+ 工程）：一期 300 工程可承受，终态需评估 Watch 长连接数与分片/分组推送（DD-13 §9 待评审）。
- **日志平台 SDK 改造跨团队依赖**（议题 C/R5）：本 change 仅做 Agent 侧 MDC 注入，日志平台侧改造归 M6，排期不在本团队控制。缓解：Agent 侧先就绪，MDC 注入可独立验证。
