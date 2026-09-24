# extension-plugins 域（extension-plugins）域总览

> 本域是 mybatis-plus-extension 模块的核心扩展集，包含 10 个叶子子系统；各叶子详情见对应文档。
> 源码基准：mybatis-plus-extension 分支 3.0，commit `bf67d907478c724120bf76292da54abf9e73c2b3`。

## 1. 域职责

extension-plugins 域在 MyBatis 原生能力之上提供「拦截改写 + 便捷 CRUD」两大扩展：

- **拦截改写**：以 `MybatisPlusInterceptor` 为统一入口，把用户配置的多个 `InnerInterceptor`（乐观锁、动态表名、占位符、分页等）串成责任链，在 Executor/SqlSession 执行前后改写 SQL 或参数。
- **便捷 CRUD**：提供 ActiveRecord（实体自操作）、仓储 IRepository、静态门面 `Db`/`SimpleQuery`、链式条件 Wrapper、扩展 SQL 注入方法、序列主键生成、JSON 类型处理器、p6spy 日志、DDL 版本化执行等开箱即用能力。

域内主语言为 Java（93 文件），含 5 个 Kotlin main 文件；所有扩展都通过 MyBatis 拦截器 SPI 或 `SqlInjector` 挂载，不侵入核心。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流图 | 职责一句话 |
|------|------|--------|---------------|-----------|
| 拦截器核心 | [interceptor-core.md](interceptor-core/interceptor-core.md) | [架构图](interceptor-core/interceptor-core-architecture.html) | [时序图](interceptor-core/interceptor-core-sequence.html) | 插件链与 InnerInterceptor SPI 的调度核心 |
| 分页模型 | [pagination.md](pagination/pagination.md) | [架构图](pagination/pagination-architecture.html) | [数据流](pagination/pagination-dataflow.html) | Page 模型、方言工厂与分页 SQL 拼装 |
| 扩展注入 | [injector-ext.md](injector-ext/injector-ext.md) | [架构图](injector-ext/injector-ext-architecture.html) | [时序图](injector-ext/injector-ext-workflow.html) | 批量插入、Upsert、逻辑删除等扩展注入方法 |
| DDL 执行 | [ddl.md](ddl/ddl.md) | [架构图](ddl/ddl-architecture.html) | [时序图](ddl/ddl-workflow.html) | 版本化 DDL 脚本执行与历史表记录 |
| AR 与仓储 | [ar-repository.md](ar-repository/ar-repository.md) | [架构图](ar-repository/ar-repository-architecture.html) | [时序图](ar-repository/ar-repository-sequence.html) | ActiveRecord 实体自操作与 IRepository 仓储 |
| JSON 处理器 | [json-handlers.md](json-handlers/json-handlers.md) | [架构图](json-handlers/json-handlers-architecture.html) | —（省略，见该叶第 10 节） | 多 JSON 库（Jackson/Fastjson/Gson）类型处理器 |
| 链式条件 | [chain-conditions.md](chain-conditions/chain-conditions.md) | [架构图](chain-conditions/chain-conditions-architecture.html) | —（省略，见该叶第 10 节） | 链式 Wrapper 与 ChainWrappers 门面 |
| Db 工具 | [db-toolkit.md](db-toolkit/db-toolkit.md) | [架构图](db-toolkit/db-toolkit-architecture.html) | [时序图](db-toolkit/db-toolkit-sequence.html) | Db/SimpleQuery/SqlHelper 静态门面 |
| 键生成器 | [key-generators.md](key-generators/key-generators.md) | [架构图](key-generators/key-generators-architecture.html) | —（省略，见该叶第 10 节） | 9 种数据库序列主键生成器 |
| p6spy 脚本 | [scripting-p6spy.md](scripting-p6spy/scripting-p6spy.md) | [架构图](scripting-p6spy/scripting-p6spy-architecture.html) | [时序图](scripting-p6spy/scripting-p6spy-sequence.html) | 多模板语言驱动与 p6spy SQL 日志 |

## 3. 域级机制细节

- **统一拦截链**：所有 SQL 改写能力都实现 `InnerInterceptor`（六个钩子：`willDoQuery`/`beforeQuery`/`beforeUpdate` 等），由 `MybatisPlusInterceptor` 按注册顺序调度；插件链只对 Executor 的 `query/update` 做动态代理，重建 CacheKey 以匹配改写后的 SQL。
- **方言抽象**：分页等数据库差异通过 `DialectFactory` 按 `DbType` 路由到 14 种 `IDialect`，产出 `DialectModel`（SQL 片段 + 参数列表）。
- **SqlHelper 会话边界**：AR、仓储、Db 门面都通过 `SqlHelper.execute(cls, fn)` 统一「开 Session → 取 Mapper → 执行 → 关 Session」，避免连接泄漏。
- **扩展注入与核心解耦**：扩展方法（批量插入等）在启动期经 `SqlInjector.inspectInject` 拼好动态 SQL，注册进 MyBatis `Configuration` 的 `MappedStatement`，运行期走原生 Mapper 调用路径。

## 4. 域级图

![extension-plugins 域架构](extension-plugins-architecture.html)
![插件域请求时序](extension-plugins-sequence.html)
![插件域 SQL 改写数据流](extension-plugins-dataflow.html)
