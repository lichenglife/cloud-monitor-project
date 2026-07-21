# Java API 规范

适用范围：Spring Boot / Jakarta REST 等 Java HTTP 接口。依据 REST 成熟实践 + Google API Design + OpenAPI 3。

## 1. 统一响应结构（MUST）

所有接口 **MUST** 返回统一信封：
```java
public record Result<T>(int code, String message, T data, String traceId) {
    public static <T> Result<T> ok(T data) { ... }
    public static <T> Result<T> fail(int code, String message) { ... }
}
```
- `code`：业务码（0 / `SUCCESS` 表示成功；非 HTTP 状态码）。
- `message`：可读提示，**禁止**带堆栈 / SQL / 内部路径。
- `data`：业务数据，无数据时 `null`。
- `traceId`：链路 ID（从 MDC 取），便于排障。

## 2. 统一异常处理（MUST）

- **MUST** 用 `@RestControllerAdvice` + `@ExceptionHandler` 集中处理，禁止在各 controller 散落 try-catch。
- 业务异常 → 映射为 `Result.fail(code, msg)`；系统异常 → 500 + 通用提示 + 记 ERROR 日志（脱敏）。
- **禁止**把 `Exception.getMessage()` 原始内容直接返回客户端。
- 校验失败（`MethodArgumentNotValidException`）→ 400 + 字段级错误。

## 3. 路由与版本（MUST）

- 资源用名词复数：`/users`、`/orders/{id}`；**禁止**动词路由（`/getUser`）。
- 版本 **MUST** 放路径前缀：`/api/v1/...`（或 Accept Header，路径优先）。
- 层级不超过 2 级：`/api/v1/orders/{id}/items`。
- 用 HTTP 方法表达动作：GET 只读、POST 新建、PUT 全量改、PATCH 局部改、DELETE 删除。

## 4. 状态码（MUST）

正确使用 HTTP 状态码：200/201/204、400 参数错误、401 未认证、403 无权限、404 不存在、409 冲突、422 校验不过、429 限流、5xx 服务端。接口文档 **MUST** 标注每个错误码含义。

## 5. 入参校验（MUST）

- 用 `@Valid` / Jakarta Validation（`@NotNull`、`@Size`、`@Pattern`）在 DTO 上声明，**禁止**在业务里手写大量 if-null。
- 分页 / 排序 **MUST** 用标准参数：`?page=0&size=20&sort=createdAt,desc`，返回带总数的分页对象。
- 大请求体 **SHOULD** 限制大小；文件上传走独立端点并限流。

## 6. 幂等（SHOULD）

- 写接口（支付 / 下单）**SHOULD** 支持幂等：客户端传 `Idempotency-Key`，服务端去重。
- 查询参数化，禁止拼接 SQL（见安全规范）。

## 7. 文档与契约（MUST）

- **MUST** 用 OpenAPI 3 注解（`@Tag`/`@Operation`/`@Schema`）生成 `openapi.yaml`，随代码同步。
- 公共字段 **MUST** 有示例与说明；枚举字段列出允许值。

## 8. 禁止事项（红线）

- ❌ controller 直接返回 Entity / 异常堆栈。
- ❌ 在 controller 内散落 try-catch 处理业务异常。
- ❌ 动词路由、`/api/getX`。
- ❌ 校验靠手写 if 而非 `@Valid`。

## 检查清单

- [ ] 响应统一为 Result 信封，含 traceId
- [ ] 异常经 @RestControllerAdvice 集中处理，不泄露内部信息
- [ ] 路由名词复数 + /api/v1 前缀
- [ ] 入参 @Valid，分页标准化
- [ ] OpenAPI 注解齐全且同步
