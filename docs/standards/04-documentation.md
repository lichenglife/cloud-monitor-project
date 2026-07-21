# 文档规范（通用，语言无关）

适用范围：README、架构决策记录（ADR）、API 文档、CHANGELOG、代码注释、运行手册。

## 1. README（MUST）

每个仓库根目录 **MUST** 有 `README.md`，至少包含：
- 项目简介与定位
- 快速开始（环境、安装、启动命令）
- 构建 / 测试 / 部署基本命令
- 目录导航（指向 `docs/standards`）
- 贡献方式（指向提交规范、PR 规范）

## 2. 架构决策记录 ADR（SHOULD）

- 重要架构 / 技术选型变更 **SHOULD** 写 ADR，置于 `docs/adr/NNNN-title.md`。
- 模板：`状态 / 背景 / 决策 / 后果 / 备选方案`。
- ADR 只追加不删除；被推翻时标记 `已废弃` 并写新 ADR。

## 3. API 文档（MUST）

- 所有对外 API **MUST** 提供机器可读契约：REST 用 OpenAPI 3（`openapi.yaml`/`swagger`），gRPC 用 `.proto` + 注释。
- 文档 **MUST** 随代码变更同步更新（CI 校验契约与实现一致）。
- 公共 API 字段 **MUST** 有说明、示例、错误码。

## 4. CHANGELOG（MUST）

- 遵循 Keep a Changelog：按 `Added / Changed / Deprecated / Removed / Fixed / Security` 分类。
- 每个发版 **MUST** 对应一个版本段落，关联 issue / PR。
- 版本号引用 SemVer（见发版规范）。

## 5. 代码注释（SHOULD，克制）

- 注释解释 **Why**，不重复 **What**（代码本身可读）。
- 公共 API / 导出符号 **MUST** 有文档注释（Java `/** */`、Go `//` 紧邻符号）。
- 复杂算法 / 业务规则 / 坑点 **SHOULD** 写注释并引用 issue。
- 禁止无意义注释（`// 设置 name`、`// 循环`）。
- TODO **SHOULD** 带 issue 链接：`// TODO(#123): 处理时区边界`。

## 6. 运行手册 / Runbook（SHOULD）

- 关键服务 **SHOULD** 有 `docs/runbook.md`：如何启停、常见故障、告警含义、on-call 动作、回滚步骤。

## 7. 文档即代码（MUST）

- 文档随代码评审；文档过期视为缺陷。
- 禁止在 IM / 个人笔记里沉淀"唯一真相"，必须进仓库。

## 检查清单

- [ ] README 含快速开始与导航
- [ ] 对外 API 有 OpenAPI / proto 且同步更新
- [ ] CHANGELOG 含本次版本段落
- [ ] 公共符号有文档注释，无废话注释
- [ ] 重要决策有 ADR
