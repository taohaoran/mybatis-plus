# 生成器配置体系（generator-config）

> 本文是 `generator` 域下的叶子子系统文档。域级总览见 `../generator.md`。
> 本文只展开生成器"配置如何被定义、汇总并解析为内部结构"的职责边界；数据库元数据采集见 `../generator-query/generator-query.md`，模板渲染见 `../generator-engine/generator-engine.md`，生成主流程编排见 `../generator-core/generator-core.md`。
>
> 源码基准：`mybatis-plus-generator` 分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据源配置 | 保存 JDBC 连接信息、schema、连接属性、查询类与类型转换器选择；懒建 Connection | `config/DataSourceConfig.java`（`getConn()` 行 235） |
| 全局配置 | 输出目录、作者、是否 Kotlin/Swagger/Open、是否打开目录、注释日期 | `config/GlobalConfig.java` |
| 策略配置 | 表包含/排除、正则匹配、schema 开关、entity/mapper/service/controller 子策略访问入口 | `config/StrategyConfig.java`（`entity()` 行 160、`matchIncludeTable()` 行 284） |
| 包配置 | 父包及各模块（entity/mapper/service/controller/xml）包名与路径 | `config/PackageConfig.java` |
| 注入配置 | 自定义输出文件 `CustomFile`、`beforeOutputFile` 钩子、自定义属性 Map | `config/InjectionConfig.java` |
| 模板配置（已废弃） | 各模板路径覆盖，3.5.6 起迁入 StrategyConfig 子策略 | `config/TemplateConfig.java`（`@Deprecated`） |
| 配置汇总器 | 聚合六类配置，补默认值，计算输出路径，反射实例化 `IDatabaseQuery` | `config/builder/ConfigBuilder.java`（构造器行 109） |
| 子策略建造器 | Entity/Mapper/Service/Controller/CustomFile 各模块的生成开关、模板路径、命名策略、渲染数据 | `config/builder/Entity.java`、`Mapper.java`、`Service.java`、`Controller.java`、`CustomFile.java` |
| 默认值供给 | 当用户未传某类配置时提供空默认（GeneratorBuilder） | `config/builder/GeneratorBuilder.java` |
| 输出路径计算 | 按包配置 + 注入配置为每个 `OutputFile` 枚举算出绝对输出目录 | `config/builder/PathInfoHandler.java`（构造器行 59） |
| 类型转换 | 各数据库 JDBC 类型 → Java 类型的转换实现（MySQL/Oracle/PG/SQLServer 等 14 套） | `config/converts/*TypeConvert.java`、`TypeConverts.java` |
| 方言元数据查询 | 各库表/列元数据 SQL 查询方言（23 个 DbQuery 实现） | `config/querys/*Query.java`、`DbQueryRegistry.java` |
| 命名与列类型枚举 | 命名策略、日期类型、列类型枚举 `IColumnType` | `config/rules/NamingStrategy.java`、`DbColumnType.java`、`DateType.java` |
| 表/列领域对象 | 生成中间产物 `TableInfo`/`TableField`/`LikeTable` | `config/po/TableInfo.java`、`TableField.java` |
| 输出文件枚举 | 标记 entity/mapper/xml/service/serviceImpl/controller/parent 七类产物槽位 | `config/OutputFile.java`、`TemplateType.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ConfigBuilder` | `config/builder/ConfigBuilder.java:36` | 配置汇总根；持有 dataSource/strategy/global/package/injection/pathInfo/databaseQuery，是引擎与查询的共享上下文 |
| `DataSourceConfig` | `config/DataSourceConfig.java` | 唯一必传配置；`getConn()`（行 235）懒建连接，`databaseQueryClass`（行 122，默认 `DefaultQuery.class`）决定用哪种元数据查询 |
| `StrategyConfig` | `config/StrategyConfig.java:37` | 表过滤与子策略容器；`entity()/mapper()/service()/controller()` 返回子策略建造器 |
| `PathInfoHandler` | `config/builder/PathInfoHandler.java` | 构造期一次性算出 `Map<OutputFile,String>` 输出路径 |
| `IConfigBuilder<T>` | `config/IConfigBuilder.java` | 所有 `Builder` 接口，统一 `T build()` |
| `INameConvert` | `config/INameConvert.java` | 表/列名 → 类/属性名转换策略点 |
| `ITypeConvert` / `TypeConverts` | `config/ITypeConvert.java`、`converts/TypeConverts.java` | JDBC 类型转 Java 类型 SPI；按 `DbType` 路由到 14 个实现 |
| `IDbQuery` / `DbQueryRegistry` | `config/IDbQuery.java`、`querys/DbQueryRegistry.java:30` | 方言元数据查询接口与注册表（`EnumMap<DbType,IDbQuery>`） |
| `OutputFile` 枚举 | `config/OutputFile.java` | 七类产物槽位，驱动路径表与输出分派 |
| `TableInfo` / `TableField` | `config/po/TableInfo.java:43` | 单表生成中间模型；`processTable()`（行 325）收尾处理 |

## 3. 关键调用链

1. **配置汇总构造链**：用户经 `FastAutoGenerator`/`AutoGenerator` 传入六类配置 → `ConfigBuilder` 构造器（`ConfigBuilder.java:109`）：对每个可空配置调用 `GeneratorBuilder::xxxConfig` 补默认（行 113–117）→ `new PathInfoHandler(...).getPathInfo()` 算出输出路径并放入 `pathInfo`（行 118）→ 从 `dataSourceConfig.getDatabaseQueryClass()`（行 119）反射拿到带 `ConfigBuilder` 入参的构造器并 `newInstance(this)` 创建元数据查询实例（行 121–122）。
2. **表信息懒加载链**：引擎 `batchOutput()` 需要表列表时调 `config.getTableInfoList()`（`ConfigBuilder.java:170`）——若 `tableInfoList` 为空则调 `databaseQuery.queryTables()`（行 172）回填，否则直接返回缓存。
3. **输出路径解析链**：引擎各 `outputXxx()` 方法按 `OutputFile.entity/mapper/xml/service/...` 调 `getPathInfo(...)`（`AbstractTemplateEngine`，见 engine 叶子）取回 `PathInfoHandler` 预计算的绝对目录，再拼类名落盘。
4. **子策略渲染数据链**：`getObjectMap()` 时依次调 `strategyConfig.controller().renderData(tableInfo)`、`mapper().renderData()`、`service().renderData()`、`entity().renderData()`（`Entity.java:425` 等）把各模块开关/模板名/类名变量合并进模板上下文。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `DataSourceConfig.databaseQueryClass` | `DefaultQuery.class`（3.5.3 起默认元数据查询方式） | `DataSourceConfig.java:122` |
| `DataSourceConfig.schemaName` | 空；`getConn()` 建连后若为空则取 `connection.getSchema()`（行 251–253） | `DataSourceConfig.java:251` |
| `DataSourceConfig.connectionProperties` | 空 Map；MySQL 读表注释需 `remarks=true&useInformationSchema=true` | `DataSourceConfig.java:113` |
| `StrategyConfig.enableSchema` | false；开启后拼接 `schema.table` 并设 `tableInfo.setConvert(true)` | `StrategyConfig.java:105` |
| `StrategyConfig.include/exclude` | 空集合；非空时 `filter()` 反向生成或排除表 | `StrategyConfig.java:347/352` |
| `GlobalConfig.outputDir/open/kotlin/swagger` | 输出目录、是否生成后打开目录、是否 Kotlin 输出、是否 swagger 注解 | `GlobalConfig.java` |
| `TemplateLoadWay` | `FILE`（文件模板）；可切 `TEXT`（字符串模板） | `ConfigBuilder.java:98` |
| 各子策略 `fileOverride` | 默认 false；`isCreate()` 命中已存在文件仅告警不覆盖 | `AbstractTemplateEngine.java:362` |

## 5. 错误与重试语义

- `ConfigBuilder` 反射实例化 `IDatabaseQuery` 失败时抛 `RuntimeException(exception)`（`ConfigBuilder.java:123–124`），无重试——配置期即快速失败。
- 用户配置的 include/exclude 表在库中不存在时，`AbstractDatabaseQuery.filter()` 仅 `LOGGER.warn("Table [...] does not exist in the database!")`（`query/AbstractDatabaseQuery.java:86`），不中断生成。
- 主键为自增却又配置了全局 idType 时，`DefaultQuery.convertTableFields()` 告警自增将覆盖全局策略（`DefaultQuery.java:120–122`）。
- `notLikeTable` 配置当前不被支持，`DefaultQuery.getTables()` 仅 warn（`DefaultQuery.java:102–103`）。
- 无重试/退避机制：本叶子是一次性本地代码生成，失败即抛异常终止。

## 6. 并发细节

- 本叶子是**单线程本地构建库**，不创建任何线程池、锁或异步任务；`ConfigBuilder` 在构造期一次性完成路径计算与查询实例创建，之后只读。
- `tableInfoList`（`ConfigBuilder.java:49`）、`pathInfo`（行 54）为普通 `ArrayList`/`HashMap`，靠"构造完成后只读"保证可见性，无并发原语。
- `getTableInfoList()` 的懒加载非线程安全，但生成流程在单线程 `execute()` 内顺序调用，不存在并发调用者。
- JDBC `Connection` 由 `DataSourceConfig.getConn()` 懒建并在 `DefaultQuery.queryTables()` 的 `finally` 中 `closeConnection()` 释放（`DefaultQuery.java:88–91`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `config/` 包全部配置模型、建造器、路径计算、类型转换、方言 DbQuery、命名/列类型枚举与表/列领域对象。

**Out-of-Scope（不在本仓库源码内）**
- 实际的 JDBC 元数据读取（`java.sql.DatabaseMetaData`）经由 `jdbc/DatabaseMetaDataWrapper` 封装，落在 generator-query 叶子；JDBC 驱动本身（MySQL/Oracle 等驱动）为第三方依赖，不在本仓库源码内。
- 模板渲染与落盘（Velocity/FreeMarker 等）见 generator-engine；生成主流程编排见 generator-core。
- 第三方模板引擎 Velocity / FreeMarker / Beetl / Enjoy 均为外部依赖，不在本仓库源码内。

## 8. 与相邻子系统交互

- **上游**：generator-core 的 `AutoGenerator`/`FastAutoGenerator` 持有本叶子的六类配置并在 `execute()` 时组装出 `ConfigBuilder`。
- **本叶子 → generator-query**：`ConfigBuilder` 反射创建 `IDatabaseQuery` 实现，并在 `getTableInfoList()` 时调用其 `queryTables()`。
- **本叶子 → generator-engine**：`ConfigBuilder` 作为共享上下文传给 `AbstractTemplateEngine.setConfigBuilder()`；引擎反向读取 `pathInfo`、`strategyConfig`、`globalConfig`。
- **下游依赖**：`config/po/TableInfo` 被 query 写入、被 engine 读取渲染，是两叶子之间的数据契约。

## 9. 语言专项适配口径（JVM）

- **库型项目无 main**：本叶子是被用户 `main` 方法调用的配置库，不启动进程；入口是各 `Builder` 与 `ConfigBuilder` 构造器。
- **依赖方向**：generator 模块 implementation 依赖 mybatis-plus-spring（facts.md §3），本叶子只用 core 的 `StringUtils/StringPool`，不反向依赖。
- **建造器范式**：所有配置类采用"不可变配置 + 内部 `Builder implements IConfigBuilder`"，`FastAutoGenerator` 用 `Consumer<Builder>` 流式填充，是 Java/Spring Boot 风格的 fluent builder。
- **反射装配**：`IDatabaseQuery` 由 `databaseQueryClass` 反射实例化（`ConfigBuilder.java:121`），属插件式扩展点，新增方言查询类只需改数据源配置。
- **构建**：Gradle 多模块，Lombok `@Getter/@Setter` 省略样板；`options.release = 8`，无字节码增强运行时副作用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 配置体系架构图 | `generator-config-architecture.html` | architecture | showcase |
| 配置汇总构建数据流 | `generator-config-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未生成 workflow/sequence/lifecycle 图：配置体系是静态拓扑与数据装配结构，无多角色分步审批、多方消息时序或单一实体状态机语义；第二图选用 dataflow 表达"六类配置 → ConfigBuilder 汇总 → 路径表/查询实例"的数据加工管道，与架构图信息互补。
