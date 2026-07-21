## ADDED Requirements

### Requirement: 用户认证与会话
auth-service MUST 支持用户登录/登出/注销，MUST 签发 JWT 并存会话 Redis user:session(2h TTL)，MUST 支持令牌刷新，MUST 内置超级管理员。

#### Scenario: 登录签发 JWT
- **WHEN** 用户凭密码登录成功
- **THEN** 签发 JWT + 存 Redis 会话（2h TTL），后续请求携 JWT 鉴权

### Requirement: RBAC + CMDB 数据权限兜底（决议 D）
MUST 实现平台 RBAC（t_user/t_role/t_permission，功能权限由角色控制）+ CMDB 数据权限兜底（可见应用/系统/机房/中心范围），平台无显式授权时 MUST 默认拒绝（最小权限）。

#### Scenario: 无授权默认拒绝
- **WHEN** 用户访问非权限范围内系统链路数据
- **THEN** 拒绝并提示无权限，返回 40301，记审计（PRD REQ-M15-E1）

### Requirement: CMDB 同步时效（D12）
MUST 分钟级同步 CMDB 权限至 perm:cache:{userId}(5m TTL)，CMDB/角色变更 MUST 主动失效缓存，缓存失效时 MUST 默认拒绝（不缓存放行）。

#### Scenario: CMDB 不可用默认拒绝
- **WHEN** CMDB 权限数据不可用且缓存失效
- **THEN** 默认最小权限拒绝并告警，不缓存放行（D12）

#### Scenario: 权限变更主动失效
- **WHEN** 用户角色或 CMDB 权限变更
- **THEN** 主动失效 perm:cache，≤5m 生效新权限
