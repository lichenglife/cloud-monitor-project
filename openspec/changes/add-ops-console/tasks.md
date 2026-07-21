# add-ops-console（M16）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-permission RBAC + add-storage-ingestion MySQL。Agent 台账已在 M1 落地，不重复。集群接口 mock。
> **涉及组件**：audit-service(Java) / 前端 admin 页(Vue3)。设计依据：DD-01 §3.1 / DD-02 / DD-09 / DD-17 §6.7。

## 阶段 1：用户/项目/集群/字典管理（Java）

### 1.1 用户管理
- [ ] 1.1.1 [RED] 用户 CRUD 单测：创建/编辑/禁用/角色分配；与 M15 RBAC 共用 t_user/t_role/t_permission；CheckPerm 校验
- [ ] 1.1.2 [GREEN] 实现 /users CRUD gRPC（复用 M15 t_user）
- [ ] 1.1.3 [验收] 用户 CRUD + 角色分配 + 权限校验；覆盖率 ≥80%

### 1.2 项目/集群/字典管理
- [ ] 1.2.1 [RED] 项目单测：创建/清单/集群关联（t_project）
- [ ] 1.2.2 [RED] 集群单测：列表含网络区域/机房/数据中心/类型（t_cluster，多中心 biz/dmz）
- [ ] 1.2.3 [RED] 字典单测：CRUD（t_dict）
- [ ] 1.2.4 [GREEN] 实现 /projects /clusters /dicts CRUD
- [ ] 1.2.5 [验收] 项目/集群/字典 CRUD + 多中心集群；覆盖率 ≥80%

## 阶段 2：操作审计（Java，audit-service）

### 2.1 审计记录与不可篡改
- [ ] 2.1.1 [RED] 审计单测：配置变更/权限调整/Agent升级/数据导出写 t_audit_log；append-only 不可改删
- [ ] 2.1.2 [GREEN] 实现 audit-service 审计记录（append-only）
- [ ] 2.1.3 [验收] 审计 append-only 不可篡改；覆盖率 ≥80%

### 2.2 审计查询与归档
- [ ] 2.2.1 [RED] 查询单测：按操作人/动作/时间筛选（/audit/logs）
- [ ] 2.2.2 [RED] 归档单测：180d 定期清理
- [ ] 2.2.3 [GREEN] 实现 /audit/logs 查询 + 归档定时任务
- [ ] 2.2.4 [验收] 审计查询 + 归档；覆盖率 ≥80%

## 阶段 3：前端运营管理页（Vue3）

- [ ] 3.1 [RED] 管理页单测：用户/项目/集群/字典/审计 CRUD 页（features/admin）
- [ ] 3.2 [GREEN] 实现 features/admin 运营管理页
- [ ] 3.3 [RED] 权限单测：仅超管/运维负责人可访问（PermissionGate）
- [ ] 3.4 [GREEN] 实现管理页权限控制
- [ ] 3.5 [验收] 运营管理 CRUD + 权限控制；覆盖率 ≥70%

## 阶段 4：跨组件联调验收（Gate，DD-17 §6.7）

- [ ] 4.1 [联调] M15 RBAC -> M16 用户/角色管理共用 t_user/t_role
- [ ] 4.2 [联调] 用户操作 -> audit-service 审计记录 -> 审计查询
- [ ] 4.3 [联调] 集群管理 -> 多中心拓扑(M4)/CMDB 权限(M15)
- [ ] 4.4 [验收] DD-17 §6.7：用户/项目/集群/Agent台账(M1)/系统字典/审计可用
- [ ] 4.5 [验收] 覆盖率 ≥80%（后端）/≥70%（前端）

