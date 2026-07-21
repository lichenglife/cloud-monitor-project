## Context

绿地项目，仓库无任何代码/脚手架/契约。本 change 产出**编码地基与设计约束**，使后续 M1~M17 模块 change 可纵向切片并行推进。

- 系统设计说明书 §2.6 拆分 13 微服务 + 1 前端（共 14 组件），§2.7 定技术栈，DD-01 §3/§5 定接口骨架但字段级未定义。
- 语言划分（依 §2.6 硬标 + "Go 为主"原则）：**Java 5 个**（tracing-gateway / auth-service / alert-service / scenario-service / audit-service）+ **Go 8 个**（otel-collector / topology-service / ingestion-service / metric-aggr-service / trace-query-service / config-center / agent-registry + 前端 tracing-frontend 为 Vue3，不计后端语言）。
- 13 微服务 + 前端需并行编码，缺契约/脚手架/统一基础模块则无法纵向切片。

## Goals / Non-Goals

**Goals:**
- 初始化全部 14 组件工程脚手架，每个组件可独立启动运行（`/health` 200、可接收请求）
- 锁定全部技术栈依赖版本（§依赖版本锁定）
- 统一设计并声明横切基础模块：日志/错误处理/接口规范/代码规范/文档规范/提交规范/组件访问协议(gRPC+HTTP)，落地为可验收的 `conventions` capability spec
- 产出字段级 API 契约（OpenAPI 3.0 REST + gRPC .proto，14 服务）
- 产出 CI/CD + Helm/K8s 部署模板
- 初始化 `npm-trace-testkit` 铺底组件骨架

**Non-Goals:**
- 不实现任何业务功能（采集/存储/查询/拓扑/告警等由 M1~M17 各 change 承担）
- 不对齐外部系统真实接口（CMDB Q4/统一监控告警 Q5 用 mock 占位）
- 不做性能压测（D10 在各模块提测阶段）
- 不含 SkyWalking/OTel Agent 二次开发（Agent 治理在 add-collection）

## 架构设计（设计约束，所有后续 change 须遵循）

### 组件分层与数据流

```
┌──────────────────────────── 展示层 ────────────────────────────┐
│  tracing-frontend (Vue3 SPA, Nginx 托管, 上海 DC1 biz)          │
└───────────────────────────────┬────────────────────────────────┘
                                │ HTTPS REST /api/v1 + WebSocket
┌───────────────────────────────▼────────────────────────────────┐
│  tracing-gateway (Java, Spring Cloud Gateway)                   │
│  统一入口 / 认证鉴权(JWT) / 限流(2000/100 QPS) / 路由 / 审计      │
└──┬─────────────────────────────────────────────────────────────┘
   │ gRPC (内部，mTLS)
   ├──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
   ▼              ▼              ▼              ▼              ▼              ▼
┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│trace-  │  │metric-   │  │topology- │  │alert-    │  │config-   │  │scenario- │
│query   │  │aggr      │  │service   │  │service   │  │center    │  │service   │
│(Go)    │  │(Go)      │  │(Go)      │  │(Java)    │  │(Go)      │  │(Java)    │
└───┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────┘
    │            │             │             │             │
    │   ┌────────┴────┐  ┌─────┴────┐  ┌─────┴────┐  ┌─────┴────┐
    │   ▼             ▼  ▼          ▼  ▼          ▼  ▼          
┌───┴──────────────────────────────────────────────────────────┐
│  ingestion-service (Go, Kafka 消费 -> ES/MySQL/Redis 写入)     │
└───┬──────────────────────────────────────────────────────────┘
    │ Kafka (上海 DC1): otel-spans/metrics/logs/infra-metrics
┌───┴──────────────────────────────────────────────────────────┐
│  otel-collector (Go, OTel 定制) 各中心 biz 部署                │
│  Receiver(OTLP) -> Processor(协议转换/脱敏/去重) -> Exporter   │
└───┬──────────────────────────────────────────────────────────┘
    │ OTLP gRPC:4317/HTTP:4318 (biz 直连 / dmz 代理转发)
┌───┴──────────────────────────────────────────────────────────┐
│  Agent 层: Java=SkyWalking / Go=OTel (K8S initContainer/VM)    │
└───────────────────────────────────────────────────────────────┘

横切服务（被各业务服务调用）:
  auth-service (Java, JWT/RBAC/CMDB 权限兜底)
  agent-registry (Go, Agent 注册/心跳/台账)
  audit-service (Java, 操作审计 append-only)
```

### 13 组件职责（设计约束，对应系统设计 §2.6 + §2.17 模块清单）

| 组件 | 语言 | 职责（边界，禁止越界） | 主要数据读写 |
|---|---|---|---|
| `tracing-gateway` | Java | 统一入口、JWT 认证、RBAC 鉴权、限流(集群2000/单用户100 QPS)、路由转发、审计埋点 | 读 Redis 会话/权限缓存；写 audit |
| `trace-query-service` | Go | TraceID 检索、瀑布图组装、代码栈、下钻方法/SQL、三域关联入口 | 读 ES trace_npm_jaeger + npm_online_metric；读外部日志平台 |
| `metric-aggr-service` | Go | 指标接收预聚合、TopN 计算、JVM/资源/中间件指标聚合 | 写 ES npm_online_metric/db/performe/infra；读写 Redis 实时指标 |
| `topology-service` | Go | 依赖自动发现、强弱依赖计算、拓扑生成、跨中心合并、爆炸半径 | 写 ES npm_online_topology + MySQL t_topology_snapshot；读写 Redis 拓扑缓存 |
| `alert-service` | Java | 告警规则引擎(流式消费预聚合+滑动窗口)、静态阈值一期、降噪收敛、Oncall、事件落库 | 写 ES npm_online_event + MySQL t_alert_*；读 Redis 静默 |
| `config-center` | Go | Agent 配置热下发(采样率/开关/灰度)、版本管理、Watch 推送 ≤30s | 读 MySQL t_agent_config；写 Redis agent:config/config:version |
| `auth-service` | Java | 用户登录/登出/注销、RBAC、JWT 签发、CMDB 权限同步(分钟级+缓存5m+失效拒绝 D12) | 读写 MySQL t_user/t_role/t_permission；写 Redis session/perm |
| `agent-registry` | Go | Agent 注册/握手令牌/心跳/台账、元数据管理、存活判定 | 读写 MySQL t_agent_instance/t_agent_registration；写 Redis agent:heartbeat |
| `ingestion-service` | Go | Kafka 消费 -> 写 ES/MySQL/Redis、幂等、降级缓冲 | 写 ES 6 索引 + MySQL 元数据 + Redis 实时 |
| `scenario-service` | Java | 入口 URL 业务场景标签 CRUD、标签聚合 | 读写 MySQL t_biz_scenario；写 Redis biz:url:tag |
| `audit-service` | Java | 操作审计记录(append-only)、审计查询、归档 | 写 MySQL t_audit_log |
| `otel-collector` | Go | OTLP 接收、SkyWalking/OTel->OTel 转换、PII 脱敏、幂等去重、导出 Kafka、背压降级 | 写 Kafka(上海 DC1)；读写 Redis 去重 |
| `tracing-frontend` | Vue3 | 自研前端 SPA、Nginx 托管、Feature-based 分层 | 经 gateway 调后端 |

> **SRP 约束**：每个组件只承担上表职责，禁止跨域。如拓扑计算只在 topology-service（不在 Collector）；告警评估只在 alert-service（不在 metric-aggr）；审计只在 audit-service。跨组件需求数据走 gRPC 调用，不直连他库。

### 组件交互协议（设计约束）

- **外部用户 -> 前端 -> gateway**：HTTPS REST `/api/v1/*` + WebSocket（告警/大屏实时）。
- **gateway -> 后端服务**：gRPC（mTLS），gateway 做认证后透传用户身份（JWT claims）于 gRPC metadata。
- **服务间**：gRPC（mTLS），超时 1000ms，幂等 RPC 重试 2 次指数退避，非幂等不重试（DD-01 §9）。
- **Agent -> Collector**：OTLP gRPC:4317 / HTTP:4318（TLS 1.3 + 注册令牌，决议 A biz 直连/dmz 代理）。
- **Collector -> Kafka**：Avro + Schema Registry，分区按 trace_id hash。
- **服务 -> 存储**：经统一 Repository 抽象（LSP/DIP），ES/MySQL/Redis 各适配实现。
- **统一响应**：REST `{code,message,data,traceId}`；gRPC `UnifiedResponse{code,message,Any data,trace_id}`（DD-01 §7 错误码）。

### 部署约束（决议 A）

- 全部 14 组件 + 全部 DB 部署 **上海 DC1 biz 区**；各中心 biz 区额外部署 otel-collector；各中心 dmz 区部署代理（HAProxy/Nginx）转发至本中心 biz Collector。
- namespace `tracing-system`；每组件 Deployment + HPA（按 CPU/吞吐）+ PDB + Service；mTLS via cert-manager。

## 依赖版本锁定（设计约束，所有组件须严格遵循）

### 后端 Java（5 服务：tracing-gateway/auth-service/alert-service/scenario-service/audit-service）

| 依赖 | 版本 | 说明 |
|---|---|---|
| JDK | 17 (主) / 11 (兼容) | SpringBoot 3.x 需 JDK17 |
| Spring Boot | 3.2.x | 主框架 |
| Spring Cloud | 2023.0.x | 微服务治理（gateway/config） |
| Spring Cloud Gateway | 2023.0.x | tracing-gateway |
| Spring Security | 6.2.x | auth-service RBAC/JWT |
| gRPC Java | 1.62.x | 服务间 + protobuf |
| Protobuf Java | 3.25.x | .proto 生成 |
| Elasticsearch Java Client | 8.12.x | ES 读写 |
| MySQL Connector/J | 8.3.0 | MySQL 8.0 |
| Lettuce (Redis) | 6.3.x | Redis Cluster 客户端 |
| Kafka Client | 3.6.x | 与 Kafka 3.6 服务端对齐 |
| MyBatis-Plus | 3.5.x | ORM（参数化防注入） |
| Caffeine | 3.1.x | 本地缓存（多级缓存） |
| Lombok | 1.18.x | 样板代码 |
| JUnit 5 + Mockito | 5.10.x / 5.11.x | 单测（覆盖率 ≥70%/≥80%） |
| JaCoCo | 0.8.x | 覆盖率报告 |
| Logback + logstash-logback-encoder | 1.4.x / 7.4.x | 结构化日志（JSON） |

### 后端 Go（7 服务：otel-collector/ingestion/metric-aggr/trace-query/topology/config-center/agent-registry）

> 注：`npm-trace-testkit` 铺底组件为 Java（SpringBoot 3 + Java 17），其依赖见上「后端 Java」节，不在此 Go 节。

| 依赖 | 版本 | 说明 |
|---|---|---|
| Go | 1.22+ | 工具链 |
| OpenTelemetry Collector | 0.90+ (v1) | otel-collector 定制基线（cmd + builder） |
| OpenTelemetry Go SDK | 1.24.x | 链路/指标（testkit 与 Go 服务埋点） |
| gRPC Go | 1.62.x | 服务间 + protobuf |
| Protobuf Go | 1.33.x | .proto 生成 |
| grpc-ecosystem/go-grpc-middleware | 2.x | 拦截器（鉴权/日志/恢复/重试） |
| go-resty / gin | 1.12.x (resty) | HTTP 客户端/轻量路由 |
| elastic/go-elasticsearch | 8.12.x | ES 客户端 |
| go-sql-driver/mysql | 1.8.x | MySQL 驱动 |
| redis/go-redis | 9.x | Redis Cluster 客户端 |
| segmentio/kafka-go 或 IBM/sarama | 0.4.x (kafka-go) | Kafka 客户端 |
| spf13/viper | 1.18.x | 配置（ConfigMap + config-center） |
| uber/zap | 1.27.x | 结构化日志 |
| prometheus/client_golang | 1.19.x | /metrics（黄金指标 Dogfooding） |
| testify | 1.9.x | 单测 |
| connaisseur/uuid 或 google/uuid | 1.6.x | TraceID 等唯一 ID |

### 前端 tracing-frontend

| 依赖 | 版本 | 说明 |
|---|---|---|
| Node | 20 LTS | 构建 |
| Vue | 3.4+ | 框架（D1） |
| Vite | 5.x | 构建 |
| TypeScript | 5.4+ | strict:true |
| Pinia | 2.1+ | 客户端状态 |
| Vue Router | 4.3+ | 路由（lazy） |
| TanStack Query (Vue Query) | 5.x | 服务端状态 |
| Element Plus | 2.7+ | UI 库 |
| @antv/g6 | 5.x | 拓扑（WebGL，D2） |
| ECharts | 5.5+ | 图表 |
| Vitest | 1.6+ | 单测 |
| Playwright | 1.44+ | E2E |
| ESLint + Prettier + Stylelint | 9.x / 3.3+ / 16.x | 代码质量 |
| Husky + lint-staged | 9.x / 15.x | Git 钩子 |
| openapi-typescript | 7.x | OpenAPI -> TS 类型 |

### 中间件与基础设施（部署侧）

| 组件 | 版本 | 说明 |
|---|---|---|
| Kubernetes | 1.25 (主) / 1.21 (兼容) | 容器编排 |
| Helm | 3.14+ | K8s 包管理 |
| Docker | 24.x | 容器运行时 |
| Nginx | 1.24+ | 反向代理/LB/前端托管 |
| HAProxy | 2.6+ | dmz 代理 |
| Elasticsearch | 8.12+ | 链路检索（按天 Rollover，D9 本地 HDD 冷层） |
| MySQL | 8.0.35+ | 元数据（主从） |
| Redis | 7.2+ | 缓存（Cluster 3主3从） |
| Apache Kafka | 3.6+ | 管道（RF≥3 跨 AZ，D8） |
| Confluent Schema Registry | 7.6.x | Avro Schema 版本化 |
| cert-manager | 1.14+ | mTLS 证书 |
| Prometheus + Grafana | 2.51+ / 11.x | 平台自监控（Dogfooding） |
| Jenkins / GitLab CI | LTS / 17.x | CI/CD |

### 采集侧

| 组件 | 版本 | 说明 |
|---|---|---|
| SkyWalking Java Agent | 9.x / 10.x | Java 自动埋点（决议 E） |
| OpenTelemetry Java Agent | 1.x | 备选（决议 E） |
| OpenTelemetry Go Agent/SDK | 1.24.x | Go 手动埋点 |

> **版本约束规则**：所有组件依赖版本须与上表一致；升级须经评审并更新本表 + project.md。Avro Schema 与 .proto 走 Registry/buf 版本校验，破坏性变更升 v2。

## Decisions

### D1: 契约优先，OpenAPI 为 REST 真源、proto 为 gRPC 真源
REST 以 OpenAPI 3.0 定义，前端 TS 类型由 `openapi-typescript` 生成，Java 用 SpringDoc、Go 用 `oapi-codegen` 生成 server stub。gRPC 14 服务用 `.proto`（`tracing.v1`），Java/Go 各生成 stub。契约即门禁：`openapi diff` + `buf breaking`。

### D2: Repository 抽象遵循 LSP/DIP（DD-01 §5）
ES/MySQL/Redis 访问经统一 Repository 接口，各存储适配实现。业务层依赖抽象，存储切换不影响业务。

### D3: 配置热更新双轨
静态配置 ConfigMap（viper 读取）；动态运行配置（采样率/开关/灰度）走 config-center，Watch 拉取 ≤30s 生效。

### D4: 前端 Feature-based + 服务端状态走 Query
目录 `app/features/shared/lib/types`；服务端状态走 TanStack Query；重型可视化路由级 lazy + manualChunks 分包。

### D5: testkit 独立 SpringBoot 应用旁路铺底
`npm-trace-testkit`（SpringBoot 3 + Java 17）独立部署，经 SkyWalking Agent 注入产生 OTLP，经 Collector(biz/dmz)->Kafka(上海 DC1)->聚合->ES，与真实接入一致。S1-S8 场景随各模块 change 填充。

### D6: 横切规范统一设计为 conventions capability（可验收）
日志/错误处理/接口规范/代码规范/文档规范/提交规范/组件访问协议(gRPC+HTTP) 统一设计并声明为 `specs/conventions/spec.md`，作为所有组件与后续 change 必须遵循的可验收约束。

### D7: 每组件可独立启动运行
每个组件脚手架须可独立 `make run` / `mvn spring-boot:run` 启动，`/health` 返回 200，可接收请求（依赖 TestContainers 起本地中间件），不依赖其他业务组件即可冒烟。

## Risks / Trade-offs

- **契约过早冻结**：缓解--版本化（/v1 + Avro Registry + buf），新增兼容，破坏升 v2 走 Deprecation。
- **双语言横切规范维护**：Java+Go 各一套日志/错误/拦截器实现。缓解--共享 .proto/错误码/响应结构，横切行为由 conventions spec 统一约束。
- **testkit 依赖 SkyWalking Agent**：缓解--骨架先 OTel SDK 直连 Collector，add-collection 落地后切 SkyWalking。
- **外部接口 mock 漂移**：缓解--mock 严格按 DD-01 §6 + 系统设计 §2.10 字段约定，对齐时 diff 校验。
- **14 组件脚手架工作量**：缓解--Go 组件共享统一 go-clean 模板，Java 共享 SpringBoot starter 模板，脚手架由模板派生。
