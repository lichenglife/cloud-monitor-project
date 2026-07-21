# Go 代码规范

适用范围：Go 源码编写。依据 Uber Go Style Guide + Go Code Review Comments + Effective Go。

## 1. 格式化（MUST）

- **MUST** 通过 `gofmt` / `goimports`（CI 门禁）。
- 用 `golangci-lint` 聚合 lint（errcheck / govet / staticcheck / revive 等）。
- 行长 **SHOULD** ≤ 120；import 分组（标准库 / 第三方 / 内部，空行分隔）。

## 2. 命名（MUST）

- 包名小写单数无下划线（见目录规范）。
- 导出标识符 PascalCase；非导出 camelCase。
- 缩写 **MUST** 全大写：`HTTPServer`、`userID`（非 `UserId`）、`URL`、`API`。
- 接口名 **SHOULD** 以 `-er` 结尾（`Reader`、`Closer`）；单方法接口小而专。
- 接收者名 **MUST** 简短一致（类型首字母，如 `s *userService`），同一类型所有方法用同一名。
- 禁止拼音、禁止 `tmp`/`data1` 无意义名。

## 3. 错误（MUST，详见错误处理规范）

- 函数返回 `error` **MUST** 被处理或显式忽略（`_ = f()` 并注释原因）；**禁止**默默丢弃。
- 包装用 `fmt.Errorf("...: %w", err)` 保留链；判断用 `errors.Is` / `errors.As`。
- **禁止**用 panic 控制正常流程（仅 init / 真不可恢复时用，且 main 应 recover）。

## 4. 接口与组合（MUST）

- 接口 **SHOULD** 由**消费方**定义、小而隐式（接受接口、返回结构体）。
- 禁止为所有实现预定义巨型接口；用组合复用。
- 接收者：变更内部状态用指针接收者；值接收者仅用于小型不可变值（如 `type Point struct`）。

## 5. 并发（MUST）

- **禁止**共享内存，用 channel 通信；共享状态用 `sync.Mutex` / `atomic`。
- goroutine **MUST** 有明确退出路径（context / channel 关闭），防泄漏。
- **MUST** 用 `errgroup` / `context` 管理并发取消；禁止无限起 goroutine 无上限。
- 锁粒度清晰；**禁止**在持有锁时做 IO / 调用外部。

## 6. 零值与默认值（SHOULD）

- 利用零值：未初始化即有合理默认（slice `nil` 可 range、`sync.Mutex` 零值可用）。
- 返回错误时，成功值 **SHOULD** 给零值或合理默认，禁止返回部分有效 + error 混用造成歧义。
- `defer` 用于释放（关连接 / 解锁），**MUST** 在获取资源后立即 defer。

## 7. 测试（MUST，详见补充规范）

- **MUST** 用 table-driven tests；`t.Run` 命名子用例。
- 基准用 `Benchmark` + `b.ReportAllocs()`；**禁止**依赖真实外部服务（用 httptest / 容器 / mock）。
- mock 优先接口（gomock / 手写轻量假实现）。

## 8. 禁止事项（红线）

- ❌ `fmt.Println` 充当生产日志。
- ❌ 忽略 error 不处理不注释。
- ❌ panic 控制正常流程；goroutine 泄漏。
- ❌ 持有锁做 IO；巨型预定义接口；未 `gofmt`。

## 检查清单

- [ ] gofmt / goimports 通过，golangci-lint 绿
- [ ] 命名规范（缩写全大写、接收者一致）
- [ ] error 已处理或显式忽略，wrap 保留链
- [ ] 接口消费方定义、小而专
- [ ] 并发有退出路径，无锁内 IO
- [ ] table-driven 测试，无外部依赖
