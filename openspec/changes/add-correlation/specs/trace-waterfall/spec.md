## MODIFIED Requirements

### Requirement: 瀑布图关联入口接通
trace-waterfall（M3）的"关联指标/关联日志"入口按钮 MUST 接通 add-correlation 的实际关联面板（M6 落地前为占位，M6 后接通）。

#### Scenario: 关联入口接通
- **WHEN** M6 落地后用户点击瀑布图"关联日志"
- **THEN** 接通实际日志关联面板（M6），不再占位
