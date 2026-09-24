# 分页模型与方言（pagination）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
> 本文只展开分页数据模型 `Page` 与多数据库分页方言 `IDialect`/`DialectFactory`/`DialectModel`；真正改写 SQL 的 `PaginationInnerInterceptor` 位于 jsqlparser 域，见 `../../jsqlparser/sql-interceptors/`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 分页结果模型 | `Page<T>` 实现 `IPage<T>`，持有 records/total/size/current/orders 及 count 优化开关 | `plugins/pagination/Page.java:34` |
| 分页参数语义 | `searchCount/optimizeCountSql/optimizeJoinOfCountSql/maxLimit/countId` 等运行时开关 | `plugins/pagination/Page.java:66-85` |
| 方言工厂 | `DialectFactory.getDialect(DbType)` 按 DbType 枚举缓存并装配对应方言，支持 MySQL/Oracle/PostgreSQL 等同族归并 | `plugins/pagination/DialectFactory.java:35` |
| 分页方言 SPI | `IDialect.buildPaginationSql(originalSql, offset, limit)` 返回带 `?` 占位符的分页 SQL 与参数 | `plugins/pagination/dialects/IDialect.java:27` |
| 14 种方言实现 | MySql/Oracle/Oracle12c/Postgre/SQLServer/SQLServer2005/DB2/Hive2/Informix/Sybase/Trino/GBase8s/GaussDBDialect/XCloud | `plugins/pagination/dialects/` |
| 分页参数装配模型 | `DialectModel` 把 offset/limit 注入 `BoundSql.parameterMappings` 与 `additionalParameters` | `plugins/pagination/DialectModel.java:36` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Page<T>` | `plugins/pagination/Page.java:34` | 分页模型；`getPages()` 委托 `IPage.super.getPages()` 计算总页数 |
| `IPage<T>` | core `metadata/IPage.java` | 分页接口契约（core-mapping 叶子 table-metadata） |
| `IDialect` | `plugins/pagination/dialects/IDialect.java:27` | 分页 SQL 组装 SPI；`FIRST_MARK/SECOND_MARK` 为 `?` |
| `DialectFactory` | `plugins/pagination/DialectFactory.java:31` | `EnumMap<DbType,IDialect>` 缓存 + 同族归并 + 懒加载 |
| `DialectModel` | `plugins/pagination/DialectModel.java:36` | 持有 `dialectSql` 与 first/second 两个 long 参数，链式 `setConsumer` 装配参数映射 |
| `MySqlDialect` | `plugins/pagination/dialects/MySqlDialect.java:27` | `LIMIT ?,?` 方言样例 |

## 3. 关键调用链

**链 1：方言选择与 SQL 组装**

1. 调用方按 `DbType` 调 `DialectFactory.getDialect(dbType)`（`DialectFactory.java:35`）。
2. 先查 `DIALECT_ENUM_MAP` 缓存；未命中则按 `dbType.mysqlSameType()/oracleSameType()/postgresqlSameType()` 同族归并，否则逐条 `DbType` 枚举分支 new 对应方言，回填缓存（`DialectFactory.java:42-83`）。
3. `DbType.OTHER` 直接抛 `%s database not supported`（`DialectFactory.java:38-40`）。

**链 2：MySQL 分页 SQL 生成**

1. `MySqlDialect.buildPaginationSql` 在原 SQL 后拼 `LIMIT ?`；`offset!=0` 时追加 `,?`（`MySqlDialect.java:30-37`）。
2. offset 非 0 返回 `new DialectModel(sql, offset, limit).setConsumerChain()`（两个参数都注入）；offset 为 0 只 `setConsumer(true)`（单参数 limit）。

**链 3：分页参数注入 BoundSql**

1. `DialectModel.consumers(parameterMappings, configuration, additionalParameters)` 先断言三者非空（`DialectModel.java:145-149`）。
2. 依次触发 first/second 的 `Consumer<List<ParameterMapping>>` 与 `Consumer<Map<String,Object>>`，把 `mybatis_plus_first`/`mybatis_plus_second` 两个命名参数同时写入参数映射列表与附加参数 Map（`DialectModel.java:150-155`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `Page.size` | 10（每页条数） | `Page.java:50` |
| `Page.current` | 1（当前页，构造时 `current>1` 才生效） | `Page.java:55,109` |
| `Page.optimizeCountSql` | true（自动优化 count SQL） | `Page.java:66` |
| `Page.searchCount` | true（是否执行 count 查询）；`total<0` 时强制 false | `Page.java:70,289-294` |
| `Page.optimizeJoinOfCountSql` | true | `Page.java:75` |
| `Page.maxLimit` | null（单页条数上限） | `Page.java:80` |
| `Page.countId` | null（自定义 count MappedStatement id） | `Page.java:85` |

## 5. 错误与重试语义

- `DialectFactory` 不支持的 `DbType.OTHER` 直接抛 `MybatisPlusException`，不重试（`DialectFactory.java:38-40`）。
- `DialectModel.consumers` 对 `configuration/parameterMappings/additionalParameters` 做 `Assert.notNull`，任一为空立即抛断言异常（`DialectModel.java:147-149`）。
- 分页拦截器层（jsqlparser 域）负责 count 查询异常处理；本叶子纯模型/方言组装，不接触数据库，无重试退避。

## 6. 并发细节

- `Page` 是每次请求 new 的分页对象，无线程共享问题；`records/orders` 为普通 ArrayList。
- `DialectFactory.DIALECT_ENUM_MAP` 是静态 `EnumMap`，懒加载 + 多线程下可能重复 new 同一种方言（幂等，方言无状态），最终 put 覆盖结果一致，无锁也安全。
- `DialectModel` 一次性使用对象，无共享。
- 无线程池/异步。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `Page`/`PageDTO` 分页模型、`IDialect` 方言 SPI、`DialectFactory` 工厂、`DialectModel` 参数装配、14 个方言实现。

**Out-of-Scope（不在本仓库源码内）**

- `PaginationInnerInterceptor`（真正拦截 `Executor.query`、执行 count、改写分页 SQL）——在 `mybatis-plus-jsqlparser` 模块，见 `../../jsqlparser/sql-interceptors/`。
- `IPage` 接口契约本身在 `mybatis-plus-core`（table-metadata 叶子）。
- 各数据库 JDBC 驱动——不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游：业务代码 new `Page(current, size)` 传入 Mapper 方法；`PaginationInnerInterceptor`（jsqlparser 域）调用 `DialectFactory.getDialect` 与 `DialectModel` 完成 SQL 改写。
- 本叶子 → 下游：`IDialect.buildPaginationSql` 产出字符串 SQL + `DialectModel` 参数，交回 MyBatis `BoundSql`。
- 与 `interceptor-core`：本叶子的模型被 jsqlparser 分页拦截器消费，二者不直接依赖。

## 9. 语言专项适配口径（JVM）

- **库型项目**：无 main，作为 MyBatis 插件生态的模型/方言库被引用。
- **策略模式 + 工厂**：`IDialect` 策略接口 × `DialectFactory` 路由；`EnumMap` 做按枚举缓存，同族归并（mysqlSameType 等）减少方言类数量。
- **并发**：方言无状态、可共享；`EnumMap` 懒加载容忍重复构造。
- **依赖方向**：extension → core（`IPage`/`OrderItem`/`Assert`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 分页架构图 | `pagination-architecture.html` | architecture | showcase |
| 分页方言数据流 | `pagination-dataflow.html` | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 dataflow：分页是"原始 SQL → 方言组装 → 占位符参数注入 → BoundSql"的管道，符合 dataflow 语义；未选 sequence 是因为本叶子无多方消息交互，改写链已在 jsqlparser 分页拦截器中体现。
