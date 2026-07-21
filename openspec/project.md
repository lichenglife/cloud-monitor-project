# 项目背景与技术基线（Project Context）

> 本文件是全链路追踪分析平台一期所有 OpenSpec change 的**共享上下文基线**。创建任何 proposal/design/spec/tasks 时须遵循此处的技术栈、约定、外部依赖与命名规范。
>
> 详细需求与设计见 `.claude/reference/`（需求规划 v3、PRD v1.2、系统设计说明书 v1.0、DD-01~DD-17、评审决议清单 D1-D13）。本文为基线索引，不重复完整设计。

---

## 1. 项目背景

**全链路追踪分析平台（一期，9 个月）**：绿地建设，补齐应用链路层（Trace）监控并整合既有指标/日志底座，构建"系统内 + 跨系统 + 多中心"的全局链路拓扑与三域关联诊断能力。

- **北极星**：分钟级故障定位 + 资源利用率可量化提升。
- **现状基线**：当前无应用链路层监控、无 OTel 采集，仅有基础设施/容器/日志三类监控。本平台为"新建链路采集 + 整合既有指标/日志"。
- **一期范围**：10 套系统 / 300 工程 / 上海苏州两地三中心 / 峰值 QPS≈500 / 存储 1 年 / 查询 P95<1s。
- **成熟度目标**：L1 全量落地（链路采集 + 三域关联 + 全局拓扑 + 故障快速定位）+ L2 部分（火焰图/热点方法/健康评分，议题 B 纳入一期）。
- **不在一期**：M8 成本优化、M9 深度剖析、M10 深度 RCA、M11 SLO、M12 压测、M13 混沌（M13 仅做工程级故障演练 DD-11/DD-17）、深度合规脱敏、全量 200 套接入。

### 1.1 一期功能模块（12 个，按生命周期推进）

| 生命周期序 | 模块 | 能力 | 依赖 |
|---|---|---|---|
| M1 | 统一数据采集层 | OTel 定制 Collector + Java=SkyWalking 自动 / Go=OTel SDK 手动埋点 + 全栈指标 + 采样双模 + Agent 管理 | 外部：议题 A 部署 |
| M2 | 统一接入与存储 | MySQL+Redis+ES+Kafka 分层，存储 1 年，P95<1s | M1 |
| M3 | 链路追踪与检索 | TraceID 检索、瀑布图、代码栈、下钻方法/SQL | M2 |
| M4 | 全局链路拓扑 | 自动发现 + CMDB 元数据 + 强弱依赖 + 纵向下钻 | M2 |
| M5 | 故障影响分析 | 上下游依赖、爆炸半径 | M4 |
| M5.1 | 统一告警中心 | 规则、动态基线、降噪/收敛、通知、Oncall | M2 |
| M6 | 三域关联诊断 | Trace×指标×日志（含链路关联日志改造） | M2, M3 + 外部：日志平台 |
| M7 | 多中心链路汇聚 | 两地三中心，数据集中上海 DC1 | M1, M2 |
| M14 | 统一可观测视图 | 大盘、性能监测、topN、热点方法、火焰图、服务健康评分 | M2, M3, M4 |
| M15 | 权限与合规 | 基础权限对接 CMDB | M2 + 外部：CMDB |
| M16 | 运营管理平台 | 用户/项目/集群/Agent 台账/系统字典/审计 | M1 |
| M17 | 业务场景设置 | 入口 URL 业务标签 | M3 |

### 1.2 关键架构约束（落地硬约束）

- **两地三中心 + DMZ**：上海 DC1（主/汇聚中心）、苏州 DC2；各机房网络互通；**同机房 biz 区域与 dm 区域网络不通**。
- **部署架构（决议 A）**：各中心 biz 区部署 Collector；各中心 dmz 区部署代理转发至本中心 Collector；所有 Collector 数据汇聚至**上海 DC1 Kafka**；管理平台与全部 DB（Redis/MySQL/Kafka/ES）部署在上海 DC1 biz 区；用户查询经上海 DC1 biz 的 ES/MySQL。
- **混合部署**：K8S（1.21/1.25）容器化为主 + 部分虚拟机；中间件 MySQL/MongoDB/Oracle/Redis/Kafka/RabbitMQ/MQ/Tomcat/BES、nginx。

---

## 2. 技术栈（决议 E + D1）

| 层 | 选型 | 版本 | 决策来源 |
|---|---|---|---|
| 采集 - Java Agent | SkyWalking Agent（字节码增强，自动埋点） | 9.x/10.x | 决议 E |
| 采集 - Go Agent | OpenTelemetry Agent/SDK（手动埋点） | 1.x | 决议 E |
| Collector | 基于 OpenTelemetry 定制开发（统一 SkyWalking/OTel->OTel 协议转换） | 0.90+ | 决议 E |
| 后端语言 | Java（SpringBoot 3.2.x 主 / 2.7.x 兼容，SpringCloud 2023.x）+ Go 1.22+ | JDK 17 主 / 11 兼容 | 决议 E |
| 前端 | Vue 3.4+ + `<script setup>` + Composition API + Pinia + Vue Router v4 + Vite 5 | - | D1 |
| 前端可视化 | @antv/g6 v5（WebGL，拓扑）、ECharts 5、自研 Canvas+Web Worker（瀑布/火焰） | - | D2 |
| 存储管道 | Apache Kafka | 3.6+ | 决议 D |
| 链路检索 | Elasticsearch（按天 Rollover 索引，ILM 冷热分层） | 8.12+ | 决议 D |
| 关系型 | MySQL | 8.0.35+ | 决议 D |
| 缓存 | Redis Cluster | 7.2+ | 决议 D |
| 反向代理/LB | Nginx / HAProxy（dmz 代理） | 1.24+ / 2.6+ | 决议 A |
| API 网关 | Spring Cloud Gateway / Kong | - | - |
| 容器编排 | Kubernetes | 1.25 主 / 1.21 兼容 | 决议 E |
| CI/CD | Jenkins / GitLab CI + Helm Charts | Helm 3.14+ | - |

### 2.1 13 个微服务（系统设计说明书 §2.6）

`tracing-gateway` / `trace-query-service` / `metric-aggr-service` / `topology-service` / `alert-service` / `config-center` / `auth-service` / `agent-registry` / `ingestion-service` / `scenario-service` / `audit-service` / `otel-collector` / `tracing-frontend`。

- 服务间优先 gRPC（高性能），对外/遗留兼容 RESTful JSON。
- 服务注册发现：K8S CoreDNS+Service（同集群）或 Consul/Nacos（跨集群按需）。
- 配置：ConfigMap（静态）+ config-center（动态运行配置）双轨。

---

## 3. 外部依赖（须对齐接口规范，PRD Q4/Q5 / 研发任务拆解 §4.2）

| 系统 | 提供能力 | 接口 | 状态 |
|---|---|---|---|
| **CMDB** | 工程/系统名、应用/运维负责人、权限、机房、数据中心 | 元数据/权限查询 RESTful API（只读） | ⛔待对齐（Q4：权限接口字段与最小权限兜底） |
| **日志平台** | 应用日志检索 | 日志查询 API；**SDK 改造复用 Agent TraceID**（MDC 注入，决议 C） | ✅已定一期，改造工作量待评估 |
| **统一监控告警平台** | 物理/虚拟机基础设施指标、中间件/数据库指标、告警规则与事件 | 指标查询/告警订阅 webhook | ⛔待对齐（Q5：指标接口规范） |
| **容器平台** | K8S 集群信息、Pod/Node 元数据、Prometheus 容器资源指标、**initContainer 注入能力** | Kubernetes API + Prometheus API | 已定（决议 C/E） |
| **网络/安全** | 跨区通道、防火墙 | **biz Collector + dmz 代理转发** | ✅已定（决议 A） |

> 对接策略：外部依赖先 mock + 契约测试，接口对齐后切真实。CMDB 权限接口规范（Q4）是 M4/M15/M16 的前置阻塞。

---

## 4. 约定（编码/接口/数据/安全）

### 4.1 接口契约

- **REST 基址** `/api/v1`；管理平台接口经 API 网关（鉴权/限流/审计）。
- **统一响应结构**：`{ "code": 0, "message": "success", "data": {}, "traceId": "..." }`，`traceId` 用于 Dogfooding 链路定位。
- **接口路径/页面归属/关键参数**：见 DD-01 §3（骨架已定稿 v1.0）。**字段级 request/response 明细**：在对应模块 change 的 design.md 中按模块深化，并以 OpenAPI 3.0 + gRPC .proto 落地（地基 change 产出）。
- **错误码**（DD-01 §7）：0 成功 / 40001 参数校验 / 40101 未认证 / 40301 无权限（CMDB 兜底）/ 40901 幂等冲突 / 42901 限流 / 50001 内部错误（带 traceId）。
- **gRPC 服务间**：默认超时 1000ms；仅幂等 RPC 重试 2 次、指数退避（200ms->400ms）；非幂等写不重试。
- **限流**：API 网关集群 2000 QPS / 单用户 100 QPS；Collector OTLP 入口按试点峰值 500×3≈1500 QPS 预留。
- **版本化与向后兼容**：REST 路径含 `/v1`；Kafka 消息 Avro + Schema Registry，新增字段兼容、删改字段升 `v2`。

### 4.2 数据模型（DD-02）

- **MySQL**：元数据（用户/项目/集群/Agent 台账/配置/审计）。通用审计字段（`created_by/created_time/updated_by/updated_time`）所有表必含。分表阈值：单表 >2000 万行或 >50GB 按 `project_id`/`engineering_id` 分表（Z 轴）。实体三级归属：项目->工程->Agent 实例。
- **Redis**：14 类 Key（agent 配置/心跳、拓扑缓存、trace 去重、限流、会话、权限缓存、健康评分、实时指标等）。统一 `allkeys-lru`；热配置常驻不淘汰；单 Value <10KB；多级缓存 Caffeine+Redis。
- **ES**：7 大索引（1 原始 `trace_npm_jaeger` + 6 聚合 `npm_online_metric/db/performe/topology/event/infra`）。`trace_npm_jaeger` 按天 Rollover（`-YYYY.MM.DD`），单日索引初始 6 主分片+1 副本；聚合索引 3 主分片+1 副本。维度字段全 keyword 低基数；`attributes` 用 `flattened`；禁 UserID/OrderID 入索引。
- **Kafka**：Topic `otel-spans`(12分区)/`otel-metrics`(6)/`otel-logs`(6)/`otel-infra-metrics`(6)/`topology-calc`(3)/`alert-events`(3)/`agent-config`(3)；分区按 `trace_id`/`resource_id` hash 保序；Avro+Registry 版本化。

### 4.3 埋点规范（DD-03.5）

- Span 命名 `{组件类型}.{操作}`（`http.server.request`/`mysql.query`/`redis.command`/`kafka.produce` 等），禁业务参数入名。
- 语义约定对齐 OTel Semantic Conventions v1.21+；强制键 `service.name`/`span.kind`/`status.code`/`trace.id`/`span.id`/各组件 `system`。
- 单 Span ≤30 个索引 Tag；Value 须枚举/低基数；禁 UserID/OrderID/手机号/身份证/明文 token 作 Tag 或索引字段。
- PII 源头脱敏（Agent/Collector Transform，DD-14 §4.2）：手机号 `138****1234`、身份证前6后4、密码/token 整体 `***`、银行卡前6后4。

### 4.4 安全（DD-09）

- **零信任**：内外网访问均鉴权；mTLS 全链路；Agent 注册令牌 + TLS。
- **权限**：平台 RBAC + CMDB 数据权限兜底；无显式授权默认拒绝（最小权限，决议 D）。CMDB 权限分钟级同步 + 缓存 5m + 失效默认拒绝（D12）。
- **加密**：敏感配置 AES-256-GCM（KMS 托管主密钥，定期轮换）；传输 TLS 1.2+/mTLS；明文不落日志/ES。
- **审计**：全关键操作（配置变更/权限调整/Agent 升级/数据导出）写 `t_audit_log`，append-only 不可篡改。

### 4.5 工程规范（DD-11/DD-17）

- **分支模型**：`main`(生产,保护) / `release/x.y` / `develop` / `feature/任务号-简述`。提交 `[任务号] 动作：简述`，关联 TAPD 任务 ID。
- **合入**：PR ≥1 人 Review + CI 全绿（编译/单测/Lint/兼容性）方可入 develop；main 仅经 release 合并。
- **测试覆盖率**：核心逻辑 ≥70%，采集/采样/告警 ≥80%；接口契约与 DD-01 一致；静态扫描无高危。
- **发布**：蓝绿/金丝雀（1%->10%->100%），可一键回滚；config-center 热更新 ≤30s 生效无需重启。
- **混沌演练**（D13）：预发每迭代 + 生产季度级（变更窗口+审批），重大版本前必做。
- **压测**（D10）：试点期 3× 验证背压/降级，上线前 10× 全量压测出具报告。

---

## 5. 命名规范

### 5.1 OpenSpec change 命名

- **格式**：`{type}-{capability}`，kebab-case，小写。
- **type**：`add`（新增功能模块）/ `chore`（基础设施/工程化，非功能）/ `feat`（功能增强，二期）/ `fix`（缺陷修复）。
- **capability**：英文短词，对应功能模块或基础设施域。
- **模块编号不入名**（保持可读），但在 change 的 proposal.md "Impact" 段标注对应 PRD 模块号（如 M1）。
- **示例**：`chore-foundation`（地基）、`add-collection`（M1 采集）、`add-storage-ingestion`（M2）、`add-trace-query`（M3）、`add-topology`（M4）、`add-impact-analysis`（M5）、`add-alert-center`（M5.1）、`add-correlation`（M6）、`add-multicenter`（M7）、`add-observability-view`（M14）、`add-permission`（M15）、`add-ops-console`（M16）、`add-business-scenario`（M17）。

### 5.2 capability（spec 目录）命名

- kebab-case，单一职责，对应一个可独立验收的能力域。
- change 的 `specs/<capability>/spec.md` 中 `<capability>` 与 change 名中的 capability 一致（如 `add-collection` -> `specs/collection/spec.md`）。
- 跨多个 capability 的 change（如地基含 contracts+scaffold+infra），按子域拆多个 spec 目录。

### 5.3 代码/服务命名

- 微服务：见 §2.1 英文名（`trace-query-service` 等），小写连字符。
- Java 包：`com.{company}.npm.trace.<service>`（如 `com.lichenglife.npm.trace.query`）。
- Go module：`github.com/lichenglife/cloud-monitor/<service>`。
- 前端 feature 目录：与 DD 模块对齐（`features/dashboard`/`features/topology`/`features/traces`/`features/apm`/`features/infra`/`features/alerts`/`features/admin`/`features/ingest`）。

### 5.4 文档交叉引用

- change 内引用设计依据时写明 DD 编号 + 节，如"采样双模见 DD-07 §3 / 决议 D3/D5"，避免重复设计细节。
- 引用决议时写"决议 A/B/C/D/E"或"D1-D13"。

---

## 6. 评审决议索引（已固化，编码须遵循）

| 决议 | 内容 | 落地 |
|---|---|---|
| A | 数据集中上海 DC1；biz 直连 / dmz 代理转发 | DD-07/DD-08 |
| B | L2 能力（火焰图/热点/健康评分）纳入一期 | DD-03 §9/DD-15 |
| C | 日志平台复用 Agent TraceID（MDC 注入） | DD-03 §7/DD-14 |
| D | CMDB 权限兜底、最小权限 | DD-09 |
| E | OTel 定制 Collector + Java=SkyWalking Agent 自动 / Go=OTel SDK 手动埋点（PRD REQ-M1-U3） + 自研前端 | DD-07/DD-15 |
| D1 | 前端 Vue3 线（React 备选冻结） | DD-15 v0.3 |
| D2 | 拓扑渲染 G6 WebGL+LOD+聚类（B 降级）；后端重新聚合落库 ES | DD-15/DD-16 |
| D3 | 试点采样 100% 全量 | DD-07 |
| D4 | 自适应采样规则触发调档 ±10% | DD-14 |
| D5 | 尾部采样 window=10s / in-flight=20000 | DD-07 |
| D6 | 告警引擎流式消费预聚合流 + 静态阈值一期 | DD-06 |
| D7 | TraceID 真 128bit（高64[续位23+随机41] ‖ 低64[时间低41+中心3+节点10+序列10]） | DD-14 §3 |
| D8 | Kafka 仅上海 DC1 + 关键 Topic RF≥3 跨 AZ | DD-08 |
| D9 | ES 冷层本地大容量 HDD | DD-04 |
| D10 | 压测试点 3× -> 上线 10× | DD-10 |
| D11 | Agent 红线统一基线 CPU<1%/内存<128M + 重负载白名单 | DD-13 |
| D12 | CMDB 分钟级同步 + 缓存 5m + 失效默认拒绝 | DD-09 |
| D13 | 混沌预发每迭代 + 生产季度级 | DD-11 |

---

## 7. 编码就绪状态（2026-07-20 二次核查）

- **编码前硬阻塞已全部解除**：前端框架（D1 Vue3）、TraceID 位数（D7 补正）均闭合。
- **编码前需推进项**：P0② API 契约字段级（OpenAPI3.0 + .proto，地基 change 产出）；P0③ 四个外部系统接口规范对齐（CMDB Q4 / 统一监控告警 Q5 等）；P0④ 试点 10 套/300 工程名单。
- **非阻塞 flag（不影响整体进度，遇到就地处理）**：F7 多活/集中名实、F8 采样↔存储隐性依赖、F9 《详细设计阶段任务计划》引用悬空、F11 infra 落库一期/二期口径、F12 `npm_online_performe` 拼写、F13 ES 字段（db 缺 env / event message-title 类型）。

详见 memory `coding-readiness-gaps`。
