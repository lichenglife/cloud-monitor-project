## Why

一期 13 个微服务（Java+Go）+ 自研前端（Vue3）需并行编码，但当前仓库为全新空仓库，**无任何脚手架、无 API 契约字段级定义（DD-01 只有路径骨架，无完整 request/response，无 gRPC .proto）、无 CI/CD 模板、无 K8s/Helm 部署清单**。这是 P0② 阻塞项（见 memory `coding-readiness-gaps`）：没有契约地基，13 服务无法并行开发，前后端无法并行。

本 change 不交付任何业务功能，而是产出**编码地基**--契约层（OpenAPI 3.0 + gRPC .proto）、前后端脚手架、CI/CD 与部署模板--作为 M1~M17 所有功能模块 change 的前置依赖。先行落地，使后续模块 change 可纵向切片并行推进。

## What Changes

1. **API 契约层**：将 DD-01 §3 接口骨架 + DD-01 §5 gRPC 服务定义，深化为字段级 OpenAPI 3.0（REST `/api/v1`）+ gRPC `.proto`（14 服务内部调用），统一响应结构 `{code,message,data,traceId}` 与错误码（DD-01 §7）。契约作为前后端/服务间并行开发的唯一真源，配合 Mock（MSW/本地）。
2. **后端脚手架**：Java（SpringBoot 3.2 + SpringCloud 2023，JDK17）与 Go 1.22 双语言工程模板，含统一异常处理、错误码、Repository 抽象（LSP/DIP，ES/MySQL 同接口）、配置热更新接入、可观测性打点（traceId 贯穿）、健康检查 `/health`。
3. **前端脚手架**：Vue3 + `<script setup>` + Pinia + Vue Router v4 + Vite 5 + TS strict + TanStack Query + Element Plus/Ant Design Vue + @antv/g6 v5 + ECharts 5；Feature-based 分层（DD-15 §3）；ESLint+Prettier+Stylelint+Husky+lint-staged；契约类型由 OpenAPI 生成。
4. **CI/CD 模板**：Jenkinsfile/GitLab CI（lint->type-check->unit->build->preview），PR 跑 Playwright（前端）/ 兼容性（DD-13）；Schema 变更走 Avro Registry 版本校验（DD-14）。
5. **部署模板**：Helm Charts + K8s yaml（namespace `tracing-system`，Deployment/HPA/PDB/Service/ConfigMap/Secret），mTLS 证书初始化，Git 分支保护规则。
6. **测试铺底组件骨架**：`npm-trace-testkit`（Java SpringBoot 3，DD-17 §7）初始化，S1-S8 场景骨架（后续 add-collection 等模块逐步填充各 scenario），为集成/性能/混沌/验收测试提供真实链路数据。

## Capabilities

### New Capabilities
- `conventions`: 横切基础规范（日志/错误处理/接口/代码/文档/提交/组件访问协议 gRPC+HTTP），作为所有组件与后续 change 必须遵循的可验收约束
- `api-contracts`: 平台统一 API 契约层（REST OpenAPI 3.0 + gRPC .proto），14 服务内部调用与对外 REST 的字段级定义、错误码、统一响应结构、Mock 约定
- `app-scaffold`: 14 组件工程脚手架（5 Java 业务服务 + 1 Java testkit + 7 Go + 1 前端），每组件可独立启动运行，含统一异常处理/错误码/Repository 抽象/可观测性/健康检查/单测/容器化
- `deploy-templates`: Helm Charts + K8s 部署清单 + Git 分支模型与发布策略模板
- `test-kit`: Java 微服务测试铺底组件 `npm-trace-testkit` 骨架（S1-S8 场景占位，对接 SkyWalking Agent/Collector）

### Modified Capabilities
<!-- 无，本 change 为绿地地基 -->

## Impact

- **阻塞解除**：闭合 P0② API 契约字段级缺口；为 M1~M17 全部功能模块 change 提供契约与脚手架地基。
- **依赖**：本 change 无功能依赖，是生命周期第一步；后续模块 change 引用此处产出的契约与脚手架。
- **外部依赖**：暂无（外部系统接口规范 Q4/Q5 待对齐，本 change 用 mock 占位）。
- **对应 PRD 模块**：跨模块基础设施（非 M1-M17 任一），对应研发任务拆解 M0 准备与选型阶段（T-M1-1 技术选型落地的工程化部分）。
- **设计依据**：DD-01（接口契约/错误码/限流/gRPC）、DD-02（数据模型/Redis Key/Kafka Topic）、DD-11（工程规范/分支模型/提测标准）、DD-15（前端架构/目录/工程化基线）、DD-17（testkit 铺底组件）、系统设计说明书 §2.6/§2.7（服务拆分/技术栈）。
