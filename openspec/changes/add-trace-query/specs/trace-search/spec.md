## ADDED Requirements

### Requirement: Trace 列表多条件筛选
trace-query-service MUST 支持 GET /traces 按 service/status/time/duration/httpStatus 多条件筛选（ES bool query 组合），MUST 分页返回（默认 20/页），MUST 支持 GET /traces/error-analysis 按错误码/慢调用阈值筛选与排序。

#### Scenario: 多条件组合筛选
- **WHEN** 用户按 service=payment AND status=error AND duration>2s 筛选
- **THEN** ES bool query 组合过滤返回匹配 Trace 列表，分页正确

### Requirement: 错误/慢调用分析
MUST 支持按错误码/慢调用阈值（>2s 可配）筛选 Trace 并按耗时/错误排序。

#### Scenario: 慢调用排序
- **WHEN** 用户查询慢调用并按耗时降序
- **THEN** 返回 duration>2s 的 Trace 按耗时降序排列
