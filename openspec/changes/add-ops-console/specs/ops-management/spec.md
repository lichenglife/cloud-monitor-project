## ADDED Requirements

### Requirement: 用户/项目/集群/字典 CRUD
MUST 支持用户管理（创建/编辑/禁用/角色分配，/users）、项目管理（创建/清单/集群关联，/projects）、集群管理（K8S 集群列表含网络区域/机房/数据中心/类型，/clusters）、系统字典维护（/dicts），MUST 与 M15 RBAC 共用 t_user/t_role/t_permission，MUST 经 CheckPerm 权限校验（仅超管/运维负责人可访问）。

#### Scenario: 用户创建与角色分配
- **WHEN** 超管创建用户并分配角色
- **THEN** 用户写 t_user + 角色关联 t_role，经 CheckPerm 校验权限，操作记审计

#### Scenario: 项目关联集群
- **WHEN** 运维创建项目并关联集群
- **THEN** 项目写 t_project + 集群关联，支撑多中心拓扑与权限

### Requirement: 集群多中心管理
集群列表 MUST 含网络区域(biz/dmz)/机房/数据中心/类型(k8s/vm)，支撑多中心拓扑与 CMDB 权限基线。

#### Scenario: 多中心集群列表
- **WHEN** 运维查看集群列表
- **THEN** 展示各中心 biz/dmz 集群（机房/数据中心/类型），支撑拓扑与权限
