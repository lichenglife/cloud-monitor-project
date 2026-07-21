# add-observability-view（M14）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-storage-ingestion 聚合索引 + add-trace-query + add-topology + add-alert-center。
> **涉及组件**：metric-aggr-service(Go) / trace-query-service(Go) / 前端 dashboard+apm 页(Vue3)。设计依据：DD-03 §9.1/§9.2 / DD-15 §2 / DD-01 §3.3/§3.5 / 议题B / DD-17 §6.2/§6.4。

## 阶段 1：服务健康评分（Go，DD-03 §9.1）

### 1.1 4 维惩罚制计算
- [ ] 1.1.1 [RED] 评分单测：health=clamp(100-(E×0.35+L×0.25+R×0.20+A×0.20),0,100)；E 错误率(min(100,err/total×200))/L 时延(clamp((p95-slo)/slo,0,1)×100)/R 资源(max(cpu,mem,gc)归一)/A 告警(Σsev_w×cnt,P0=40/P1=20/P2=10/P3=5)
- [ ] 1.1.2 [RED] 分级单测：90-100健康/70-89关注/40-69警告/<40严重
- [ ] 1.1.3 [GREEN] 实现健康评分计算（读 npm_online_performe/metric/event）
- [ ] 1.1.4 [验收] 4 维惩罚制 + 分级正确；覆盖率 ≥80%

### 1.2 定时计算与缓存
- [ ] 1.2.1 [RED] 定时单测：5min 定时触发；缓存 Redis health:score(1m TTL)；写 t_health_score_history
- [ ] 1.2.2 [GREEN] 实现定时计算 + 缓存 + 历史记录
- [ ] 1.2.3 [验收] 定时计算，前端读缓存不实时算；覆盖率 ≥80%

## 阶段 2：APM topN/热点/火焰图（Go，议题 B）

### 2.1 接口/中间件 topN
- [ ] 2.1.1 [RED] topN 单测：按 throughput/error/latency topN（读 npm_online_performe）；中间件 topN（npm_online_db）
- [ ] 2.1.2 [GREEN] 实现 GET /services/{name}/top-apis + 中间件 topN
- [ ] 2.1.3 [验收] topN 排序正确；覆盖率 ≥80%

### 2.2 热点方法分析
- [ ] 2.2.1 [RED] 热点单测：按 operation_name/代码级 span 聚合 self_time(duration-Σchildren)+调用次数 TopN；未开 code_level->提示
- [ ] 2.2.2 [GREEN] 实现 GET /services/{name}/hotspots
- [ ] 2.2.3 [验收] 热点 TopN + self_time 正确，未开启提示；覆盖率 ≥80%

### 2.3 火焰图（代表 Trace 聚合）
- [ ] 2.3.1 [RED] 火焰图单测：trace_npm_jaeger 源，按 parent_span_id (service,1m bucket) 层级；选 Top-K 慢 Trace(K=100) 聚合 self_time；不呈现每条 Trace；缓存保障 P95<1s
- [ ] 2.3.2 [GREEN] 实现 GET /services/{name}/flamegraph（代表 Trace 聚合 + 缓存）
- [ ] 2.3.3 [验收] 火焰图聚合正确，P95<1s；覆盖率 ≥80%

## 阶段 3：统一大盘与服务指标（Go）

### 3.1 统一大盘聚合
- [ ] 3.1.1 [RED] 大盘单测：全局健康/核心指标卡(错误率/P95/吞吐)/告警摘要/多中心健康/健康排行/拓扑缩略图；缓存 dashboard:overview:cache(60s TTL)
- [ ] 3.1.2 [GREEN] 实现 GET /dashboard/overview（并行聚合 + 多级缓存）
- [ ] 3.1.3 [验收] 大盘聚合 + 缓存命中降 ES 压力，P95<1s；覆盖率 ≥80%

### 3.2 服务/实例指标图表
- [ ] 3.2.1 [RED] 指标单测：服务级/实例级切换；JVM/资源/接口指标图表（读 npm_online_metric）
- [ ] 3.2.2 [GREEN] 实现 GET /services/{name} 指标图表
- [ ] 3.2.3 [验收] 粒度切换 + 指标图表正确；覆盖率 ≥80%

## 阶段 4：前端大盘 + APM 页（Vue3）

### 4.1 统一大盘页
- [ ] 4.1.1 [RED] 大盘页单测：时间范围/scope 切换；指标卡+健康排行+错误率QPS趋势+告警列表+拓扑缩略图；下钻联动
- [ ] 4.1.2 [GREEN] 实现 features/dashboard 大盘页（ECharts + 健康仪表）
- [ ] 4.1.3 [验收] 大盘下钻联动；覆盖率 ≥70%

### 4.2 APM 页（热点/火焰/健康/topN）
- [ ] 4.2.1 [RED] APM 页单测：火焰图(Canvas+Worker 层级聚合)/热点列表/健康仪表(0-100)/接口 TopN 表格
- [ ] 4.2.2 [GREEN] 实现 features/apm 页（FlameGraph Canvas+Worker + HealthGauge + TopN）
- [ ] 4.2.3 [验收] 火焰图/热点/健康/topN 渲染正确；覆盖率 ≥70%

## 阶段 5：跨组件联调验收（Gate，DD-17 §6.2/§6.4）

- [ ] 5.1 [联调] M2 聚合索引 -> M14 大盘/健康/topN/火焰图
- [ ] 5.2 [联调] 健康分异常 -> 下钻拓扑(M4)/链路(M3)/告警(M5.1)
- [ ] 5.3 [联调] 火焰图 -> 代表 Trace 聚合 -> 下钻方法/调用链
- [ ] 5.4 [联调] 大盘告警摘要 -> M5.1 告警中心
- [ ] 5.5 [验收] DD-17 §6.4 M4(APM)：火焰图/热点TopN/健康评分/接口TopN；§6.2 大盘健康分/指标/topN
- [ ] 5.6 [验收] 健康评分权重模型 v1（错误35/时延25/资源20/告警20）；覆盖率 ≥80%（后端）/≥70%（前端）

