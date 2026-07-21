## ADDED Requirements

### Requirement: Helm Charts + K8s 部署清单
每个微服务 MUST 有 Helm Chart + K8s yaml（namespace `tracing-system`），含 Deployment + HPA（按 CPU/吞吐）+ PDB + Service + ConfigMap（静态配置）+ Secret（敏感配置 KMS/AES-256-GCM 加密），mTLS 经 cert-manager 初始化。

#### Scenario: 服务水平扩缩容
- **WHEN** Collector CPU 或吞吐超阈值
- **THEN** HPA 自动扩容 Collector 副本，PDB 保证滚动更新不丢可用副本

#### Scenario: dmz 代理转发至 biz Collector
- **WHEN** dmz 区 Agent 上报数据
- **THEN** 经 dmz 代理（HAProxy/Nginx）转发至本中心 biz 区 Collector，符合决议 A 部署架构

### Requirement: Git 分支模型与发布策略
MUST 定 `main`(生产,保护)/`release/x.y`/`develop`/`feature/任务号-简述` 分支模型，提交 `[任务号] 动作：简述` 关联 TAPD 任务，PR ≥1 人 Review + CI 全绿方入 develop，main 仅经 release 合并，发布蓝绿/金丝雀可一键回滚。

#### Scenario: 金丝雀发布异常自动回滚
- **WHEN** 金丝雀阶段（1%->10%->100%）指标异常（错误率突增/心跳丢失）
- **THEN** 自动回滚至稳定版，无需人工介入
