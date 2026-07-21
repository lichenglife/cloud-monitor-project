## Why

链路数据含敏感业务信息，需权限控制谁能看哪些系统数据--安全生产诉求（PRD US-06）。M15 基础权限对接 CMDB，RBAC + 数据权限兜底最小权限，是 M4 拓扑/M3 链路/M14 大盘权限过滤的依赖。决议 D：CMDB 权限兜底最小权限；D12：分钟级同步 + 缓存 5m + 失效默认拒绝。

本 change 落地 M15，对应研发任务 T-M15-1~T-M15-3。

## What Changes

1. **用户认证**：auth-service（Java Spring Security）登录/登出/注销，JWT 签发，会话 Redis user:session(2h TTL)，令牌刷新。
2. **RBAC 权限**：t_user/t_role/t_permission，功能权限（菜单/接口）由角色控制。
3. **CMDB 数据权限兜底（决议 D）**：可见应用/系统/机房/中心范围由 CMDB 同步；平台无显式授权时默认拒绝（最小权限），避免越权查看其他团队 Trace。
4. **CMDB 同步时效（D12）**：分钟级同步 + 缓存 perm:cache:{userId}(5m TTL) + 失效默认拒绝；CMDB/角色变更主动失效。
5. **权限校验**：VerifyToken/CheckPerm gRPC（DD-01 §5），各服务查询按 perm:cache 过滤可见范围（M3/M4/M14 已预留权限过滤）。
6. **审计**：权限调整/越权访问写 t_audit_log（与 M16 audit-service 协同）。
7. **深度合规留二期**：敏感字段脱敏规则、等保专项不在一期（PRD §3.4）。

## Capabilities

### New Capabilities
- `auth-rbac`: 用户认证 + RBAC + CMDB 数据权限兜底（决议 D）+ 同步时效（D12）
- `permission-enforcement`: 权限校验拦截（VerifyToken/CheckPerm）+ 越权拒绝 + 审计

### Modified Capabilities
<!-- 无，M15 为新能力；M3/M4/M14 已预留权限过滤接入点 -->

## Impact

- **依赖**：`chore-foundation`（auth-service Java 脚手架 + 契约）+ `add-storage-ingestion`（MySQL t_user/role/permission）。外部：CMDB 权限接口（Q4 待对齐，先用 mock，D12 时效基于 mock）。
- **下游解锁**：M3/M4/M14 权限过滤（已预留）、M16 运营管理（用户/角色管理）。
- **对应 PRD 模块**：M15（权限与合规基础）。
- **设计依据**：DD-09（安全与权限方案）、DD-02（t_user/role/permission/perm:cache）、DD-01 §5（VerifyToken/CheckPerm）、决议 D/D12、DD-17 §6.7（权限验收）。
