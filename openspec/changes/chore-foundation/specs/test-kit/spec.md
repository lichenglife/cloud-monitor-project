## ADDED Requirements

### Requirement: 测试铺底组件骨架
`npm-trace-testkit`（SpringBoot 3 + Java 17，入 DD-13 兼容矩阵）MUST 初始化为独立可部署应用，经 SkyWalking Agent 注入产生 OTLP 上报 Collector（biz/dmz 双路）->Kafka（上海 DC1）->聚合->ES，与真实接入应用链路一致；含 S1-S8 八场景骨架（db/mq/http/error/fault/log/deeptrace/resource）通过配置/接口动态开关，可调 QPS/错误率/慢调用比例/Trace 深度。

#### Scenario: 铺底数据驱动页面验收
- **WHEN** 集成/性能/验收测试需真实链路数据（错误/慢/跨中心）
- **THEN** testkit 按 S1-S8 场景产出对应链路，覆盖全部页面功能验证，消除"无真实流量难触发错误/慢链路"盲区

#### Scenario: 场景可编排可回放
- **WHEN** 回归测试需重复某场景
- **THEN** 经配置/接口开启指定 scenario 并设参数（QPS/错误率/Trace 深度），产出可重复可回放的链路数据

### Requirement: testkit 链路串联与上报通路
testkit 各 scenario 产生的 span MUST 经 `traceId` 串联形成可下钻瀑布图树，上报通路与真实接入应用完全一致（Agent->Collector->Kafka->聚合->ES 6 索引）。

#### Scenario: 多层级调用形成可下钻链路
- **WHEN** scenario-deeptrace 触发 A->B->C->DB 多层级调用
- **THEN** 产出的 Trace 经 traceId 串联，可在瀑布图下钻至方法/SQL 级，覆盖拓扑下钻与火焰图场景
