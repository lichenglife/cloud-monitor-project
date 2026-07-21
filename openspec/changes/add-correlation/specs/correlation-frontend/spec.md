## ADDED Requirements

### Requirement: 三域关联面板
前端 MUST 在链路详情页实现三域关联面板（Tab：Metric 指标曲线 / Log 日志侧栏），Metric 面板展示时间窗内指标曲线（ECharts），Log 面板展示按 traceId 关联日志（级别筛选 + 展开详情 + 日志->Trace 跳转），MUST 接通 M3 瀑布图详情页的"关联指标/关联日志"入口按钮。

#### Scenario: 入口按钮接通关联面板
- **WHEN** 用户在瀑布图点击"关联日志"按钮
- **THEN** 侧栏加载按 traceId 关联的日志列表，无需跨系统跳转

### Requirement: 状态覆盖与降级
MUST 覆盖加载/空/异常/降级态，关联平台不可用时标注"关联数据暂不可用"，日志平台可用性状态指示。

#### Scenario: 日志平台不可用状态指示
- **WHEN** 日志平台不可达
- **THEN** 面板顶部显示"日志平台不可用"状态指示，不阻塞链路查看
