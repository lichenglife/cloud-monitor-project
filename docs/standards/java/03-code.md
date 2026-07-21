# Java 代码规范

适用范围：Java 源码编写。依据 Google Java Style Guide + 阿里巴巴《Java 开发手册》要点。

## 1. 格式化（MUST）

- 使用 `google-java-format` 或 `spotless` 统一格式；行宽 **MUST** ≤ 100（或团队统一值）。
- 缩进 4 空格，**禁止** Tab；文件末尾换行；UTF-8。
- import **MUST** 用 `*` 不展开、`com.google.common.*` 风格；禁止 `import .*` 通配（IDE 配置）。
- 禁止未使用的 import / 未使用的变量（checkstyle / errorprone 拦截）。

## 2. 命名（MUST）

- 类 / 接口 / 枚举：PascalCase（`OrderService`、`PaymentStatus`）。
- 方法 / 变量 / 参数：camelCase（`calculateTotal`）。
- 常量：`static final` 全大写下划线（`MAX_RETRY_COUNT`）。
- 包：全小写，无下划线（见目录规范）。
- 测试类 / 方法：`OrderServiceTest`、`shouldReturnXWhenY`。
- 禁止拼音命名、禁止 `data1`/`tmp`/`a`/`b` 无意义名。

## 3. 类与方法设计（MUST）

- 单一职责：一个类只做一件事；方法 **MUST** ≤ 50 行（建议 ≤ 30），超过 **MUST** 拆分。
- 文件 **SHOULD** ≤ 500 行；超过考虑拆类。
- 接口与实现分离；面向接口编程，依赖注入用构造器注入（禁止字段注入便于测试）。
- 避免 `null`：返回值 **SHOULD** 用 `Optional` 或空集合（非 `null`）；`@Nullable` 显式标注。
- 魔法值 **MUST** 提取为命名常量；禁止散落 `if (status == 3)`。
- 枚举代替常量组；状态 / 类型优先用 enum。

## 4. 集合与字符串（SHOULD）

- 返回集合 **MUST** 不为 `null`（用 `Collections.emptyList()`）。
- 遍历 **SHOULD** 用增强 for / Stream；并发集合用 `ConcurrentHashMap` 等。
- 字符串拼接在循环 / 日志里 **MUST** 用 `StringBuilder` 或日志占位符（见日志规范）。
- `equals` 用 `Objects.equals`；比较常量放前（`"x".equals(y)`）防 NPE。

## 5. 并发（MUST）

- 共享可变状态 **MUST** 加锁或用线程安全结构；明确 `synchronized` / `ReentrantLock` 范围。
- 线程池 **MUST** 显式定义（核心/最大/队列/拒绝策略），**禁止** `Executors.newFixedThreadPool` 无界队列（OOM 风险）。
- `volatile` / `AtomicX` 用于可见性 / 原子计数；禁止随意用 `double-checked locking` 不熟写法。
- 异步任务 **MUST** 有超时与异常兜底。

## 6. 异常（MUST，详见错误处理规范）

- 业务异常用自定义 unchecked（RuntimeException 子类）；不滥用 checked exception。
- `equals`/`hashCode`/`toString` **MUST** 同时重写且基于同一字段。
- 序列化类 **MUST** 显式 `serialVersionUID`。

## 7. 单元测试（MUST，详见补充规范）

- 用 JUnit 5 + Mockito；测试类与被测同包，`<Class>Test`。
- 一个测试只验一个行为；用 `@DisplayName` 描述。
- 禁止依赖真实外部服务（DB / MQ 用 testcontainers / mock）。

## 8. 禁止事项（红线）

- ❌ `System.out.println` / `printStackTrace`（生产日志走 SLF4J）。
- ❌ 魔法值、拼音命名、`null` 当返回值默认。
- ❌ 方法超 50 行不拆分、超大类。
- ❌ 字段注入、无界线程池、`switch` 无 `default`。

## 检查清单

- [ ] 通过 google-java-format / spotless，行宽合规
- [ ] 命名规范，无魔法值 / 拼音
- [ ] 方法 ≤ 50 行，类单一职责
- [ ] 返回集合非 null，避免 null
- [ ] 线程池有界、有拒绝策略
- [ ] 无 System.out / printStackTrace
