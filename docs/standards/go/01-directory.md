# Go 目录规范

适用范围：Go 项目目录结构与包组织。依据 Standard Go Project Layout + Uber Go Style Guide + Go 官方惯例。

## 1. 标准布局（MUST）

```
<root>/
  cmd/
    <app>/
      main.go              # 仅做装配：解析配置、注入依赖、启动
  internal/                # 私有代码，禁止外部 import
    <pkg>/...
  pkg/                     # 可被外部 import 的公共库（谨慎使用）
  api/                     # 对外 API 契约：.proto / openapi.yaml
  configs/                 # 配置文件样例
  deployments/             # 部署清单（k8s / docker）
  test/                    # 额外测试数据 / 集成测试
  docs/                    # 文档
  scripts/                 # 构建 / 运维脚本
  go.mod / go.sum
```

- `cmd/<app>/main.go` **MUST** 保持极薄：只装配，**禁止**放业务逻辑。
- `internal/` **MUST** 放不想被外部依赖的包；编译器强制保护。能用 internal 就不要用 pkg。
- `pkg/` 仅放**确实要对外复用**的库；禁止把整个业务塞进 pkg。

## 2. 包命名（MUST）

- 包名 **MUST** 全小写、单数、无下划线 / 驼峰 / 连字符（`user`，非 `users`/`UserUtil`）。
- 包名 **MUST** 简洁且有意义，避免 `common`/`util`/`base` 等空泛名；工具按领域分（`strutil`、`httputil`）。
- 避免与标准库同名（`base64` 等）引发导入冲突。
- 一个目录一个包；禁止同目录多包（测试包 `xxx_test` 除外）。

## 3. 分层与依赖方向（MUST）

- 依赖方向单向、由外向内：handler → service → repository；domain / 实体在最内，不依赖外层。
- 禁止循环依赖；出现循环 **MUST** 抽接口或上移公共依赖。
- 跨层传递用明确的 DTO / 领域对象；**禁止**把 DB 行结构（如 GORM Model）直接作为 API 响应。

## 4. 模块与版本（MUST）

- `go.mod` **MUST** 声明 module path（通常含仓库路径 `github.com/org/repo`）与 go 版本。
- 依赖 **MUST** 锁版本，`go.sum` 入库；禁止 `replace` 指向本地未提交路径长期存在。
- 多模块用 workspace（`go.work`）或子 module，按边界划分。

## 5. 文件组织（SHOULD）

- 一个文件一个职责；类型与其方法放同文件（或同包合理拆分）。
- 测试文件 `<x>_test.go` 同包；黑盒测试用 `package x_test`。
- 编译产物 / 临时文件 **MUST** 加入 `.gitignore`（二进制、`*.test`）。

## 6. 禁止事项（红线）

- ❌ 在 `cmd/` 写业务逻辑。
- ❌ 把整个业务放进 `pkg/` 暴露；滥用 `common`/`util`。
- ❌ 同目录多包（除 `_test`）。
- ❌ 循环依赖不处理；`go.sum` 不入库。

## 检查清单

- [ ] 使用 standard layout，main.go 仅装配
- [ ] internal/pkg 使用正确，无空泛包名
- [ ] 依赖方向单向、无循环
- [ ] go.mod/go.sum 完整，module path 正确
- [ ] DB 行结构不直出 API
