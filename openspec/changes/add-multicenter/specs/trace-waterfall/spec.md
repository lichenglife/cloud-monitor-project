## MODIFIED Requirements

### Requirement: 瀑布图跨中心拼接展示
trace-waterfall（M3）MUST 扩展支持跨中心 Span 拼接展示（同 trace_id 多中心重组 + 因果/时间排序），Span MUST 标注来源中心（颜色/标签），跨中心缺失 MUST 标注 CROSS_DC_MISSING。

#### Scenario: 瀑布图跨中心 Span 标注来源
- **WHEN** 瀑布图含跨中心 Span
- **THEN** 按 center 标注来源（如紫色标注跨中心），缺失部分标注 CROSS_DC_MISSING
