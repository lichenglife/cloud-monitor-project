## Why

业务方需按业务场景（如"支付交易"）而非技术接口视角查看链路与指标--业务场景标签聚合是业务运维诉求（PRD US-08/M17）。M17 为入口 URL 设置业务场景标签，按标签聚合链路与指标。

本 change 落地 M17，对应研发任务 T-M17-1/2。

## What Changes

1. **业务场景标签设置**：管理端为入口 URL 设置业务场景标签（URL pattern 通配，如 /api/payment/* -> "支付交易"，DD-01 /biz-scenario，存 t_biz_scenario + Redis biz:url:tag）。
2. **Agent 侧标签附加**：Agent 采集 HTTP Span 时，若 request.url 匹配已配置 pattern，在 Span attribute 附加 biz_scenario 标签（系统设计 §3.5）。
3. **按标签聚合查询**：按业务场景标签筛选时，聚合展示该标签下关联链路的关键指标（QPS/错误率/耗时）+ Top 慢 Trace（读 ES span.attribute.biz_scenario）。
4. **前端业务场景页**：场景管理 CRUD + 场景看板（指标卡 + 关联慢 Trace + 按场景筛选链路跳转）。

## Capabilities

### New Capabilities
- `business-scenario`: 入口 URL 业务场景标签设置 + Agent 侧附加 + 按标签聚合查询

### Modified Capabilities
- `agent-instrumentation`: M1 Java/Go Agent 扩展在 HTTP Span 附加 biz_scenario attribute（匹配 URL pattern）

## Impact

- **依赖**：`add-collection`（M1 Agent HTTP Span 采集 + biz:url:tag Redis）+ `add-trace-query`（M3 按标签筛选链路）+ `chore-foundation`（scenario-service Java 脚手架 + 前端）。
- **下游解锁**：业务视角运维。
- **对应 PRD 模块**：M17（业务场景设置）。
- **设计依据**：DD-01（/biz-scenario 接口）、DD-02（t_biz_scenario/biz:url:tag）、系统设计说明书 §3.5、PRD REQ-M17-E1/E2、DD-17 §6.5。
