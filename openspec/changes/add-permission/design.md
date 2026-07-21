## Context

auth-service（Java Spring Security）负责认证与权限。决议 D CMDB 兜底最小权限，D12 分钟级同步+缓存5m+失效拒绝。设计见 DD-09。

## Goals / Non-Goals

**Goals:**
- 用户认证（登录/JWT/会话）
- RBAC + CMDB 数据权限兜底（默认拒绝）
- CMDB 分钟级同步 + 缓存 5m + 失效拒绝（D12）
- 权限校验拦截 + 越权审计

**Non-Goals:**
- 不做深度合规脱敏（二期，PRD §3.4）
- 不做等保专项（二期）
- 不重建 CMDB（复用现有 CMDB 权限接口，Q4 对齐，mock 占位）

## Decisions

### DM1: RBAC + CMDB 数据权限双层
功能权限 RBAC（角色->权限，菜单/接口）；数据权限 CMDB 兜底（可见应用/系统/机房/中心范围）。无显式授权默认拒绝（最小权限，决议 D）。

### DM2: CMDB 同步时效（D12）
分钟级同步 CMDB 权限 -> perm:cache:{userId}(Redis Set, 5m TTL) + CMDB/角色变更主动失效 + 失效时默认拒绝（不缓存放行）。

### DM3: 权限校验服务端二次校验
前端隐藏仅体验，服务端 CheckPerm 二次校验（不依赖前端）。各服务查询按 perm:cache 过滤可见范围。越权访问返回 40301 + 审计。

### DM4: 审计协同
权限调整/越权访问写 t_audit_log（audit-service，M16 协同，本 change 先写日志接 audit-service）。

## Risks / Trade-offs

- **CMDB 接口未对齐**（Q4）：缓解--mock 占位，D12 时效基于 mock 实现，对齐后切真实。
- **CMDB 不可用致全部拒绝**：缓解--这是安全红线（最小权限），但需告警（接 alert-service）；缓存 5m 内可用。
- **权限缓存与 CMDB 延迟**：5m TTL 窗口内权限变更可能滞后。缓解--角色/CMDB 变更主动失效缓存。
