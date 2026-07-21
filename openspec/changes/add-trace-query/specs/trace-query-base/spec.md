## MODIFIED Requirements

### Requirement: TraceID 精确检索扩展为列表与详情
trace-query-service 的 TraceID 精确检索（M2 trace-query-base）MUST 扩展支持多条件列表筛选（bool query）、瀑布图详情组装（parent_span_id 树）、方法/SQL 下钻、完整性标注、超长截断，P95 <1s 保持。

#### Scenario: TraceID 精确查询返回瀑布图
- **WHEN** 用户输入 TraceID 查询
- **THEN** 返回该 Trace 的完整 span 树（瀑布图数据），含完整性标注，P95<1s
