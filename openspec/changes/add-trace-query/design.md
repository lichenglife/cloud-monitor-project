## Context

M2 已落库 ES trace_npm_jaeger + npm_online_metric。M3 在 trace-query-service 上构建检索/瀑布/下钻能力，并交付前端链路检索页与详情页。设计见 DD-03 §4 + 系统设计 §3.3。

## Goals / Non-Goals

**Goals:**
- Trace 列表多条件筛选（service/status/time/duration/httpStatus）+ 错误/慢分析
- Trace 详情瀑布图（parent_span_id 树形）+ 代码栈 + 方法/SQL 下钻
- 链路完整性标注（断裂/跨中心缺失）+ 超长 Trace 截断（5万采/1万展）
- 前端瀑布图 Canvas 虚拟化 + Web Worker 建树（P95<1s 体感）

**Non-Goals:**
- 不实现三域关联面板（M6，本变更留入口按钮）
- 不实现火焰图（M14 APM）
- 不实现拓扑（M4）

## Decisions

### DM1: 瀑布图组装在后端，前端仅渲染
trace-query-service 按 parent_span_id 组装 span 树 + 计算耗时占比 + 判断完整性（gap/跨中心 datacenter_id 分布），前端仅渲染。超长 Trace 展示上限 1 万 spans，超出保留错误/慢 span + 采样子集（DD-03 §9.4）。

### DM2: 前端 Canvas 虚拟化 + Web Worker 建树
瀑布图用自研 Canvas + 虚拟化（仅渲染可视时间窗 Span），5 万 Span 建树在 Web Worker 完成 postMessage 传主线程（DD-15 §7/§8.2）。

### DM3: 链路完整性判断
检查 parent-child 时间差异常/Span 缺失 -> 标注 COMPLETE/PARTIAL_BROKEN/CROSS_DC_MISSING（PRD REQ-M3-W1，跨中心 datacenter_id 分布判断）。

## Risks / Trade-offs

- **超长 Trace 渲染卡顿**：缓解--展示上限 1 万 + Worker 建树 + Canvas 虚拟化。
- **ES 列表查询性能**：多条件 bool query。缓解--预聚合索引 + 分页（默认 20/页）。
