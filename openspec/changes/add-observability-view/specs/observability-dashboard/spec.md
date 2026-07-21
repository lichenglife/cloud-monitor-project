## ADDED Requirements

### Requirement: 统一大盘与下钻联动
MUST 实现 GET /dashboard/overview 统一大盘（全局健康仪表/核心指标卡错误率-P95-吞吐/告警摘要/多中心健康/服务健康排行/拓扑缩略图），任意视图 MUST 可下钻至链路/指标/日志/拓扑详情（PRD REQ-M14-U1）。

#### Scenario: 大盘下钻链路
- **WHEN** 用户点击大盘某异常服务卡片
- **THEN** 下钻至该服务链路/指标/告警详情，无需多页面跳转

### Requirement: 服务/实例指标图表
MUST 支持 GET /services/{name} 服务/实例级指标图表（JVM GC/线程/堆、CPU/内存/磁盘/网络、接口 QPS/耗时/错误码），MUST 支持服务级/实例级切换 + 时间范围选择。

#### Scenario: 服务级实例级切换
- **WHEN** 用户在指标页切换服务级/实例级
- **THEN** 展示对应粒度 JVM/资源/接口指标图表（读 npm_online_metric）

### Requirement: 大盘聚合缓存
大盘聚合数据 MUST 缓存 Redis dashboard:overview:cache（60s TTL）+ 多级缓存降 ES 压力，MUST P95<1s。

#### Scenario: 大盘缓存命中
- **WHEN** 多用户频繁刷新大盘
- **THEN** 命中 Redis 缓存（60s TTL），ES 查询量不随用户数线性增长
