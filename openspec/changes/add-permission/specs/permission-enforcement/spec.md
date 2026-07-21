## ADDED Requirements

### Requirement: 权限校验拦截与越权审计
MUST 提供 VerifyToken/CheckPerm gRPC 供各服务调用，MUST 服务端二次校验（不依赖前端），各服务查询 MUST 按 perm:cache 过滤可见范围，越权访问 MUST 返回 40301 + 写 t_audit_log。

#### Scenario: 服务端二次校验越权
- **WHEN** 前端隐藏后用户直接请求无权限接口
- **THEN** 服务端 CheckPerm 拦截返回 40301 + 审计，不依赖前端隐藏

#### Scenario: 查询按权限过滤可见范围
- **WHEN** 用户查询链路/拓扑/大盘
- **THEN** 按 perm:cache 过滤仅返回权限范围内数据，越权数据不返回
