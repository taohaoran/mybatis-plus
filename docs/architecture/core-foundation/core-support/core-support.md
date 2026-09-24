# 核心支撑（core-support）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开 `core/{enums,exceptions,spi,assist,plugins}/`：SQL 方法枚举与模板、统一异常、兼容 SPI、SQL 运行门面、拦截器忽略策略。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| SQL 方法枚举 | `SqlMethod`：枚举 BaseMapper 全部内置方法（insert/update/delete/select/逻辑删除），每个绑定 SqlTemplate 模板 | `enums/SqlMethod.java:26` |
| SQL 模板渲染 | `SqlTemplate.of2..of6`：lambda 模板按占位符 a-f 拼 SQL 字符串 | `enums/SqlMethod.java:30,142-163` |
| 关键字枚举 | `SqlLike`/`SqlKeyword`/`WrapperKeyword`：SQL 片段关键字常量 | `enums/SqlLike.java:37` |
| 统一异常 | `MybatisPlusException`：RuntimeException 子类 | `exceptions/MybatisPlusException.java:41` |
| 兼容 SPI | `CompatibleHelper`：静态块 ServiceLoader 加载 `CompatibleSet`，多实现取最后一个 | `spi/CompatibleHelper.java:28,34` |
| 兼容能力集 | `CompatibleSet`：SPI 接口，由 extension 提供实现 | `spi/CompatibleSet.java:75` |
| SQL 运行接口 | `ISqlRunner`：无实体直接跑 SQL 的门面（insert/delete/update/selectList/count/page） | `assist/ISqlRunner.java:37` |
| SQL 运行基类 | `AbstractSqlRunner`：ISqlRunner 抽象实现，委托注入的 SqlRunner Mapper | `assist/AbstractSqlRunner.java:151` |
| 拦截器忽略策略 | `InterceptorIgnoreHelper`：按 @InterceptorIgnore 或手动 ThreadLocal 控制租户/乐观锁等插件忽略 | `plugins/InterceptorIgnoreHelper.java:34` |
| 忽略策略模型 | `IgnoreStrategy`：租户/乐观锁/动态表名等开关的 builder | `plugins/IgnoreStrategy.java:35` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `SqlMethod` | `SqlMethod.java:26` | 内置 CRUD 方法的元数据（method 名 + desc + sqlTemplate） |
| `ISqlRunner` | `ISqlRunner.java:37` | 裸 SQL 运行门面，常量名空间指向 SqlRunner Mapper |
| `AbstractSqlRunner` | `AbstractSqlRunner.java:151` | ISqlRunner 实现，持有 SqlSession |
| `InterceptorIgnoreHelper` | `InterceptorIgnoreHelper.java:34` | 插件忽略策略中枢（静态 ConcurrentHashMap 缓存 + ThreadLocal） |
| `IgnoreStrategy` | `IgnoreStrategy.java:35` | 各插件忽略开关 |
| `CompatibleHelper` | `CompatibleHelper.java:28` | 兼容 SPI 加载器 |
| `CompatibleSet` | `CompatibleSet.java:75` | 兼容 SPI 接口 |
| `MybatisPlusException` | `MybatisPlusException.java:41` | 统一异常 |

## 3. 关键调用链

**内置 SQL 模板拼装（SQL 注入期）**：

1. sql-injector 叶子按 `SqlMethod.INSERT_ONE` 取模板（`SqlMethod.java:30`）。
2. `INSERT_ONE.format(tableName, columns, values)`（`:147`）→ 内部 `SqlTemplate.SqlTemplate3.format`（`:148`）把 a/b/c 拼进 `<script>INSERT INTO a (b) VALUES c</script>`。
3. 生成的 SQL 脚本注册为 MappedStatement，供运行期调用。

**拦截器忽略判定**：

1. 插件（租户/乐观锁）在拦截 SQL 前查 `InterceptorIgnoreHelper`：先看 ThreadLocal 手动策略（`:58 handle`），再查 `IGNORE_STRATEGY_CACHE`（`:40`，按 msId/className 缓存 @InterceptorIgnore 解析结果）。
2. `IGNORE_STRATEGY_LOCAL`（`:44`）须手动 `clearIgnoreStrategy()`（`:65`）或用 `execute(strategy, supplier)` 包裹。

**兼容 SPI 加载**：

1. `CompatibleHelper` 静态块（`:34`）用 `ServiceLoader.load(CompatibleSet.class)`（`:35`）扫描 META-INF/services，多实现时取最后一个并告警（`:43`）。
2. 调用方 `getCompatibleSet()`（`:74`）在无实现时 `Assert` 抛错。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| SqlMethod 模板 | 内置常量；逻辑删除用 UPDATE 替代 DELETE（LOGIC_*） | `SqlMethod.java:61-91` |
| 兼容实现 | 通过 ServiceLoader 自动发现，或 `setCompatibleSet` 手动指定 | `CompatibleHelper.java:63` |
| 忽略策略 | 注解 @InterceptorIgnore 或 `handle(IgnoreStrategy)` 手动设置 | `InterceptorIgnoreHelper.java:58` |

## 5. 错误与重试语义

- `CompatibleHelper.getCompatibleSet()` 无实现时 `Assert.isTrue` 抛 `MybatisPlusException`（`CompatibleHelper.java:75`）。
- `MybatisPlusException` 为 unchecked，由调用方处理；本叶子无重试。
- SqlTemplate 模板拼装纯字符串，无失败路径。

## 6. 并发细节

- `InterceptorIgnoreHelper.IGNORE_STRATEGY_CACHE`（`:40`）为 ConcurrentHashMap，线程安全。
- `IGNORE_STRATEGY_LOCAL`（`:44`）为 ThreadLocal，线程隔离；用后须 remove（`:65`），否则线程池复用会泄漏。
- `CompatibleHelper.COMPATIBLE_SET`（`:32`）静态可变，类加载后一次赋值；`setCompatibleSet` 非线程安全但属启动期调用。
- `SqlMethod` 枚举不可变。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/enums/`、`core/exceptions/`、`core/spi/`、`core/assist/`、`core/plugins/`（拦截器忽略部分，不含具体插件实现）。

**Out-of-Scope（不在本仓库源码内）**
- 具体拦截器插件实现（租户/乐观锁/分页，extension 域）。
- `SqlTemplate`（toolkit/sql 叶子）。
- ServiceLoader 加载的 CompatibleSet 具体实现（extension，不在 core）。
- `org.apache.ibatis` 第三方。

## 8. 与相邻子系统交互

- 上游：sql-injector 叶子读 `SqlMethod` 模板注入 MappedStatement；插件（extension）查 `InterceptorIgnoreHelper`。
- 本叶子 → 下游：`ISqlRunner` 委托 SqlSession；`SqlMethod.format` 委托 `SqlTemplate`。
- 数据流：**SqlMethod 枚举 → format 占位符填充 → SQL 脚本 → MappedStatement**。

## 9. 语言专项适配口径（Java/JVM）

- **枚举即元数据**：`SqlMethod` 用枚举携带 lambda 模板（`SqlTemplate.of3` 函数式），`format` 泛型重载按参数个数分发。
- **ServiceLoader**：JDK SPI 机制（`CompatibleHelper`），META-INF/services 约定。
- **ThreadLocal**：`IGNORE_STRATEGY_LOCAL` 手动清理，线程池泄漏风险已在 API 注释提示。
- **ConcurrentHashMap**：策略缓存。
- **库型无 main**。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 核心支撑架构图 | `core-support-architecture.html` | architecture | showcase |
| SQL 模板渲染数据流 | `core-support-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 dataflow 图：SqlMethod 枚举 → 占位符填充 → SQL 脚本 → MappedStatement 是典型"源→加工→目的地"数据流。
未生成 sequence/lifecycle/workflow：本叶子为支撑工具集，无多参与方时序主链、无单实体状态机、无多角色审批，按资源节省原则省略。
