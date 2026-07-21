## Why

平台需运营管理平面--用户/项目/集群/系统字典/审计管理（PRD US-05/M16）。Agent 台账已在 add-collection(M1) 落地（agent-registry 台账查询 + 前端 /agents 页），本 change 不重复，只做用户/项目/集群/系统字典/审计。对应研发任务 T-M16-1/2/4/5（T-M16-3 Agent 台账已在 M1）。

## What Changes

1. **用户管理**：创建/编辑/禁用/角色分配（DD-01 GET/POST/PUT/DELETE /users），含超级管理员，与 M15 auth-service RBAC 协同。
2. **项目管理**：创建项目/清单展示/项目与集群关联（DD-01 /projects）。
3. **集群管理**：K8S 集群列表（网络区域/机房/集群名/类型，DD-01 /clusters），多中心 biz/dmz。
4. **系统字典**：下拉选项/默认数据配置维护（DD-01 /dicts）。
5. **操作审计**：用户全量操作审计查询（DD-01 /audit/logs），audit-service append-only，按操作人/动作/时间筛选，不可篡改。
6. **前端运营管理页**：用户/项目/集群/字典/审计 CRUD 页（Vue3 admin feature）。

## Capabilities

### New Capabilities
- `ops-management`: 用户/项目/集群/系统字典 CRUD（管理平面）
- `audit-management`: 操作审计查询与归档（audit-service）

### Modified Capabilities
<!-- 无；Agent 台账（T-M16-3）已在 add-collection 落地，本 change 不重复 -->

## Impact

- **依赖**：`add-permission`（M15 RBAC + t_user/role/permission + audit 接入）+ `add-storage-ingestion`（MySQL t_user/project/cluster/dict/audit_log）+ `chore-foundation`（audit-service Java 脚手架 + 前端 + 契约）。外部：集群接口（容器平台 K8S API，先用 mock）。
- **下游解锁**：运营管理是平台治理出口。
- **对应 PRD 模块**：M16（运营管理平台，Agent 台账除外）。
- **设计依据**：DD-01 §3.1（/users /projects /clusters /dicts /audit/logs）、DD-02（t_user/project/cluster/dict/audit_log）、DD-09（审计 append-only）、DD-15 §2（管理后台页）、DD-17 §6.7。
