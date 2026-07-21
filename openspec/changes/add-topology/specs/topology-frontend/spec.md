## ADDED Requirements

### Requirement: G6 WebGL 万级节点渲染（D2）
前端拓扑页 MUST 用 @antv/g6 v5 WebGL 渲染器 + 视口 LOD + 按 scope 聚类收起 + Web Worker 布局，支撑万级节点不卡顿，方案 B（默认聚类收起、仅下钻展开）MUST 作为降级兜底（超阈值自动切聚类）。

#### Scenario: 万级节点流畅交互
- **WHEN** 全局拓扑含万级节点
- **THEN** WebGL 渲染 + Worker 布局，缩放/平移/下钻流畅，主线程不卡顿

#### Scenario: 超阈值降级聚类
- **WHEN** 节点数超 WebGL 渲染阈值
- **THEN** 自动切方案 B 聚类收起，仅下钻展开，保可用性

### Requirement: 纵向下钻与指标叠加
MUST 支持节点下钻（服务->实例列表->容器->主机资源指标），边/节点 MUST 可叠加错误率/P95/吞吐指标，MUST 支持时间回放（选历史 snapshot_time 回放拓扑演变）。

#### Scenario: 节点下钻至主机
- **WHEN** 用户点击服务节点 -> 实例 -> 主机
- **THEN** 展示该主机 CPU/内存/磁盘/网络资源指标

### Requirement: CMDB 元数据补全与权限过滤
节点/边 MUST 补全 CMDB 元数据（机房/系统/负责人），拓扑 MUST 按 CMDB 数据权限过滤可见节点/边，无权限置灰/隐藏。

#### Scenario: 权限过滤拓扑可见范围
- **WHEN** 用户查看全局拓扑但仅有 A 系统权限
- **THEN** 仅展示 A 系统及上下游相关节点/边，其他置灰/隐藏
