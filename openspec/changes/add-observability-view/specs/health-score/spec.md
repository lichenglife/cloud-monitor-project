## ADDED Requirements

### Requirement: 服务健康评分（4 维惩罚制，DD-03 §9.1）
MUST 按公式 health=clamp(100-(E×0.35+L×0.25+R×0.20+A×0.20),0,100) 计算，E 错误率/L 时延劣化/R 资源使用率/A 告警事件（P0=40/P1=20/P2=10/P3=5），MUST 定时 5min 计算并缓存 Redis health:score（1m TTL），MUST 分级 90~100 健康/70~89 关注/40~69 警告/<40 严重。

#### Scenario: 健康评分计算
- **WHEN** 定时任务每 5min 触发某服务评分
- **THEN** 按 4 维惩罚制计算，结果缓存 Redis，前端读缓存不实时计算

#### Scenario: 健康分异常下钻
- **WHEN** 某服务健康分为 CRITICAL(<40)
- **THEN** 点击跳转该服务拓扑/链路/告警视图（PRD REQ-M14-E1）

### Requirement: 健康评分历史与扣分明细
MUST 写 t_health_score_history 历史记录（含三维得分 + 权重快照），MUST 展示扣分明细（哪项指标拖后腿）+ 历史趋势（近 7d/30d）。

#### Scenario: 扣分明细展示
- **WHEN** 用户查看某服务健康评分详情
- **THEN** 展示三维雷达图 + 扣分明细（错误率/时延/资源/告警各项得分）+ 历史趋势
