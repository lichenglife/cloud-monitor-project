## ADDED Requirements

### Requirement: 后端双语言脚手架
Java（SpringBoot 3.2 + SpringCloud 2023，JDK17）与 Go 1.22 工程模板 MUST 可运行，含统一全局异常处理（映射 DD-01 §7 错误码）、Repository 抽象（ES/MySQL 同接口，LSP/DIP）、config-center 客户端接入点、可观测性打点（traceId 贯穿日志）、`/health` 探活端点。

#### Scenario: 新服务基于脚手架创建
- **WHEN** 新建一个微服务（如 topology-service）
- **THEN** 从脚手架模板派生，开箱即有异常处理/错误码/健康检查/可观测性，开发者只补业务逻辑

#### Scenario: 存储切换不影响业务层
- **WHEN** trace 索引从单索引重构为按天 Rollover
- **THEN** 因业务层依赖 Repository 抽象，存储适配层变更不影响 trace-query-service 业务逻辑

### Requirement: 前端 Vue3 脚手架与工程化基线
前端 MUST 为 Vue3 + `<script setup>` + Composition API + Pinia + Vue Router v4 + Vite 5 + TS strict + TanStack Query + Element Plus/Ant Design Vue + @antv/g6 v5 + ECharts 5，Feature-based 分层（`app/features/shared/lib/types`），含 ESLint+Prettier+Stylelint+Husky+lint-staged，契约类型由 OpenAPI 生成。

#### Scenario: 服务端状态统一走 Query
- **WHEN** 页面请求 Trace/指标/拓扑数据
- **THEN** 经 TanStack Query 统一管理（staleTime/缓存/重取），禁手写 fetch 缓存，实时推送经 `queryClient.setQueryData` 就地更新

#### Scenario: 重型可视化分包
- **WHEN** 用户首次进入拓扑页或 APM 页
- **THEN** g6/echarts 作为独立 chunk 按需加载，不拖累首屏（主包 <300KB gzip，FCP<1.5s）

### Requirement: CI/CD 流水线模板
MUST 提供 Jenkinsfile/GitLab CI 模板，阶段 lint->type-check->unit->build->preview，PR 跑 Playwright（前端）/兼容性（DD-13），Schema 变更走 Avro Registry 版本校验，金丝雀发布（1%->10%->100%）支持一键回滚。

#### Scenario: PR 提交触发门禁
- **WHEN** 开发提交 PR 到 develop
- **THEN** CI 跑 lint/type-check/unit/build，全绿方可合入；前端 PR 额外跑 Playwright 关键链路
