# Java 错误处理规范

适用范围：Java 异常设计、全局处理、事务、错误码。详见 API 规范（统一响应）与日志规范。

## 1. 异常体系（MUST）

- 自定义异常继承 `RuntimeException`（unchecked）用于业务错误；**不滥用 checked exception**（避免强制 try-catch 污染调用链）。
- 分层异常：
  - `BizException`（业务可预期错误：参数非法、状态冲突、额度不足）→ 映射为用户可见错误。
  - `SystemException` / `InfraException`（下游 DB / RPC / MQ 失败）→ 归为 5xx。
- 异常 **MUST** 携带稳定 `errorCode`（字符串/数字），便于前端与日志聚合；**禁止**仅用 message 区分。

## 2. 错误码规范（MUST）

- 错误码 **MUST** 集中定义（枚举 / 常量），格式如 `ORDER_NOT_FOUND(100404)`、`PAY_TIMEOUT(200408)`。
- 码空间分段：通用 `0xxxx`、各域 `1xxxx`/`2xxxx`，便于定位模块。
- 同一错误 **MUST** 用同一码；**禁止**同一码表达不同含义。

## 3. 处理规则（MUST）

- **禁止**空 catch（`catch (e) {}`）或仅 `printStackTrace()`；必须处理、记录或重抛。
- **禁止** `catch (Throwable)` / `catch (Exception)` 后吞掉；精确捕获预期异常。
- 捕获后若重抛，**MUST** 保留原因为 `cause`（`throw new BizException(..., e)`），不丢链路。
- **禁止**在 finally 里 return / 抛异常掩盖 try 结果。
- 不应因异常导致资源泄漏：try-with-resources **MUST** 用于 `Closeable` / `AutoCloseable`。

## 4. 事务与回滚（MUST）

- `@Transactional` 方法 **MUST** 指定 `rollbackFor = Exception.class`（Spring 默认仅回滚 RuntimeException，checked 不回滚是常见坑）。
- 事务方法 **MUST** 为 public、自调用失效需避免（通过代理调用）；长事务 / 网络调用 **禁止** 放进 `@Transactional`。
- 跨服务一致性 **SHOULD** 用事务消息 / Saga，禁止在本地事务里同步调远程并期望原子。

## 5. 对外暴露（MUST）

- 所有未捕获异常经 `@RestControllerAdvice` 转为统一 `Result.fail`，返回**稳定的错误码 + 可读 message**。
- **禁止**向外返回 `e.getMessage()` 原文、堆栈、SQL、内部类名。
- 关键错误 **MUST** 记 ERROR 日志（含 traceId、errorCode、上下文，脱敏）。

## 6. 禁止事项（红线）

- ❌ 空 catch / `printStackTrace` 吞异常。
- ❌ `catch (Throwable)` 吞掉。
- ❌ `@Transactional` 不写 `rollbackFor` 且依赖默认。
- ❌ 把原始异常信息直接返回客户端。

## 检查清单

- [ ] 异常继承 RuntimeException，含稳定 errorCode
- [ ] 错误码集中定义、唯一语义
- [ ] 无空 catch，重抛保留 cause
- [ ] `@Transactional(rollbackFor = Exception.class)`
- [ ] 全局异常处理，不泄露内部信息
- [ ] 关键错误记 ERROR 日志（脱敏 + traceId）
