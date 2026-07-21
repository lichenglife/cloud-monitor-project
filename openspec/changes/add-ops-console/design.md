## Context

M15 auth-service 已落 RBAC + t_user/role/permission。M1 已落 Agent 台账。M16 做用户/项目/集群/字典/审计管理平面（audit-service Java + 前端 admin 页）。设计见 DD-01 §3.1 + DD-09 审计。

## Goals / Non-Goals

**Goals:**
- 用户/项目/集群/系统字典 CRUD
- 操作审计查询（append-only，不可篡改）
- 前端运营管理页

**Non-Goals:**
- 不做 Agent 台账（M1 已落地）
- 不重建 CMDB（复用）
- 不做深度合规报表（二期）

## Decisions

### DM1: 管理平面与权限协同
用户/角色管理与 M15 auth-service RBAC 共用 t_user/t_role/t_permission，本 change 做 CRUD UI + 接口，权限校验复用 M15 CheckPerm。

### DM2: 审计 append-only
audit-service 写 t_audit_log append-only（不可修改/删除，仅追加），按 operator+action+time 索引查询，定期归档（系统设计 §2.14.1，180d 清理）。

### DM3: 集群管理多中心
集群列表含网络区域(biz/dmz)/机房/数据中心/类型(k8s/vm)，支撑多中心拓扑与 CMDB 权限。集群接口先用 mock（容器平台 K8S API）。

## Risks / Trade-offs

- **集群接口未对齐**：缓解--mock 占位，对齐后切真实 K8S API。
- **审计高频写入**：t_audit_log 量大。缓解--按 operator+action+time 索引 + 180d 清理 + 读写分离（DD-02）。
