# Java 日志规范

适用范围：Java 应用日志。依据 SLF4J + Logback/Log4j2 最佳实践 + 可观测性惯例。

## 1. 门面与实现（MUST）

- **MUST** 用 SLF4J 门面（`LoggerFactory.getLogger`），禁止直接使用 Logback / Log4j2 具体类。
- 实现统一（Logback 或 Log4j2），配置放 `resources`，**禁止** `System.out` / `System.err` 打业务日志。

## 2. 占位符（MUST）

- **MUST** 用占位符 `log.info("user {} login from {}", uid, ip)`；**禁止**字符串拼接 `"user " + uid + " login"`（即便没开日志也会拼接，浪费且可能 NPE）。
- 异常 **MUST** 作为末参传入：`log.error("fail to pay {}", orderId, ex)`，让框架打印堆栈。

## 3. 级别语义（MUST）

- **ERROR**：影响业务的故障（调用失败、数据不一致、无法恢复）。**MUST** 含 traceId + errorCode + 关键上下文。
- **WARN**：可恢复异常、降级、重试、临界阈值（如连接池接近上限）。
- **INFO**：关键业务节点（下单成功、状态变更）、启动 / 关闭、配置加载。
- **DEBUG**：排查用的详细信息（请求参数、分支命中）。
- **TRACE**：最细粒度（循环内、协议级），**禁止**在生产默认开启。
- **禁止**用 ERROR 刷正常流程；**禁止**用 INFO 打调试细节。

## 4. 上下文与追踪（MUST）

- **MUST** 用 MDC 注入 `traceId` / `userId` / `requestId`，使每条日志带链路信息；请求结束 **MUST** `MDC.clear()`（用 filter / interceptor）。
- 跨线程 **MUST** 传递 MDC（`MDC.getCopyOfContextMap` / `TaskDecorator`），禁止线程池丢失上下文。
- 分布式调用 **SHOULD** 透传 `traceId`（OpenTelemetry / 自研）。

## 5. 脱敏（MUST）

- **禁止**日志打印：密码、Token、完整卡号 / 身份证 / 手机号、签名密钥。
- 敏感字段 **MUST** 掩码（`138****8000`）；日志脱敏 **SHOULD** 在统一切面 / 转换器做，而非每处手写。

## 6. 性能（SHOULD）

- 大对象 / 集合 toString **SHOULD** 避免进日志；低频 DEBUG 用 `log.isDebugEnabled()` 守卫昂贵序列化。
- 高吞吐场景 **SHOULD** 用异步 Appender；**禁止**同步写远程阻塞主流程。
- 日志量 **SHOULD** 设采样 / 限流，避免磁盘打满。

## 7. 结构化（SHOULD）

- 生产 **SHOULD** 输出 JSON（Logstash / JSON encoder），字段化便于采集（含 `traceId`、`level`、`service`、`env`）。
- 禁止无结构的大段文本日志难以检索。

## 8. 禁止事项（红线）

- ❌ `System.out.println` / `printStackTrace` 打日志。
- ❌ 字符串拼接日志、异常不传末参。
- ❌ 日志泄露密钥 / PII。
- ❌ MDC 不清理导致上下文串号。

## 检查清单

- [ ] 用 SLF4J 门面，无 System.out
- [ ] 占位符，异常作末参
- [ ] 级别语义正确，ERROR 含 traceId + errorCode
- [ ] MDC 注入 traceId 且清理 / 跨线程传递
- [ ] 敏感信息已脱敏
- [ ] 生产 JSON 结构化（SHOULD）
