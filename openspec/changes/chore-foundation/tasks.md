# chore-foundation 实现任务清单

> 粒度：14 组件各一组（13 后端/前端 + testkit）+ 跨服务基础模块 + 契约 + 部署 + 验证。每组件脚手架须可独立启动运行（design.md D7）。依赖版本见 design.md「依赖版本锁定」。横切规范见 specs/conventions。

## 0. 跨服务基础模块（统一设计，先于各组件落地）

- [ ] 0.1 设计并声明横切规范（specs/conventions：日志/错误/接口/代码/文档/提交/协议 gRPC+HTTP）
- [ ] 0.2 Java 统一基础 starter 模块（全局异常处理 @ControllerAdvice + 错误码枚举 DD-01 §7 + 统一响应体 + 日志 MDC trace_id + 健康检查 + gRPC 拦截器：鉴权/日志/恢复/重试）
- [ ] 0.3 Go 统一基础库模块（pkg/apperror 错误码 + pkg/log zap 结构化 + pkg/grpc middleware 拦截器 + 统一响应 + 健康检查 + config viper）
- [ ] 0.4 Repository 抽象接口（ES/MySQL/Redis 同接口，LSP/DIP）+ ES/MySQL/Redis 各一适配实现骨架
- [ ] 0.5 统一配置加载（Java Spring Cloud Config / Go viper，ConfigMap 静态 + config-center 动态双轨接入点）
- [ ] 0.6 统一可观测性（trace_id 贯穿 MDC/context + Micrometer/OTel Metrics 暴露 /metrics + Prometheus 黄金指标）
- [ ] 0.7 统一 /health（Liveness/Readiness）端点规范，依赖异常返回 degraded
- [ ] 0.8 统一 Dockerfile 基线（Java multi-stage JRE17；Go multi-stage distroless；前端 Nginx alpine）
- [ ] 0.9 统一 Makefile/mvn 脚手架命令（run/test/build/lint/docker/helm）
- [ ] 0.10 统一 .editorconfig / .gitignore / LICENSE

## 1. tracing-gateway（Java，API 网关：Spring Cloud Gateway，JWT 认证/限流(2000/100 QPS)/路由/审计）

- [ ] 1.1 从 Java SpringBoot 3.2 starter 模板派生工程（包 com.lichenglife.npm.trace.<service>，分层 controller/service/repository/dto）
- [ ] 1.2 接入统一基础模块 0.2（异常处理/错误码/响应体/日志 MDC/gRPC 拦截器）
- [ ] 1.3 配置加载（application.yml + ConfigMap + config-center 动态配置接入，0.5）
- [ ] 1.4 结构化日志（Logback + logstash-encoder JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 1.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 1.6 REST 端点骨架（对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页约定）
- [ ] 1.7 Repository 接入（依职责接 ES/MySQL/Redis，0.4）
- [ ] 1.8 /health（Liveness/Readiness）+ /metrics（Prometheus）
- [ ] 1.9 单测骨架（JUnit5 + Mockito + JaCoCo，覆盖率 ≥70%）+ TestContainers 本地中间件
- [ ] 1.10 Dockerfile + Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 1.11 独立启动验证（mvn spring-boot:run，/health 200，可接收 gRPC/HTTP 请求）
- [ ] 1.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 2. auth-service（Java，认证服务：Spring Security，登录/RBAC/JWT/CMDB 权限同步(D12)）

- [ ] 2.1 从 Java SpringBoot 3.2 starter 模板派生工程（包 com.lichenglife.npm.trace.<service>，分层 controller/service/repository/dto）
- [ ] 2.2 接入统一基础模块 0.2（异常处理/错误码/响应体/日志 MDC/gRPC 拦截器）
- [ ] 2.3 配置加载（application.yml + ConfigMap + config-center 动态配置接入，0.5）
- [ ] 2.4 结构化日志（Logback + logstash-encoder JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 2.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 2.6 REST 端点骨架（对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页约定）
- [ ] 2.7 Repository 接入（依职责接 ES/MySQL/Redis，0.4）
- [ ] 2.8 /health（Liveness/Readiness）+ /metrics（Prometheus）
- [ ] 2.9 单测骨架（JUnit5 + Mockito + JaCoCo，覆盖率 ≥70%）+ TestContainers 本地中间件
- [ ] 2.10 Dockerfile + Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 2.11 独立启动验证（mvn spring-boot:run，/health 200，可接收 gRPC/HTTP 请求）
- [ ] 2.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 3. alert-service（Java，告警服务：规则引擎(流式+滑动窗口)/静态阈值一期/降噪/Oncall）

- [ ] 3.1 从 Java SpringBoot 3.2 starter 模板派生工程（包 com.lichenglife.npm.trace.<service>，分层 controller/service/repository/dto）
- [ ] 3.2 接入统一基础模块 0.2（异常处理/错误码/响应体/日志 MDC/gRPC 拦截器）
- [ ] 3.3 配置加载（application.yml + ConfigMap + config-center 动态配置接入，0.5）
- [ ] 3.4 结构化日志（Logback + logstash-encoder JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 3.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 3.6 REST 端点骨架（对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页约定）
- [ ] 3.7 Repository 接入（依职责接 ES/MySQL/Redis，0.4）
- [ ] 3.8 /health（Liveness/Readiness）+ /metrics（Prometheus）
- [ ] 3.9 单测骨架（JUnit5 + Mockito + JaCoCo，覆盖率 ≥70%）+ TestContainers 本地中间件
- [ ] 3.10 Dockerfile + Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 3.11 独立启动验证（mvn spring-boot:run，/health 200，可接收 gRPC/HTTP 请求）
- [ ] 3.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 4. scenario-service（Java，业务场景服务：URL 标签 CRUD/聚合）

- [ ] 4.1 从 Java SpringBoot 3.2 starter 模板派生工程（包 com.lichenglife.npm.trace.<service>，分层 controller/service/repository/dto）
- [ ] 4.2 接入统一基础模块 0.2（异常处理/错误码/响应体/日志 MDC/gRPC 拦截器）
- [ ] 4.3 配置加载（application.yml + ConfigMap + config-center 动态配置接入，0.5）
- [ ] 4.4 结构化日志（Logback + logstash-encoder JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 4.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 4.6 REST 端点骨架（对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页约定）
- [ ] 4.7 Repository 接入（依职责接 ES/MySQL/Redis，0.4）
- [ ] 4.8 /health（Liveness/Readiness）+ /metrics（Prometheus）
- [ ] 4.9 单测骨架（JUnit5 + Mockito + JaCoCo，覆盖率 ≥70%）+ TestContainers 本地中间件
- [ ] 4.10 Dockerfile + Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 4.11 独立启动验证（mvn spring-boot:run，/health 200，可接收 gRPC/HTTP 请求）
- [ ] 4.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 5. audit-service（Java，审计服务：操作审计 append-only/查询/归档）

- [ ] 5.1 从 Java SpringBoot 3.2 starter 模板派生工程（包 com.lichenglife.npm.trace.<service>，分层 controller/service/repository/dto）
- [ ] 5.2 接入统一基础模块 0.2（异常处理/错误码/响应体/日志 MDC/gRPC 拦截器）
- [ ] 5.3 配置加载（application.yml + ConfigMap + config-center 动态配置接入，0.5）
- [ ] 5.4 结构化日志（Logback + logstash-encoder JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 5.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 5.6 REST 端点骨架（对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页约定）
- [ ] 5.7 Repository 接入（依职责接 ES/MySQL/Redis，0.4）
- [ ] 5.8 /health（Liveness/Readiness）+ /metrics（Prometheus）
- [ ] 5.9 单测骨架（JUnit5 + Mockito + JaCoCo，覆盖率 ≥70%）+ TestContainers 本地中间件
- [ ] 5.10 Dockerfile + Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 5.11 独立启动验证（mvn spring-boot:run，/health 200，可接收 gRPC/HTTP 请求）
- [ ] 5.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 6. otel-collector（Go，OTel 定制 Collector：OTLP 接收/协议转换/脱敏/去重/导出 Kafka（各中心 biz））

- [ ] 6.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 6.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 6.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 6.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 6.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 6.6 接入端点骨架（OTLP receiver / Kafka consumer，依职责）
- [ ] 6.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 6.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 6.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 6.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 6.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 6.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 7. ingestion-service（Go，数据写入服务：Kafka 消费->ES/MySQL/Redis 写入/幂等/降级）

- [ ] 7.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 7.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 7.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 7.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 7.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 7.6 接入端点骨架（OTLP receiver / Kafka consumer，依职责）
- [ ] 7.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 7.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 7.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 7.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 7.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 7.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 8. metric-aggr-service（Go，指标聚合服务：指标预聚合/TopN/JVM-资源-中间件聚合）

- [ ] 8.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 8.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 8.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 8.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 8.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 8.6 HTTP 端点骨架（gin/resty，对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页）
- [ ] 8.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 8.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 8.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 8.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 8.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 8.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 9. trace-query-service（Go，链路查询服务：TraceID 检索/瀑布图/代码栈/下钻/三域关联入口）

- [ ] 9.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 9.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 9.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 9.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 9.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 9.6 HTTP 端点骨架（gin/resty，对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页）
- [ ] 9.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 9.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 9.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 9.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 9.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 9.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 10. topology-service（Go，拓扑服务：依赖发现/强弱依赖/拓扑生成/跨中心合并/爆炸半径）

- [ ] 10.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 10.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 10.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 10.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 10.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 10.6 HTTP 端点骨架（gin/resty，对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页）
- [ ] 10.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 10.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 10.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 10.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 10.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 10.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 11. config-center（Go，配置中心：Agent 配置热下发/版本/Watch 推送 ≤30s（HA））

- [ ] 11.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 11.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 11.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 11.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 11.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 11.6 HTTP 端点骨架（gin/resty，对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页）
- [ ] 11.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 11.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 11.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 11.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 11.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 11.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 12. agent-registry（Go，Agent 注册服务：注册/握手令牌/心跳/台账/存活判定）

- [ ] 12.1 从 go-clean 模板派生工程（module github.com/lichenglife/cloud-monitor/<service>，cmd + internal/domain,usecase,repository,interfaces 分层）
- [ ] 12.2 接入统一基础库 0.3（apperror/log/grpc middleware/响应/健康检查/config）
- [ ] 12.3 配置加载（viper，ConfigMap + config-center 动态配置接入，0.5）
- [ ] 12.4 结构化日志（uber/zap JSON，含 timestamp/level/service/trace_id，PII 脱敏）
- [ ] 12.5 gRPC server/client stub 生成（.proto tracing.v1，对应服务 RPC）+ mTLS
- [ ] 12.6 HTTP 端点骨架（gin/resty，对应 DD-01 §3 模块接口，统一响应 + 错误码 + 分页）
- [ ] 12.7 Repository 接入（依职责接 ES/MySQL/Redis/Kafka，0.4）
- [ ] 12.8 /health（Liveness/Readiness）+ /metrics（Prometheus client_golang）
- [ ] 12.9 单测骨架（testify + testcontainers-go 本地中间件，覆盖率 ≥70%）
- [ ] 12.10 Dockerfile（multi-stage distroless）+ Helm Chart（Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 12.11 独立启动验证（make run，/health 200，可接收 gRPC/HTTP/OTLP 请求）
- [ ] 12.12 README（启动/构建/依赖/配置/健康检查，0 文档规范）

## 13. tracing-frontend（Vue3，前端：Vue3 SPA/Nginx 托管/Feature-based 分层）

- [ ] 13.1 Vite 5 + TS strict 项目初始化（路径别名 @/->src/，pnpm）
- [ ] 13.2 Feature-based 目录结构（app/features/shared/lib/types，DD-15 §3）
- [ ] 13.3 状态管理（TanStack Query 服务端 + Pinia 客户端，禁 Options API 混写）
- [ ] 13.4 Vue Router v4 全 lazy() + Vite manualChunks 拆 g6/echarts 独立 chunk
- [ ] 13.5 UI 库（Element Plus）+ 设计令牌双主题（暗色大屏/亮色后台）骨架
- [ ] 13.6 ESLint+Prettier+Stylelint+Husky+lint-staged 卡点
- [ ] 13.7 测试基线（Vitest+Vue Test Utils ≥70% + Playwright E2E 骨架）
- [ ] 13.8 RealtimeService 统一层骨架（WS + SSE 降级 + 轮询回退 + 心跳重连）
- [ ] 13.9 错误态/权限态拦截器（401->登录/403->无权限/429->限流/500->traceId）
- [ ] 13.10 契约类型生成（openapi-typescript 由 OpenAPI 生成 TS 类型）
- [ ] 13.11 Nginx 托管 + Dockerfile（alpine）+ Helm Chart
- [ ] 13.12 独立启动验证（pnpm dev，可渲染，Mock 联调）
- [ ] 13.13 README（启动/构建/依赖/配置）

## 14. API 契约层（OpenAPI 3.0 + gRPC .proto）

- [ ] 14.1 OpenAPI 3.0 主文件（/api/v1 基址、统一响应 {code,message,data,traceId}、错误码枚举 DD-01 §7）
- [ ] 14.2 按模块深化 REST 字段级 request/response（管理平台/采集/监控/全链路/APM/基建/日志/告警，DD-01 §3）
- [ ] 14.3 gRPC .proto（package tracing.v1，14 服务 RPC：PushSpan/PushMetric/PushLog/GetAgentConfig/PushAgentHeartbeat/PushUpgrade/SaveTrace/QueryTrace/CalcTopology/Evaluate/EmitAlert/Watch/Publish/VerifyToken/CheckPerm，DD-01 §5）
- [ ] 14.4 契约生成：前端 openapi-typescript；Java SpringDoc + oapi-codegen；Go oapi-codegen + protoc-gen-go-grpc
- [ ] 14.5 CI 门禁：openapi diff（无破坏性）+ buf breaking（proto 兼容）
- [ ] 14.6 Mock 约定：MSW/本地 mock 严格按契约，外部系统接口 mock 占位
- [ ] 14.7 Avro Schema Registry 初始（Span/Metric v1 字段全集注册，DD-14 §5）

## 15. CI/CD 与部署模板

- [ ] 15.1 Jenkinsfile/GitLab CI 模板（lint->type-check->unit->build->preview，PR 跑 Playwright/兼容性）
- [ ] 15.2 Avro Schema Registry + buf 校验集成 CI
- [ ] 15.3 Helm Chart 骨架（namespace tracing-system，Deployment/HPA/PDB/Service/ConfigMap/Secret）
- [ ] 15.4 mTLS cert-manager 初始化（集群级 issuer）
- [ ] 15.5 dmz 代理（HAProxy/Nginx）Deployment + 转发 biz Collector 配置（决议 A）
- [ ] 15.6 金丝雀发布 + 一键回滚脚本（1%->10%->100%，指标异常触发）
- [ ] 15.7 Git 分支保护规则（main 保护，PR≥1 Review + CI 全绿入 develop）
- [ ] 15.8 Prometheus + Grafana 自监控部署（Dogfooding 黄金指标）

## 16. 测试铺底组件 npm-trace-testkit 骨架

- [ ] 16.1 SpringBoot 3 + Java 17 项目初始化（入 DD-13 兼容矩阵）
- [ ] 16.2 SkyWalking Agent 注入占位（先 OTel SDK 直连 Collector，add-collection 后切 SkyWalking）
- [ ] 16.3 上报通路连通：Agent->Collector(biz/dmz 双路)->Kafka(上海 DC1)->聚合->ES 6 索引
- [ ] 16.4 S1-S8 八 scenario 骨架（db/mq/http/error/fault/log/deeptrace/resource），controller/config 动态开关
- [ ] 16.5 可编排参数（QPS/错误率/慢调用比例/Trace 深度/跨中心）配置接口
- [ ] 16.6 部署清单（预发/试点）+ 编排配置模板 + 铺底数据校验脚本

## 17. 验证

- [ ] 17.1 14 组件全部可独立启动（/health 200，可接收请求，不依赖其他业务组件）
- [ ] 17.2 契约生成全链路验证（OpenAPI->TS / proto->Java+Go stub 一致）
- [ ] 17.3 横切规范落地验证（日志含 trace_id/错误码映射/统一响应/分页/鉴权拦截）
- [ ] 17.4 CI/CD 端到端跑通（PR->lint->test->build->preview）
- [ ] 17.5 testkit 骨架产出链路可被 ES 检索（traceId 串联、瀑布图可下钻）
- [ ] 17.6 Helm 部署预发 K8s，HPA/PDB/mTLS 生效
- [ ] 17.7 依赖版本一致性核查（所有组件依赖版本 = design.md 锁定版本）

