# 全链路追踪平台 — SDK 与探针（Agent）治理方案专题（DD-13，待评审）

| 项 | 内容 |
|---|---|
| 文档版本 | v0.2（评审决议落地：D11 Agent 资源红线=方案A 统一基线+重负载白名单） |
| 对应任务 | 3.4（Agent 治理专题） |
| 依据 | DD-01（/agents/* 接口、OTLP 协议）、DD-02（t_agent_instance/t_agent_config/t_agent_registration）、DD-07（采集）、DD-14（脱敏/采样） |
| 原则遵循 | 资源消耗红线、可靠性（防 OOM/自动降级）、零信任（注册鉴权）、可维护性（热更新/批量升级）、向后兼容、控制基数 |

> 大厂实践中 Agent 的海量部署与升级治理往往比服务端更难——本专题为必交付重点，覆盖"接得进、跑得稳、升得快、降得下"。

---

## 1. Agent 治理全景

| 治理域 | 关键能力 | 对应接口/数据 |
|---|---|---|
| 兼容治理 | 语言/框架兼容矩阵、版本支持范围 | `/agents/compatibility`、DD-01 |
| 注入治理 | K8s initContainer / VM 参数、自动注入 | t_agent_instance |
| 资源治理 | CPU/内存/线程上限、防 OOM、熔断 | agent:config(#1)、DD-07 |
| 配置治理 | 热配置下发（采样率/最大TPS）、版本 | config-center、t_agent_config |
| 升级治理 | 批量灰度升级、回滚 | `/agents/upgrade` |
| 注册治理 | 握手/令牌/心跳、存活判定 | t_agent_registration |
| 安全治理 | 令牌鉴权、传输 TLS、最小权限 | 零信任、CMDB 兜底 |

## 2. 兼容性矩阵（治理基线）

| 语言 | 版本 | 框架 | 接入方式 | 备注 |
|---|---|---|---|---|
| Java | 8 / 11 / 17 | Spring Boot 1.x/2.x/3.x | SkyWalking Agent | 字节码增强，需验证插件兼容 |
| Java | 8 / 11 / 17 | Dubbo 2.x/3.x、gRPC | SkyWalking Agent | rpc 插件 |
| Go | 1.18+ | gin / grpc-go / 标准 net/http | OTel SDK/Agent | 原生 OTLP 导出 |
| 其他 | — | 任意（OTel 生态） | OTel SDK 手动 | 遵循 DD-03.5 |

- **矩阵维护**：`/agents/compatibility` 提供查询；新版本上线前入矩阵需通过兼容性测试（CI 卡点）。
- **不支持范围**：明确拒绝列表（如已 EOL JDK 6/7），避免踩坑，治理前置。

## 3. 无侵入 / 低侵入机制（防类冲突）

- **字节码增强原理**：SkyWalking Agent 基于 Java Agent + ByteBuddy/ASM 在类加载期织入埋点逻辑，不修改业务源码（无侵入）。
- **ClassLoader 隔离**：Agent 自身依赖（ByteBuddy、OTel SDK、gRPC 客户端）置于**独立 ClassLoader**（`AgentClassLoader`），与业务 AppClassLoader 隔离，避免 `NoSuchMethodError` / `ClassNotFoundException` 类冲突。
- **插件热插拔**：各组件插件（http/db/mq）独立 jar，按需加载，冲突插件默认禁用并告警。
- **Go 侧**：OTel SDK 编译进业务或 sidecar，避免运行时冲突。

## 4. 资源开销控制（红线，呼应设计原则"资源消耗红线"）

| 指标 | 红线（v1） | 监控/熔断 |
|---|---|---|
| Agent CPU 占用 | **< 1%**（单核占比，可配） | 超阈持续 30s → 自动降采样/关闭采集 |
| Agent 内存占用 | **< 128MB**（常驻，含缓冲） | 接近上限 → 触发 GC 压力告警 + 降级 |
| 线程数 | ≤ 业务线程 5% | 超阈告警 |
| 采集丢包率 | < 0.1% | DD-05 黄金指标、DD-06 告警 |

- **评审决议 D11=方案A（统一基线 + 白名单）**：采用**统一资源红线**（CPU<1% / 内存<128MB / 线程≤业务 5% / 丢包<0.1%），**不按应用分级**；对**重负载应用**（批处理/高吞吐服务）以**白名单机制**单独调高上限，白名单经 `t_agent_config` 维度管控，避免影响轻量应用。方案 B（按应用分级）不采用（治理复杂、易漂移），方案 C（统一不变）不采用（无法兜底重负载场景）。

- **防 OOM 设计**：
  - 采样/缓冲有界队列（RingBuffer），满则丢弃并计数（绝不无界增长）；
  - 批量导出（batch size / 定时 flush），避免瞬时大对象；
  - 长时间阻塞（Collector 不可达）时本地磁盘缓冲有上限，超限丢旧保新并告警。
- **资源可观测**：Agent 自身指标（CPU/内存/线程/丢包）经 OTLP 回流本平台（Dogfooding，DD-05），先于业务方暴露异常。

## 5. 配置热更新与降级

- 热配置源 `t_agent_config` → Redis `agent:config:{clusterId}`(#1) + `config:version`(#11)，Agent Watch 拉取 **≤30s 生效**（采样率/最大 TPS/采样模式）。
- **极端场景自动降级**：
  - 业务 CPU 打满（宿主 CPU > 95% 持续 N 秒）→ Agent **自动熔断**（暂停采集）或降采样，优先保业务；
  - 内存逼近红线 → 丢弃缓冲、降采样；
  - Collector 不可达 → 本地磁盘缓冲（有界）+ 心跳异常标记。
- **手动降级**：`/agents/config` 可下发 `enabled=false` 全量关闭某集群采集（逃生通道）。

## 6. 批量升级与回滚（海量治理核心）

- **灰度发布**：`/agents/upgrade` 支持按 `cluster_id` / `engineering_id` 范围推送目标版本，先小流量（如 1 个 Pod / 1 台 VM）验证，再分批（金丝雀 1% → 10% → 100%）。
- **回滚**：升级异常（心跳丢失率突增 / 资源超红线 / 业务报错）自动回滚至稳定版；支持一键回滚。
- **版本管理**：`t_agent_instance.version` 记录运行时版本；管理页（DD-15）展示版本分布与异常版本红闪。
- **停机兼容**：升级期间不丢数据（缓冲 + 重连），保证链路连续性。

## 7. 注册与存活治理

- **握手/令牌**：Agent 启动向 `agent-mgr` 注册，签发 `t_agent_registration.token`，后续上报携带令牌鉴权（零信任）。
- **心跳**：周期心跳 `PushAgentHeartbeat`；`last_heartbeat` 超 `heartbeat_timeout`（默认 60s，可配）标记离线。
- **存活看板**：DD-15 Agent 治理页展示在线率、版本、资源、丢包，离线实例告警（DD-06 P1）。

## 8. 安全治理

- 注册令牌 + 传输 TLS（零信任）；Collector 仅接受鉴权 Agent（DD-07 §4）。
- 最小权限：Agent 仅能上报，无平台管理权限（CMDB 兜底，决议 D）。
- 敏感配置（令牌）加密存储（`t_agent_registration`，AES-256-GCM，见 DD-09）。

## 9. 设计原则遵循声明

- 资源消耗红线✅（CPU<1%/内存<128M/线程上限 + 监控熔断） 可靠性✅（有界缓冲防 OOM、自动降级、心跳存活） 零信任✅（令牌+TLS+最小权限） 可维护性✅（热更新、灰度升级、回滚、兼容矩阵） 向后兼容✅（插件隔离、版本管理） 控制基数✅（Agent 指标低基数回流）。
- **待评审 / 已决议**：
  - ~~1. 资源红线数值与是否按应用分级~~ → **评审决议 D11=方案A：统一基线（CPU<1%/内存<128M）+ 重负载应用白名单调高，不按应用分级**；
  - 2. 自动降级触发条件（CPU 95% 持续 N 秒）阈值与恢复策略；
  - 3. 灰度升级批次比例与暂停/回滚判定指标（心跳丢失率阈值、资源超红线持续时长）；
  - 4. 海量 Agent（5000+ 工程）配置下发的 Redis 压力与 Watch 长连接数评估（是否需要分片/分组推送）。

