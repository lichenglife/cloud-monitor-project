# add-permission（M15）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-storage-ingestion MySQL + chore-foundation auth-service 脚手架。CMDB 接口 mock 占位（Q4 待对齐）。
> **涉及组件**：auth-service(Java) / 各服务权限过滤接入。设计依据：DD-09 / DD-02 / DD-01 §5 / 决议 D/D12 / DD-17 §6.7。

## 阶段 1：用户认证与 RBAC（Java）

### 1.1 认证与会话
- [ ] 1.1.1 [RED] 登录单测：凭密码登录->签发 JWT + 存 Redis user:session(2h TTL)；登出/注销；令牌刷新；内置超级管理员
- [ ] 1.1.2 [GREEN] 实现 auth-service 登录/JWT/会话（Spring Security）
- [ ] 1.1.3 [验收] 登录签发 JWT + 会话；覆盖率 ≥80%

### 1.2 RBAC
- [ ] 1.2.1 [RED] RBAC 单测：t_user/t_role/t_permission CRUD；角色->功能权限（菜单/接口）
- [ ] 1.2.2 [GREEN] 实现 RBAC（角色权限分配）
- [ ] 1.2.3 [验收] RBAC 角色权限控制正确；覆盖率 ≥80%

## 阶段 2：CMDB 数据权限兜底（决议 D）

### 2.1 CMDB 同步与缓存（D12）
- [ ] 2.1.1 [RED] 同步单测：分钟级同步 CMDB 权限->perm:cache:{userId}(5m TTL)；CMDB/角色变更主动失效
- [ ] 2.1.2 [RED] 失效拒绝单测：缓存失效时默认拒绝（不缓存放行）+ 告警
- [ ] 2.1.3 [GREEN] 实现 CMDB 同步（mock 占位）+ 缓存 + 主动失效 + 失效拒绝
- [ ] 2.1.4 [验收] D12 分钟级同步+5m缓存+失效拒绝；覆盖率 ≥80%

### 2.2 数据权限过滤
- [ ] 2.2.1 [RED] 过滤单测：按 perm:cache 过滤可见应用/系统/机房/中心；无显式授权默认拒绝
- [ ] 2.2.2 [GREEN] 实现数据权限过滤（CMDB 兜底）
- [ ] 2.2.3 [验收] 无授权默认拒绝 40301；覆盖率 ≥80%

## 阶段 3：权限校验与审计（Java）

### 3.1 校验拦截
- [ ] 3.1.1 [RED] 校验单测：VerifyToken/CheckPerm gRPC；服务端二次校验不依赖前端；越权返回 40301
- [ ] 3.1.2 [GREEN] 实现 VerifyToken/CheckPerm gRPC + 拦截器
- [ ] 3.1.3 [验收] 服务端二次校验越权拦截；覆盖率 ≥80%

### 3.2 越权审计
- [ ] 3.2.1 [RED] 审计单测：越权访问/权限调整写 t_audit_log（接 audit-service）
- [ ] 3.2.2 [GREEN] 实现越权审计（写 t_audit_log）
- [ ] 3.2.3 [验收] 越权审计可追溯；覆盖率 ≥80%

## 阶段 4：跨组件联调验收（Gate，DD-17 §6.7）

- [ ] 4.1 [联调] auth-service 登录 -> JWT -> 各服务 CheckPerm 鉴权
- [ ] 4.2 [联调] M3 链路/M4 拓扑/M14 大盘按 perm:cache 过滤可见范围
- [ ] 4.3 [联调] CMDB 变更 -> 主动失效 -> ≤5m 生效新权限
- [ ] 4.4 [联调] CMDB 不可用 -> 默认拒绝 + 告警
- [ ] 4.5 [联调] 越权访问 -> 40301 + 审计（audit-service）
- [ ] 4.6 [验收] DD-17 §6.7：CMDB权限隔离/审计/越权拒绝；权限兜底 100% 越权拦截
- [ ] 4.7 [验收] 覆盖率 ≥80%

