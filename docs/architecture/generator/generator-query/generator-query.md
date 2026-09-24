# 数据库元数据查询（generator-query）

> 本文是 `generator` 域下的叶子子系统文档。域级总览见 `../generator.md`。
> 本文只展开"如何从 JDBC 读取表/列/索引元数据并转换为 `TableInfo`"的职责；配置模型见 `../generator-config/generator-config.md`，模板渲染见 `../generator-engine/generator-engine.md`。
>
> 源码基准：`mybatis-plus-generator` 分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 元数据查询 SPI | 唯一方法 `List<TableInfo> queryTables()` | `query/IDatabaseQuery.java:27` |
| 查询抽象基类 | 持有 ConfigBuilder/三类配置，实现 include/exclude 过滤与不存在表告警 | `query/AbstractDatabaseQuery.java:39`（`filter()` 行 69） |
| 默认 JDBC 元数据查询 | 基于 `DatabaseMetaDataWrapper` 读表/列/索引，做主键识别与列类型转换 | `query/DefaultQuery.java:49`（`queryTables()` 行 61） |
| 自定义 SQL 查询 | 允许用户提供 SQL 自行映射表/列 | `query/SQLQuery.java` |
| JDBC 元数据封装 | 封装 `java.sql.DatabaseMetaData` 的 getTables/getColumns/getIndexInfo，区分主键/自增 | `jdbc/DatabaseMetaDataWrapper.java` |
| 关键字处理 | 表/列名命中 SQL 关键字时加转义引号（MySQL/PG/H2 三套） | `keywords/BaseKeyWordsHandler.java`、`MySqlKeyWordsHandler.java` 等 |
| 类型注册表 | 按 GlobalConfig 初始化，JDBC 类型 → `IColumnType` 路由；支持用户 `ITypeConvertHandler` 覆盖 | `type/TypeRegistry.java:33`（`getColumnType()` 行 102） |
| 类型转换钩子 | 用户自定义列类型转换 SPI | `type/ITypeConvertHandler.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `IDatabaseQuery` | `query/IDatabaseQuery.java:27` | 元数据查询扩展点；`ConfigBuilder` 反射实例化的目标接口 |
| `AbstractDatabaseQuery` | `query/AbstractDatabaseQuery.java:39` | 模板基类；`filter()` 实现 include/exclude 集合差集与大小写不敏感处理 |
| `DefaultQuery` | `query/DefaultQuery.java:49` | 默认实现；编排 getTables → 过滤 → 逐表 `convertTableFields` → 关连接 |
| `DatabaseMetaDataWrapper` | `jdbc/DatabaseMetaDataWrapper.java` | JDBC `DatabaseMetaData` 薄封装；内部类 `Table`/`Column`/`Index` |
| `TypeRegistry` | `type/TypeRegistry.java:33` | 列类型注册与解析；`getColumnType(MetaInfo)`（行 102）默认 OBJECT 兜底 |
| `ITypeConvertHandler` | `type/ITypeConvertHandler.java` | 用户覆盖列类型转换的 SPI（`DataSourceConfig.typeConvertHandler`） |
| `BaseKeyWordsHandler` | `keywords/BaseKeyWordsHandler.java` | 关键字转义基类；`MySql/PostgreSql/H2KeyWordsHandler` 为方言实现 |
| `TableField.MetaInfo` | `config/po/TableField.java`（被 `DefaultQuery.java:113` 使用） | 携带原始 `ColumnInfo` 供类型转换与自定义器消费 |

## 3. 关键调用链

1. **表信息采集主链**（`DefaultQuery.queryTables()`，行 61–92）：先判 include/exclude 是否生效（行 63–64）→ `getTables()`（行 67，按 `likeTable` 拼表名模式、按 skipView 决定是否含 VIEW，行 94–106）→ 逐表构造 `TableInfo` 并设注释，按 `matchIncludeTable/matchExcludeTable`（行 76–80）归入 include/exclude 列表 → `AbstractDatabaseQuery.filter()`（行 84，行 69–96）做集合差集，剔除配置中库不存在的表并告警 → 仅对最终保留表 `forEach(this::convertTableFields)`（行 86，性能优化：只处理需生成的表）→ `finally` 中 `databaseMetaDataWrapper.closeConnection()`（行 90）释放连接。
2. **列信息转换链**（`convertTableFields()`，行 108–139）：`getColumnsInfo(tableName)`（行 110，底层 `getColumnsInfo(tableName, true)` 行 142）拿到 `Map<列名,Column>` → 逐列构造 `TableField.MetaInfo`（行 113）→ 若 `columnInfo.isPrimaryKey()` 则 `field.primaryKey(isAutoIncrement)` 并置 `havePrimaryKey`（行 117–119）→ 列类型转换：若 `dataSourceConfig.getTypeConvertHandler()!=null` 走用户 handler（行 125–127），否则 `typeRegistry.getColumnType(metaInfo)`（行 129）→ 组装 `TableField`（列名/类型/注释/metaInfo，行 131）→ `entity.handleTableFieldMetaInfo(...)` 交给策略后处理（行 132）→ `entity.getNameConvert().propertyNameConvert(field)` 算属性名（行 133）→ `addField` → 收尾 `setIndexList(getIndex(...))`（行 137）与 `tableInfo.processTable()`（行 138）。
3. **连接获取链**：`DefaultQuery` 构造器（行 54–58）用 `dataSourceConfig.getConn()` 建连接并传入 `DatabaseMetaDataWrapper`，schema 取自 `dataSourceConfig.getSchemaName()`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `DataSourceConfig.databaseQueryClass` | `DefaultQuery.class`；可切自定义 `AbstractDatabaseQuery` 子类 | `DataSourceConfig.java:122` |
| `DataSourceConfig.typeConvertHandler` | null；非空时列类型转换改走用户 SPI | `DataSourceConfig.java:125`（消费处） |
| `StrategyConfig.skipView` | 默认读 TABLE+VIEW；skipView=true 只读 TABLE | `DefaultQuery.java:96/105` |
| `StrategyConfig.likeTable` | null；非空时作为表名模式下推到 JDBC getTables | `DefaultQuery.java:99–100` |
| `StrategyConfig.notLikeTable` | 当前不支持，仅 warn | `DefaultQuery.java:102–103` |
| MySQL 注释读取 | 需连接属性 `remarks=true&useInformationSchema=true`（FAQ 注释） | `DefaultQuery.java:44` |

## 5. 错误与重试语义

- 连接在 `queryTables()` 的 `finally` 中**必然关闭**（`DefaultQuery.java:88–91`），即使采集抛异常也释放，杜绝连接泄漏。
- 配置 include/exclude 中库不存在的表不抛错，仅 `LOGGER.warn`（`AbstractDatabaseQuery.java:86`）。
- JDBC 元数据异常向上抛为运行时异常，由引擎 `batchOutput()` 包成 `RuntimeException("An exception occurred in the output file: ", e)`（`AbstractTemplateEngine.java:252–253`）终止生成；无重试。
- 列类型未匹配时 `TypeRegistry.getColumnType(metaInfo)` 以 `DbColumnType.OBJECT` 兜底（`TypeRegistry.java:102–103`）。

## 6. 并发细节

- 单线程本地采集，无线程池/锁；`DefaultQuery` 实例由 `ConfigBuilder` 反射创建一次即用完。
- `DatabaseMetaDataWrapper` 持有单个 `Connection`，非线程安全，但仅在单线程 `queryTables()` 内使用。
- 无共享可变状态跨调用；`tableInfoList` 由 `ConfigBuilder` 缓存一次。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `query/`、`jdbc/`、`keywords/`、`type/` 四个包：元数据读取、过滤、列类型转换与关键字转义。

**Out-of-Scope（不在本仓库源码内）**
- `java.sql.DatabaseMetaData` / JDBC 驱动本身为 JDK 与第三方数据库驱动，不在本仓库源码内。
- 表/列如何渲染成代码文件见 generator-engine；如何被主流程编排见 generator-core。
- `config/querys/*Query.java`（23 个方言 `IDbQuery`）属旧版方言查询体系，本叶子聚焦 3.5.3 新增的 `query/IDatabaseQuery` 体系；二者在 facts.md §4 并列。

## 8. 与相邻子系统交互

- **上游**：generator-config 的 `ConfigBuilder` 反射创建本叶子查询实例，并在 `getTableInfoList()` 调用 `queryTables()`。
- **本叶子 → generator-config**：读取 `dataSourceConfig/strategyConfig/globalConfig`，并把采集结果写入 `config/po/TableInfo`、`TableField`（配置叶子定义的领域对象）。
- **本叶子 → generator-engine**：产出的 `List<TableInfo>` 是引擎 `batchOutput()` 逐表渲染的输入。

## 9. 语言专项适配口径（JVM）

- **JDBC 资源管理**：严格 try/finally 关闭 `Connection`，是典型 JDBC 本地工具写法；无连接池（生成器是一次性 CLI/库）。
- **SPI + 模板方法**：`IDatabaseQuery`（接口）→ `AbstractDatabaseQuery`（模板基类，固化 filter）→ `DefaultQuery`（默认实现），扩展自定义查询时继承基类即可。
- **反射装配**：实例化经 `ConfigBuilder` 反射带参构造，要求自定义查询类拥有 `(ConfigBuilder)` 构造器。
- **方言可插拔**：关键字转义（`BaseKeyWordsHandler`）与列类型转换（`TypeRegistry`/`ITypeConvertHandler`）均为按 `DbType`/用户配置路由的策略点。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 元数据查询架构图 | `generator-query-architecture.html` | architecture | showcase |
| 元数据采集数据流 | `generator-query-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未生成 sequence/workflow/lifecycle 图：核心是"JDBC 数据源 → 表/列读取 → 类型转换 → TableInfo"的数据加工管道，dataflow 最贴切；无多角色泳道流程、无单一实体状态机。
