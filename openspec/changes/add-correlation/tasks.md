# add-correlation（M6）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-trace-query 瀑布图入口 + add-storage-ingestion npm_online_metric。日志平台 SDK 改造跨团队（议题 C），本 change 提供 SDK 改造规格 + mock 联调。
> **涉及组件**：trace-query-service(Go) 关联查询 / 前端 traces 关联面板 / 日志平台 SDK（跨团队）。设计依据：DD-03 §7/§9.4 / DD-01 §3.7 / DD-14 / 决议 C / DD-17 §6.6。

## 阶段 1：日志平台 SDK 改造（决议 C，跨团队）

### 1.1 SDK 复用 Agent TraceID
- [ ] 1.1.1 [RED] SDK 单测：检测 SkyWalking ContextManager 存在->取 Agent traceId；不存在->降级自生成；同一请求日志 traceId = 链路 traceId
- [ ] 1.1.2 [GREEN] 日志平台 SDK 改造（复用 MDC TraceID，决议 C，跨团队交付）
- [ ] 1.1.3 [验收] 接入 Agent 应用日志 traceId = 链路 traceId；无 Agent 降级兼容

### 1.2 日志平台 trace_id 索引
- [ ] 1.2.1 [RED] 索引单测：日志平台侧 trace_id 倒排索引可按 traceId 检索
- [ ] 1.2.2 [GREEN] 日志平台建 trace_id 索引（跨团队）
- [ ] 1.2.3 [验收] 按 traceId 检索日志命中

## 阶段 2：Trace×Metric×Log 关联查询（Go）

### 2.1 Trace×Metric 关联
- [ ] 2.1.1 [RED] Metric 关联单测：同 service+时间窗[start-5min,end+1min]查 npm_online_metric 返回指标曲线
- [ ] 2.1.2 [GREEN] 实现 GET /traces/{traceId}/metrics（读 npm_online_metric，时间窗可配）
- [ ] 2.1.3 [验收] 指标曲线与 Trace 时间对齐；覆盖率 ≥80%

### 2.2 Trace×Log 关联
- [ ] 2.2.1 [RED] Log 关联单测：按 traceId 从日志平台拉取日志条目（GET /logs?traceId）；级别筛选；日志->Trace 反向定位（GET /logs/{logId}/trace）
- [ ] 2.2.2 [RED] 限频单测：同 TraceID 每分钟 ≤20 次
- [ ] 2.2.3 [RED] 缓存单测：Redis trace:logs:cache(120s TTL) 命中/失效
- [ ] 2.2.4 [GREEN] 实现 GET /traces/{traceId}/logs + /logs/{logId}/trace（日志平台桥接 + 限频 + 缓存）
- [ ] 2.2.5 [验收] 按 traceId 关联日志 + 反向定位；覆盖率 ≥80%

### 2.3 关联超时降级
- [ ] 2.3.1 [RED] 降级单测：日志/指标接口超时->'关联数据暂不可用'，不影响链路
- [ ] 2.3.2 [GREEN] 实现关联超时降级
- [ ] 2.3.3 [验收] 超时不中断链路；覆盖率 ≥80%

### 2.4 日志 PII 展示脱敏
- [ ] 2.4.1 [RED] 脱敏单测：日志展示侧手机号/身份证/银行卡掩码
- [ ] 2.4.2 [GREEN] 实现日志展示 PII 脱敏（DD-14 正则）
- [ ] 2.4.3 [验收] PII 掩码展示；覆盖率 ≥80%

## 阶段 3：前端三域关联面板（Vue3）

- [ ] 3.1 [RED] Metric 面板单测：时间窗指标曲线（ECharts）渲染
- [ ] 3.2 [GREEN] 实现 Metric 关联面板（ECharts）
- [ ] 3.3 [RED] Log 面板单测：日志侧栏（级别筛选+展开详情+日志->Trace 跳转）+ 日志平台可用性状态指示
- [ ] 3.4 [GREEN] 实现 Log 关联侧栏 + 状态指示
- [ ] 3.5 [RED] 入口接通单测：M3 瀑布图'关联指标/关联日志'按钮接通本面板
- [ ] 3.6 [GREEN] 接通瀑布图关联入口（替换 M3 占位）
- [ ] 3.7 [RED] 降级态单测：关联平台不可用->'关联数据暂不可用'状态
- [ ] 3.8 [GREEN] 实现关联降级态
- [ ] 3.9 [验收] 三域关联面板 + 入口接通 + 降级；覆盖率 ≥70%

## 阶段 4：跨组件联调验收（Gate，DD-17 §6.6）

- [ ] 4.1 [联调] M1 Agent MDC 注入 -> 日志平台 SDK 复用 -> 日志 traceId = 链路 traceId
- [ ] 4.2 [联调] M3 瀑布图入口 -> M6 关联面板接通
- [ ] 4.3 [联调] Trace×Metric 时间窗指标曲线
- [ ] 4.4 [联调] Trace×Log 按 traceId 关联 + 反向定位
- [ ] 4.5 [联调] 关联超时降级 + 日志 PII 脱敏
- [ ] 4.6 [验收] DD-17 §6.6：全文检索/TraceID关联/反向定位/脱敏/三域关联/降级
- [ ] 4.7 [验收] 跨平台跳转排查次数下降 ≥70%（PRD 成功度量）；覆盖率 ≥80%（后端）/≥70%（前端）

