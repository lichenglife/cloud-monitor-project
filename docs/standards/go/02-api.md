# Go API 规范

适用范围：Go HTTP（net/http / gin / echo 等）与 gRPC 接口。依据 REST 成熟实践 + gRPC + OpenAPI 3。

## 1. 统一响应结构（MUST）

定义统一返回：
```go
type Result struct {
    Code    int    `json:"code"`    // 业务码，0 成功
    Message string `json:"message"`
    Data    any    `json:"data,omitempty"`
    TraceID string `json:"traceId"` // 从 context 取
}
func OK(data any) Result { ... }
func Fail(code int, msg string) Result { ... }
```
- HTTP 状态码表达传输层结果；`code` 表达业务结果。
- **禁止**把 `error.Error()` 字符串直接作为 `Message` 返客户端（见错误处理规范）。

## 2. 中间件与统一处理（MUST）

- 用中间件做：日志、recover（防 panic 挂服）、traceId 注入、限流、CORS、鉴权。
- **MUST** 有全局 `recover` 中间件：捕获 panic → 记 ERROR 日志 → 返回 500 统一结构；**禁止**任何 handler panic 直接 500 裸奔。
- 校验用 `struct tag`（binding / validator），**禁止**在 handler 手写大量 if 校验参数。
- 路由按版本分组：`/api/v1/...`。

## 3. 路由与版本（MUST）

- 资源用名词复数：`/users`、`/orders/{id}`；**禁止**动词路由。
- 用 HTTP 方法表意：GET/POST/PUT/PATCH/DELETE。
- 分页 / 排序 **MUST** 标准化：`?page=1&page_size=20&sort=created_at,desc`，返回带总数。
- 层级 ≤ 2 级。

## 4. gRPC 规范（SHOULD）

- 接口定义 **MUST** 在 `api/*.proto`，命名 `Service`/`Method` PascalCase，消息 `XxxRequest`/`XxxResponse`。
- 错误用 `status.Error(codes.x, msg)`，客户端用 `status.Code(err)` 判断；**禁止**用 `errors.New` 丢错误码。
- 流式接口明确背压与取消（context）。

## 5. 超时与取消（MUST）

- 每个请求 / 下游调用 **MUST** 带 `context.Context` 与超时；**禁止**无 deadline 的调用。
- handler 入口 **MUST** 建 context 并随调用链透传；长任务监听 `<-ctx.Done()`。

## 6. 幂等（SHOULD）

- 写接口 **SHOULD** 支持幂等键（`Idempotency-Key` 头或业务去重键）。

## 7. 文档与契约（MUST）

- REST **MUST** 有 OpenAPI 3（`api/openapi.yaml`）；gRPC **MUST** 有 `.proto` 注释。
- 随代码同步，CI 校验契约与实现一致。

## 8. 禁止事项（红线）

- ❌ 裸 panic 无 recover 中间件。
- ❌ 把 error 原文直接返回客户端。
- ❌ handler 内手写大量参数校验。
- ❌ 无 context / 无超时的下游调用。

## 检查清单

- [ ] 响应统一 Result，含 traceId
- [ ] 全局 recover + 日志 + 限流中间件就绪
- [ ] 路由名词复数 + /api/v1，参数用 tag 校验
- [ ] context 透传 + 超时
- [ ] gRPC 用 status code；OpenAPI/proto 同步
