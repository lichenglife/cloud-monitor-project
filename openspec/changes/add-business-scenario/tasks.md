# add-business-scenario（M17）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-collection Agent HTTP Span + add-trace-query 链路筛选。
> **涉及组件**：scenario-service(Java) / Agent(M1扩展) / 前端业务场景页(Vue3)。设计依据：系统设计 §3.5 / DD-02 / DD-01 / DD-17 §6.5。

## 阶段 1：业务场景标签设置（Java）

### 1.1 标签 CRUD
- [ ] 1.1.1 [RED] CRUD 单测：t_biz_scenario（name/url_pattern/description/owner/tags/status）；Redis biz:url:tag 映射更新
- [ ] 1.1.2 [GREEN] 实现 /biz-scenario CRUD gRPC + MySQL t_biz_scenario + Redis biz:url:tag
- [ ] 1.1.3 [验收] 标签 CRUD + Redis 映射；覆盖率 ≥80%

## 阶段 2：Agent 侧标签附加（M1 扩展）

- [ ] 2.1 [RED] 附加单测：Agent 采集 HTTP Span，request.url 匹配 pattern->Span attribute 附加 biz_scenario；不匹配不变；Span 名不变（控基数）
- [ ] 2.2 [RED] 匹配性能单测：pattern 内存缓存 + 低基数，不显著增加每请求开销
- [ ] 2.3 [GREEN] 实现 Agent HTTP Span biz_scenario 附加（Java SkyWalking 插件 + Go OTel SDK 扩展）
- [ ] 2.4 [验收] URL 匹配附加标签，Span 名不变；覆盖率 ≥80%

## 阶段 3：按标签聚合查询（Java）

### 3.1 聚合查询
- [ ] 3.1.1 [RED] 聚合单测：按 span.attribute.biz_scenario 聚合 QPS/错误率/耗时 + Top 慢 Trace
- [ ] 3.1.2 [RED] 跳转单测：按场景筛选链路->跳转 M3 链路检索（自动带入 biz_scenario filter）
- [ ] 3.1.3 [GREEN] 实现按标签聚合查询 + 链路筛选跳转
- [ ] 3.1.4 [验收] 按场景聚合 + 跳转链路检索；覆盖率 ≥80%

## 阶段 4：前端业务场景页（Vue3）

- [ ] 4.1 [RED] 管理页单测：场景 CRUD + URL pattern 配置 + 标签管理
- [ ] 4.2 [GREEN] 实现场景管理页
- [ ] 4.3 [RED] 看板单测：场景选择->指标卡(QPS/错误率/耗时)+Top慢Trace+按场景筛选链路跳转
- [ ] 4.4 [GREEN] 实现场景看板
- [ ] 4.5 [验收] 场景管理 + 看板 + 跳转；覆盖率 ≥70%

## 阶段 5：跨组件联调验收（Gate，DD-17 §6.5）

- [ ] 5.1 [联调] M17 配置 URL pattern -> M1 Agent 附加 biz_scenario -> ES span.attribute
- [ ] 5.2 [联调] 按场景聚合 -> 指标卡 + Top 慢 Trace
- [ ] 5.3 [联调] 按场景筛选 -> 跳转 M3 链路检索（自动 filter）
- [ ] 5.4 [验收] DD-17 §6.5：入口URL业务标签可设置并聚合链路
- [ ] 5.5 [验收] 覆盖率 ≥80%（后端）/≥70%（前端）

