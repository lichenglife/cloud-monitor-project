# Go 日志规范

适用范围：Go 应用日志。依据官方 `log/slog`（Go 1.21+）或 `zap` 最佳实践 + 可观测性惯例。

## 1. 结构化日志（MUST）

- **MUST** 用结构化日志：`log/slog`（stdlib）或 `zap`；**禁止** `log.Printf` / `fmt.Println` 打业务日志（仅极简工具可用 stdlib `log`）。
- 使用键值对属性：`slog.Info("order paid", "order_id", id, "amount", amt)`；**禁止**字符串拼接。
- 生产 **SHOULD** 用 JSON handler（`slog.NewJSONHandler`），字段化便于采集。

## 2. 级别语义（MUST）

- `ERROR`：业务故障、下游失败、不可恢复；**MUST** 含 traceId + 错误码 + 上下文。
- `WARN`：降级、重试、临界阈值。
- `INFO`：关键节点（启动、下单成功、状态变更）。
- `DEBUG`：排查细节；`TRACE` 最细。**禁止**用 ERROR 刷正常流程。
- `slog` 默认级别 INFO；按环境调（prod=INFO，排障可 DEBUG）。

## 3. 上下文与追踪（MUST）

- **MUST** 用 `context.Context` 携带 `traceId` / `requestID`，通过 `slog.With("trace_id", ...)` 或自定义 `context.Handler` 注入每条日志。
- 请求入口建 logger 并随 `ctx` 传递；**禁止**跨请求串号。
- 跨 goroutine **MUST** 传递 ctx（含 logger），禁止新 goroutine 丢失上下文。

## 4. 错误记录（MUST）

- 记录错误 **MUST** 附原始 error：`slog.Error("pay failed", "err", err, "order_id", id)`（slog 自动展开，或用 `slog.Any("err", err)`）。
- **禁止**仅 `slog.Error(msg)` 不附 err，导致无法定位链。

## 5. 脱敏（MUST）

- **禁止**日志打印：密码、Token、完整卡号 / 身份证 / 手机号、签名密钥。
- 敏感字段 **MUST** 掩码；脱敏 **SHOULD** 在统一 handler / 中间件做。

## 6. 性能（SHOULD）

- 高吞吐用 `zap` 或 `slog` + 异步；避免热路径昂贵序列化（用 `slog.LogAttrs` / `Attr` 惰性）。
- 大对象 **SHOULD** 不进日志；日志量 **SHOULD** 采样 / 限流。

## 7. 禁止事项（红线）

- ❌ `fmt.Println` / `log.Printf` 打业务日志。
- ❌ 字符串拼接日志。
- ❌ 记错误不附 `err`。
- ❌ 日志泄露密钥 / PII。
- ❌ 跨 goroutine 丢失 traceId 上下文。

## 检查清单

- [ ] 用 slog/zap 结构化，无 Println
- [ ] 键值对属性，无拼接
- [ ] 级别语义正确，ERROR 含 traceId + 错误码 + err
- [ ] ctx 传递 traceId，跨 goroutine 不掉
- [ ] 敏感信息脱敏
- [ ] 生产 JSON 输出（SHOULD）
