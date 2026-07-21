# add-trace-query（M3）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-storage-ingestion 的 ES trace_npm_jaeger + trace-query-base gRPC 就绪。
> **涉及组件**：trace-query-service(Go) / 前端 traces 页(Vue3)。设计依据：DD-03 §4 / DD-01 §3.4 / DD-03 §9.4 / DD-15 §7-8 / DD-17 §6.3。

## 阶段 1：trace-query-service 列表与筛选（Go）

### 1.1 列表多条件筛选
- [ ] 1.1.1 [RED] 筛选单测：service+status+time+duration+httpStatus bool query 组合；分页 page/pageSize->total/items
- [ ] 1.1.2 [GREEN] 实现 GET /traces 列表查询 gRPC（ES bool query + 分页）
- [ ] 1.1.3 [验收] 多条件组合筛选结果与 ES 一致，分页正确；覆盖率 ≥80%

### 1.2 错误/慢调用分析
- [ ] 1.2.1 [RED] 错误慢分析单测：按错误码/慢>2s 阈值筛选 + 按耗时/错误排序
- [ ] 1.2.2 [GREEN] 实现 GET /traces/error-analysis
- [ ] 1.2.3 [验收] 慢调用降序排列正确；覆盖率 ≥80%

## 阶段 2：瀑布图详情与下钻（Go）

### 2.1 瀑布图组装
- [ ] 2.1.1 [RED] 组装单测：按 parent_span_id 组装 span 树；每 span 含 service/operation/duration/status/耗时占比/颜色编码（正常蓝/慢橙/错误红/跨中心紫）
- [ ] 2.1.2 [GREEN] 实现 GET /traces/{traceId} 瀑布图组装（树形 + 耗时占比 + 颜色）
- [ ] 2.1.3 [验收] 瀑布图数据结构与树形一致；覆盖率 ≥80%

### 2.2 方法/SQL/代码栈下钻
- [ ] 2.2.1 [RED] 下钻单测：SQL span->SQL 语句+执行计划+耗时；方法 span->方法逐层调用+代码栈；异常 span->堆栈红色高亮
- [ ] 2.2.2 [GREEN] 实现下钻详情查询（code_stack/sql_statement/method_detail 字段）
- [ ] 2.2.3 [验收] 下钻展示方法/SQL/代码栈正确；覆盖率 ≥80%

### 2.3 链路完整性标注
- [ ] 2.3.1 [RED] 完整性单测：parent-child 时间差异常->PARTIAL_BROKEN；跨中心 datacenter_id 缺失->CROSS_DC_MISSING；正常->COMPLETE
- [ ] 2.3.2 [GREEN] 实现完整性判断逻辑
- [ ] 2.3.3 [验收] 断裂/跨中心缺失显式标注不静默丢弃；覆盖率 ≥80%

### 2.4 超长 Trace 截断
- [ ] 2.4.1 [RED] 截断单测：采集 >50000 spans 硬截断+计数；展示 >10000 保留错误/慢 span+采样子集+truncated=true
- [ ] 2.4.2 [GREEN] 实现展示侧截断逻辑（采集侧在 add-collection 已做，此处校验）
- [ ] 2.4.3 [验收] 超长 Trace 截断+标记正确；覆盖率 ≥80%

## 阶段 3：前端链路检索页与瀑布图（Vue3）

### 3.1 链路检索页
- [ ] 3.1.1 [RED] 检索页单测：搜索栏（TraceID 精确+组合筛选）+时间范围+分页+列表渲染（TraceID/服务/耗时/Span数/状态/操作）
- [ ] 3.1.2 [GREEN] 实现 features/traces 检索页（TanStack Query + DataTable 虚拟滚动）
- [ ] 3.1.3 [验收] 筛选+分页+列表正确；覆盖率 ≥70%

### 3.2 瀑布图详情页
- [ ] 3.2.1 [RED] 瀑布图单测：Canvas 渲染 span 时序/耗时条/父子嵌套/颜色编码；点击 span 弹右侧面板（基本信息+堆栈+SQL+[关联指标][关联日志]占位按钮）
- [ ] 3.2.2 [RED] Worker 建树单测：5 万 span 在 Web Worker 建树 postMessage 主线程不卡顿
- [ ] 3.2.3 [RED] 虚拟化单测：仅渲染可视时间窗 span，滚动动态加载
- [ ] 3.2.4 [GREEN] 实现 WaterfallChart（Canvas + Web Worker 建树 + 虚拟化）
- [ ] 3.2.5 [验收] 1 万 span 详情页交互流畅（P95<1s 体感）；覆盖率 ≥70%

### 3.3 状态覆盖与错误态
- [ ] 3.3.1 [RED] 状态单测：加载/空/异常(401/403/429/500)/无权限/延迟/降级六态
- [ ] 3.3.2 [GREEN] 实现状态覆盖 + 错误码拦截器（DD-01 §7）
- [ ] 3.3.3 [验收] 404 TraceID 友好提示；六态正确；覆盖率 ≥70%

## 阶段 4：跨组件联调验收（Gate，DD-17 §6.3 M3）

- [ ] 4.1 [联调] 前端检索页 -> trace-query 列表 -> ES 筛选返回
- [ ] 4.2 [联调] 点击 Trace -> 瀑布图详情 -> Span 下钻方法/SQL/代码栈
- [ ] 4.3 [联调] 跨中心 Trace -> 完整性标注（断裂/跨中心缺失）
- [ ] 4.4 [联调] 超长 Trace -> 截断+truncated 标记
- [ ] 4.5 [联调] 三域关联入口按钮占位（M6 落地实际关联）
- [ ] 4.6 [验收] DD-17 §6.3 M3：列表筛选/瀑布图/代码栈/下钻方法SQL/三域入口/超长截断/脱敏/完整性标注
- [ ] 4.7 [验收] 压测 3×：查询 P95<1s；覆盖率 ≥80%（后端）/≥70%（前端）

