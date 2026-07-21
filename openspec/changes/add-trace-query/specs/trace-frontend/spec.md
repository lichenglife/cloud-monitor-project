## ADDED Requirements

### Requirement: 链路检索页与瀑布图详情页
前端 MUST 实现链路检索页（搜索栏 TraceID 精确 + 组合筛选 + 时间范围 + 分页）与瀑布图详情页（横轴时间、纵向每行 Span、颜色编码、点击下钻、完整性标签、三域关联入口按钮占位），MUST 用 Canvas 虚拟化 + Web Worker 建树保障 P95<1s 体感。

#### Scenario: 瀑布图大数据不卡顿
- **WHEN** 用户打开含 1 万 Span 的 Trace 详情
- **THEN** Web Worker 建树 + Canvas 虚拟化仅渲染可视窗，主线程不卡顿，交互流畅

### Requirement: 状态覆盖与错误态
前端 MUST 覆盖加载/空/异常/无权限/延迟/降级六态，错误态按 DD-01 §7 错误码映射（401->登录/403->无权限/429->限流/500->traceId）。

#### Scenario: 未找到 Trace 友好提示
- **WHEN** 用户查询不存在的 TraceID
- **THEN** 返回 404 + 友好提示"未找到匹配的链路数据，请检查 TraceID"
