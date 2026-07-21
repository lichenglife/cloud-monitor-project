## ADDED Requirements

### Requirement: 操作审计查询与不可篡改
audit-service MUST 记录用户全量操作（配置变更/权限调整/Agent 升级/数据导出）至 t_audit_log，MUST append-only 不可修改/删除，MUST 支持按操作人/动作/时间筛选查询（/audit/logs），MUST 定期归档（180d 清理）。

#### Scenario: 审计不可篡改
- **WHEN** 任意用户尝试修改或删除审计记录
- **THEN** 拒绝（append-only），审计日志仅追加写入

#### Scenario: 审计按条件查询
- **WHEN** 安全合规人员按操作人+动作+时间查询审计
- **THEN** 返回匹配审计记录，可追溯全部操作
