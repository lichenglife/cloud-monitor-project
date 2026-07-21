# add-topology（M4）实现任务清单

> **TDD 强制**：RED->GREEN->REFACTOR，覆盖率 ≥80%。前置：add-storage-ingestion 的 otel-spans Kafka + ES 就绪。CMDB 接口先用 mock（Q4 待对齐）。
> **涉及组件**：topology-service(Go) / 前端 topology 页(Vue3)。设计依据：DD-16 / DD-03 §3/§9.3 / DD-08 / DD-15 §7-8 / 决议 D2 / DD-17 §6.2。

## 阶段 1：topology-calc 依赖发现与边聚合（Go）

### 1.1 消费聚合边
- [ ] 1.1.1 [RED] 聚合单测：消费 otel-spans -> 按 (source,target_type,target)×多中心 聚合 1m 边；target_type∈{service,db,mq,cache}；call_count/error_count/avg_duration/p95_duration
- [ ] 1.1.2 [GREEN] 实现 topology-calc 消费聚合（写 npm_online_topology，SRP 独立）
- [ ] 1.1.3 [验收] A->B(service边)+A->MySQL(db边) 自动发现，无需人工录入；覆盖率 ≥80%

### 1.2 强弱依赖与异常高亮
- [ ] 1.2.1 [RED] 弱依赖单测：边出呼叫占比 <5% 节点总出调用->弱依赖标记
- [ ] 1.2.2 [RED] 异常高亮单测：错误率 >1% 或 p95>SLO->红标记
- [ ] 1.2.3 [RED] 未知依赖单测：数据缺失->'未知依赖'+提示人工确认，不静默丢弃
- [ ] 1.2.4 [GREEN] 实现强弱依赖/异常/未知依赖判定（阈值 config-center 可配）
- [ ] 1.2.5 [验收] 弱依赖虚线、异常红、未知依赖提示正确；覆盖率 ≥80%

### 1.3 scope 聚合（D2）
- [ ] 1.3.1 [RED] scope 单测：global=跨系统系统级；system=应用级；center=按 data_center；room=机房级；下钻 system->engineering（按 engineering_id）
- [ ] 1.3.2 [GREEN] 实现 scope 物化聚合（global/system/center/room + engineering 下钻）
- [ ] 1.3.3 [验收] 默认全局系统级，点击系统下钻工程级；覆盖率 ≥80%

### 1.4 快照合并与跨中心边
- [ ] 1.4.1 [RED] 合并单测：topology.merge.interval=60s 合并快照写 t_topology_snapshot + Redis topology:cache(1m TTL)
- [ ] 1.4.2 [RED] 跨中心边单测：同 trace_id 跨中心调用->跨中心边（source.data_center->target.data_center）
- [ ] 1.4.3 [GREEN] 实现快照合并 + 跨中心边生成
- [ ] 1.4.4 [验收] 60s 合并最终一致性，跨中心边特殊样式；覆盖率 ≥80%

### 1.5 CMDB 元数据补全
- [ ] 1.5.1 [RED] 补全单测：节点/边补全机房/系统/负责人（CMDB mock 接口）
- [ ] 1.5.2 [GREEN] 实现 CMDB 元数据补全（mock 占位，Q4 对齐后切真实）
- [ ] 1.5.3 [验收] 元数据补全正确（mock）；覆盖率 ≥80%

## 阶段 2：前端拓扑页 G6 WebGL（Vue3）

### 2.1 G6 WebGL 渲染
- [ ] 2.1.1 [RED] 渲染单测：G6 v5 WebGL 渲染节点/边；强弱依赖样式（实线/虚线）；异常红高亮；跨中心边特殊样式
- [ ] 2.1.2 [RED] LOD 单测：视口外/缩小时降细节（隐藏标签/合并边），放大显全
- [ ] 2.1.3 [RED] 聚类单测：按 scope(中心/机房)收起子图，下钻展开
- [ ] 2.1.4 [RED] Worker 布局单测：力导/分层计算在 Web Worker，主线程只收坐标不阻塞
- [ ] 2.1.5 [GREEN] 实现 TopologyCanvas（G6 WebGL + LOD + 聚类 + Worker 布局）
- [ ] 2.1.6 [验收] 万级节点 PoC 帧率达标（D2，2 周内出结论）；覆盖率 ≥70%

### 2.2 降级兜底（方案 B）
- [ ] 2.2.1 [RED] 降级单测：节点超阈值->自动切聚类收起（方案 B）
- [ ] 2.2.2 [GREEN] 实现超阈值降级聚类
- [ ] 2.2.3 [验收] 超阈值降级保可用性；覆盖率 ≥70%

### 2.3 下钻与指标叠加
- [ ] 2.3.1 [RED] 下钻单测：节点->实例列表->容器->主机资源指标；点击系统->工程级下钻
- [ ] 2.3.2 [RED] 指标叠加单测：边/节点叠加错误率/P95/吞吐
- [ ] 2.3.3 [RED] 时间回放单测：选历史 snapshot_time 回放拓扑演变
- [ ] 2.3.4 [GREEN] 实现下钻 + 指标叠加 + 时间回放
- [ ] 2.3.5 [验收] 下钻至主机资源指标，回放数据一致；覆盖率 ≥70%

### 2.4 权限过滤
- [ ] 2.4.1 [RED] 权限单测：按 perm:cache 过滤可见节点/边，无权限置灰/隐藏
- [ ] 2.4.2 [GREEN] 实现 PermissionGate 拓扑权限过滤（服务端二次校验）
- [ ] 2.4.3 [验收] 越权节点不返回/置灰；覆盖率 ≥70%

## 阶段 3：跨组件联调验收（Gate，DD-17 §6.2）

- [ ] 3.1 [联调] add-collection otel-spans -> topology-calc 聚合 -> npm_online_topology 边
- [ ] 3.2 [联调] 前端全局拓扑 -> 默认系统级 -> 点击系统下钻工程级
- [ ] 3.3 [联调] 强弱依赖/异常高亮/未知依赖标注
- [ ] 3.4 [联调] 跨中心边 + CMDB 元数据补全（mock）
- [ ] 3.5 [联调] 权限过滤拓扑可见范围
- [ ] 3.6 [验收] DD-17 §6.2：拓扑自动发现/CMDB补全/强弱依赖/纵向下钻/scope切换/异常高亮/时间回放/数据延迟标注
- [ ] 3.7 [验收] 拓扑自动发现覆盖率 ≥90%（试点）；万节点 PoC 帧率达标；覆盖率 ≥80%（后端）/≥70%（前端）

