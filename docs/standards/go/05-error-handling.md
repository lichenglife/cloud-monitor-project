# Go 错误处理规范

适用范围：Go 错误设计、包装、panic、错误码。详见 API 规范（统一响应）与日志规范。

## 1. error 即值（MUST）

- 错误是普通值，**MUST** 被检查。函数返回 `error` 时调用方 **MUST** 处理或显式忽略（`_ = doX() // 已知可忽略：...`）。
- **禁止** `_ = err` 不带注释地吞掉；**禁止**仅打印不处理导致错误被忽略。

## 2. 包装与链（MUST）

- 向上层传递时 **MUST** 用 `fmt.Errorf("操作上下文: %w", err)` 保留原始错误链。
- 判断具体错误 **MUST** 用 `errors.Is(err, ErrNotFound)` / `errors.As(err, &target)`；**禁止**用字符串 `strings.Contains(err.Error(), "...")` 判断。
- 在边界（handler / 入口）将错误转为对用户友好的消息与错误码（见 API 规范）。

## 3. 错误类型（MUST）

-  sentinel 错误：`var ErrNotFound = errors.New("not found")`，用于可判定错误。
- 自定义错误类型：需携带字段时用结构体实现 `error` 接口（`type ValidationError struct{ Field, Msg string }`）。
- 业务错误 **SHOULD** 携带稳定 `Code`（字符串/数字），便于聚合与前端分支。

## 4. panic（MUST）

- **禁止**用 `panic` 控制正常业务流程（参数错误、资源缺失应返回 error）。
- `panic` 仅用于：程序无法继续（配置致命错误）、`init` 失败、真不可恢复。
- 每个长期运行的服务 / goroutine 入口 **MUST** 有 `recover`（全局中间件 / `defer recover()`），转 500 / 错误日志，**禁止** panic 导致进程退出。
- 库代码 **MUST NOT** 自行 panic 把崩溃抛给调用方。

## 5. 上下文与可读性（SHOULD）

- 错误信息 **MUST** 小写开头、无句号、说明"什么失败 + 为什么"，不重复调用方已知信息。
- **禁止**用错误携带敏感信息（密码、Token）外泄（见日志 / API 规范）。

## 6. 对外暴露（MUST）

- 所有 handler 错误经中间件 / 统一封装转为 `Result.Fail(code, msg)`；`msg` 为可读提示，**禁止**透传 `err.Error()` 原文。
- 关键错误 **MUST** 记 ERROR 日志（含 traceId、错误码、上下文，脱敏）。

## 7. 禁止事项（红线）

- ❌ 忽略 error 不注释。
- ❌ 用 `strings.Contains(err.Error())` 判断错误类型。
- ❌ panic 控制正常流程（除真不可恢复）。
- ❌ 透传 `err.Error()` 给客户端。

## 检查清单

- [ ] error 已处理或显式忽略并注释
- [ ] 用 %w 包装，errors.Is/As 判断
- [ ] 业务错误含稳定 Code
- [ ] panic 仅不可恢复，入口有 recover
- [ ] 对外不暴露原始 error 文本
- [ ] 关键错误记 ERROR 日志（脱敏 + traceId）
