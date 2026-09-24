# DDL 脚本执行（ddl）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| DDL 处理器 SPI | `IDdl.runScript(Consumer<DataSource>)` 业务方实现，提供 sqlFiles 列表 | `ddl/IDdl.java:30` |
| DDL 辅助执行器 | `DdlHelper.runScript(...)` 编排：建 ddl_history 表 → 按脚本查历史 → 未执行则跑脚本并记录版本 | `ddl/DdlHelper.java:107` |
| 脚本运行封装 | `DdlScript` 封装 DataSource/ScriptRunner，提供字符串/Reader/文件多种 run 重载 | `ddl/DdlScript.java:41` |
| 错误处理策略 | `DdlScriptErrorHandler` 函数式接口，默认 `PrintlnLogErrorHandler` | `ddl/DdlScriptErrorHandler.java` |
| DDL 历史生成器 SPI | `IDdlGenerator`：建表/查历史/插历史 SQL，4 种数据库实现 Mysql/Oracle/Postgre/SQLite | `ddl/history/IDdlGenerator.java:30` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `IDdl` | `ddl/IDdl.java:30` | 业务扩展点：`getSqlFiles()` 返回脚本列表，`runScript` 指定数据源 |
| `DdlHelper` | `ddl/DdlHelper.java:45` | 静态工具，核心编排逻辑 |
| `DdlScript` | `ddl/DdlScript.java:41` | 脚本执行门面，封装 `ScriptRunner` |
| `IDdlGenerator` | `ddl/history/IDdlGenerator.java:30` | 方言相关的 ddl_history 表 DML/DDL 生成 |
| `MysqlDdlGenerator` 等 4 个 | `ddl/history/` | 各数据库建历史表方言 |

## 3. 关键调用链

**链 1：带版本控制的脚本执行主流程**

1. `DdlHelper.runScript(...)` 取 `connection.getMetaData().getURL()`，构造 `SqlRunner` 与 `ScriptRunner`（`DdlHelper.java:109-111`）。
2. `ddlGenerator` 为空时按 jdbcUrl 经 `JdbcUtils.getDbType` 自动选生成器：mysqlSame→Mysql、oracleSame→Oracle、SQLite→SQLite、postgreSame→Postgre，否则抛 `Unsupported database type`（`DdlHelper.java:246-263`）。
3. `!ddlGenerator.existTable(connection)` 时先 `createDdlHistory()` 建历史表（`DdlHelper.java:118-120`）。
4. 逐个 sqlFile：`sqlRunner.selectAll(selectDdlHistory(sqlFile, SQL))` 查历史；空则未执行过——解析 `#` 分隔符后 `file.exists()` 走 `FileReader` 否则走 classpath `InputStreamReader`，`scriptRunner.runScript(reader)` 执行，成功后 `insertDdlHistory(..., getNowTime())` 记录（`DdlHelper.java:122-145`）。
5. 单个脚本异常交 `ddlScriptErrorHandler.handle(sqlFile, e)`，不中断后续脚本（`DdlHelper.java:146-150`）。

**链 2：ScriptRunner 配置**

- `getScriptRunner` 固定 `setAutoCommit(autoCommit)`、`setEscapeProcessing(false)`、`setRemoveCRs(true)`、`setStopOnError(true)`、`setFullLineDelimiter(false)`（`DdlHelper.java:236-244`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `autoCommit` | false（非自动提交） | `DdlScript.java:62` |
| 语句分隔符 | `;`；文件名可用 `path#delimiter` 指定自定义分隔符 | `DdlHelper.java:129-137` |
| `ddl_history` 表名 | 默认 `ddl_history`，生成器可覆写 `getDdlHistory()` | `IDdlGenerator.java:65-67` |
| `scriptRunnerConsumer` | 自定义 `ScriptRunner` 回调，运行前调参 | `DdlScript.java:69` |
| 错误处理器 | 默认 `PrintlnLogErrorHandler`（打印日志不抛出） | `DdlHelper.java:59` |

## 5. 错误与重试语义

- 单脚本失败被 `catch(Exception)` 捕获，交错误处理器，继续执行后续脚本（`DdlHelper.java:146-150`）。
- 旧 `DdlScript.run(List)` 与 `runScript(...,DataSource,...)` 重载吞掉所有异常仅 `LOG.error`（已 @Deprecated，3.5.11 起推荐抛出版本）。
- 不支持的数据库类型直接 `throw RuntimeException`，无重试。
- `ScriptRunner.setStopOnError(true)`：脚本内单条 SQL 出错即停该脚本。

## 6. 并发细节

- 无自建线程池；脚本在调用线程同步执行。
- ddl_history 记录依赖数据库自身事务/唯一性，多实例同时跑 DDL 需调用方自行加分布式锁（本叶子不提供）。
- `DdlHelper` 全静态方法，无共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `IDdl`/`DdlHelper`/`DdlScript`/`DdlScriptErrorHandler` + 4 个 `IDdlGenerator`。

**Out-of-Scope（不在本仓库源码内）**

- MyBatis `ScriptRunner`/`SqlRunner`（`org.apache.ibatis.jdbc.*`）——第三方依赖。
- Spring `DdlAutoConfiguration` 自动装配触发点——boot-starter 域。
- 目标数据库本身——外部系统。

## 8. 与相邻子系统交互

- 上游：Spring Boot 启动时 `DdlAutoConfiguration` 调 `IDdl.runScript`；或用户手动调 `DdlScript.run(...)`。
- 本叶子 → 下游：通过 JDBC `Connection` 执行 DDL；`JdbcUtils.getDbType` 判定方言。

## 9. 语言专项适配口径（JVM）

- **库型、启动期执行**：DDL 通常在应用启动/迁移时跑一次。
- **JDBC 直连**：非走 MyBatis 映射，直接用 `ScriptRunner`/`SqlRunner`。
- **依赖方向**：extension → core（`JdbcUtils`/`StringPool`）+ mybatis（`ScriptRunner`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| DDL 架构图 | `ddl-architecture.html` | architecture | showcase |
| DDL 脚本执行流程 | `ddl-workflow.html` | workflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 workflow：建历史表→逐脚本查记录→未执行则跑并记录，是带分支判断（已执行跳过）的确定性步骤流程。
