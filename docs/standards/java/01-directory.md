# Java 目录规范

适用范围：Java（Maven / Gradle）项目目录结构与包组织。依据行业通用布局 + 阿里巴巴《Java 开发手册》。

## 1. 标准布局（MUST）

### 单模块（Maven）
```
src/
  main/
    java/<base-package>/...      # 源码
    resources/                   # 配置、模板、静态资源
    filters/                     # 环境相关资源过滤（可选）
  test/
    java/<base-package>/...      # 测试源码
    resources/                   # 测试资源
pom.xml
```
### 多模块（MUST）
```
<root>/
  pom.xml                        # parent，packaging=pom
  module-api/                    # 接口 / DTO / 契约
  module-service/                # 业务逻辑
  module-infrastructure/         # 外部依赖适配（DB、MQ、RPC）
  module-web/                    # 启动入口 / 控制器
```
- 模块按**业务能力**而非技术层切分优先；禁止"utils 大杂烩"模块。
- 公共契约（DTO / 枚举 / 异常）**SHOULD** 下沉到 `module-api` 或 `common`，避免循环依赖。

## 2. 包命名（MUST）

- base-package **MUST** 为反向域名：`com.company.product`（全小写，无下划线 / 连字符）。
- 包名使用单数名词；按层或按领域组织，二选一并保持一致：
  - 按层：`controller` / `service` / `repository` / `domain` / `config` / `common`
  - 按领域：`order` / `user` / `payment`，每域内再分层
- **禁止** `default` 包（无包名）。
- 工具类放 `common.util` 或 `common.util.*`，且每个工具方法必须有明确职责，禁止 `MyUtils` 式垃圾桶。

## 3. 分层与依赖方向（MUST）

- 推荐依赖方向：`web → service → domain ← infrastructure`（依赖倒置，domain 不依赖 infrastructure）。
- `controller` 只做参数解析 / 校验 / 装配，不含业务；业务逻辑在 `service` / `domain`。
- 数据访问在 `repository` / `dao`，禁止在 controller 直接调 SQL。
- 跨层 DTO **MUST** 做转换（如 `Entity → Response`），禁止把 JPA Entity 直接序列化出接口。

## 4. 资源与配置（MUST）

- 环境相关配置放 `src/main/resources/application-<env>.yml`，**禁止**把 prod 明文配置入库。
- 静态资源 / 模板放 `resources/static`、`resources/templates`。
- 编译产物 `target/` **MUST** 加入 `.gitignore`。

## 5. 源码组织（SHOULD）

- 一个 .java 文件一个顶级 public 类，文件名与类名一致。
- 类内成员顺序：**静态常量 → 静态变量 → 实例变量 → 构造器 → 公共方法 → 私有方法**。
- 测试与被测类同包路径，命名 `<Class>Test` 或 `<Class>IT`（集成）。

## 6. 禁止事项（红线）

- ❌ `default` 包。
- ❌ 在 `web` 层写业务、在 `controller` 拼 SQL。
- ❌ 把 JPA Entity / 内部模型直接作为 API 响应体。
- ❌ `target/`、`.idea/`、`*.iml` 入库。

## 检查清单

- [ ] 目录符合单/多模块布局
- [ ] 包名为反向域名，无 default 包
- [ ] 依赖方向正确，无循环依赖
- [ ] API 响应经 DTO 转换
- [ ] target/ 已忽略，prod 配置未明文入库
