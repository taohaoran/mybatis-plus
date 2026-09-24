# SQL 智能拦截器（sql-interceptors）

> 本文是 `jsqlparser` 域下的叶子子系统文档。域级总览见 `../jsqlparser.md`。
>
> 源码基准：`mybatis-plus-jsqlparser-support`（聚合模块 jsqlparser 5.2；另有 4.9/5.0 两套版本实现），分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径（聚合 5.2） |
|---|---|---|
| 多表处理基类 | `BaseMultiTableInnerInterceptor`：遍历 SelectBody/PlainSelect/Where 子查询/SelectItem/Function/FromItem，逐表拼接条件 | `plugins/inner/BaseMultiTableInnerInterceptor.java:59` 起 |
| 多租户 | `TenantLineInnerInterceptor`：对 insert/update/delete/select 自动追加租户条件；`TenantLineHandler` 回调 | `plugins/inner/TenantLineInnerInterceptor.java` |
| 数据权限 | `DataPermissionInterceptor`/`MultiDataPermissionHandler`：按权限规则过滤 SQL | `plugins/inner/DataPermissionInterceptor.java`、`plugins/handler/DataPermissionHandler.java` |
| 全表攻击阻断 | `BlockAttackInnerInterceptor`：禁止无 where 的全表 update/delete | `plugins/inner/BlockAttackInnerInterceptor.java:50` |
| 非法 SQL 拦截 | `IllegalSQLInnerInterceptor`：拦截危险 SQL | `plugins/inner/IllegalSQLInnerInterceptor.java` |
| 物理分页 | `PaginationInnerInterceptor`：count 查询 + 分页 SQL 改写（基于 jsqlparser） | `plugins/inner/PaginationInnerInterceptor.java:116` |
| 动态表名 | `DynamicTableNameJsqlParserInnerInterceptor`：AST 级动态表名（区别于 extension 正则版） | `plugins/inner/DynamicTableNameJsqlParserInnerInterceptor.java` |
| 数据变更记录 | `DataChangeRecorderInnerInterceptor`：记录 update/delete 变更 | `plugins/inner/DataChangeRecorderInnerInterceptor.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseMultiTableInnerInterceptor` | `plugins/inner/BaseMultiTableInnerInterceptor.java` | 多表 SQL 遍历骨架；`processPlainSelect/andExpression/builderExpression` |
| `TenantLineInnerInterceptor` | `plugins/inner/TenantLineInnerInterceptor.java` | 租户条件注入；`beforeQuery/beforeUpdate` 等 |
| `TenantLineHandler` | `plugins/handler/TenantLineHandler.java` | 租户列名/租户值/忽略表回调 |
| `PaginationInnerInterceptor` | `plugins/inner/PaginationInnerInterceptor.java` | 分页：`willDoQuery/beforeQuery/autoCountSql/lowLevelCountSql` |
| `BlockAttackInnerInterceptor` | `plugins/inner/BlockAttackInnerInterceptor.java:50` | 无 where 全表写阻断 |

## 3. 关键调用链

**链 1：租户条件注入（beforeQuery）**

1. `TenantLineInnerInterceptor.beforeQuery` 先 `InterceptorIgnoreHelper.willIgnoreTenantLine(msId)` 检查是否忽略；标记 `_MP_TENANT_LINE_ALREADY_PARSED=true`，`mpBs.sql(parserSingle(sql, null))`（`TenantLineInnerInterceptor.beforeQuery`）。
2. `parserSingle` → `JsqlParserSupport.processParser` 按 Statement 类型分派 → `processSelect`（继承基类遍历表）。
3. `BaseMultiTableInnerInterceptor.processPlainSelect` 对每表 `andExpression(table, where, whereSegment)` 拼接 `租户列 = 租户值`，`builderExpression` 合并到 where（`BaseMultiTableInnerInterceptor.java:99,80,385`）。

**链 2：分页拦截**

1. `willDoQuery` 判断是否需要 count/分页（`PaginationInnerInterceptor.java:116`）。
2. `beforeQuery` 处理 count 查询与分页 SQL 改写；`autoCountSql(page, sql)` 自动优化 count（`PaginationInnerInterceptor.java:259`），`lowLevelCountSql` 兜底。

**链 3：全表写阻断**

1. `BlockAttackInnerInterceptor.beforePrepare` 仅处理 UPDATE/DELETE，`parserMulti(sql, null)`（`BlockAttackInnerInterceptor.java:53-64`）。
2. `processUpdate/processDelete` 调 `checkWhere`：where 为 null 或仅匹配逻辑删除字段时 `Assert.isFalse` 抛"Prohibition of full table update/delete"（`BlockAttackInnerInterceptor.java:76-99`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 租户忽略 | `@InterceptorIgnore(tenantLine="true")` 按 msId 跳过 | InterceptorIgnoreHelper |
| 分页 count | `Page.searchCount/optimizeCountSql` 控制是否 count 与优化 | extension pagination |
| `TenantLineHandler` | 业务方实现 `getTenantId()/getTenantIdColumn()/ignoreTable(tableName)` | handler 包 |
| 重复解析标记 | `_MP_TENANT_LINE_ALREADY_PARSED`/`_MP_DYNAMIC_TABLE_NAME_ALREADY_PARSED` 防重 | 各拦截器常量 |

## 5. 错误与重试语义

- 全表写阻断直接 `Assert` 抛异常，拒绝执行。
- SQL 解析失败由 `JsqlParserSupport` 包装成 `MybatisPlusException`。
- 单拦截器失败即中断该 SQL；无重试。

## 6. 并发细节

- 拦截器实例由 MyBatis 插件单例化；共享状态仅缓存/线程池（parser-cache 叶子）。
- 每请求用 `BoundSql.additionalParameters` 传"已解析"标记，请求间隔离。
- 解析走 `JsqlParserGlobal` 线程池。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `plugins/inner/` 8 个拦截器 + `plugins/handler/` 3 个回调接口。

**Out-of-Scope（不在本仓库源码内）**

- SQL 解析/缓存基础设施——见 `../parser-cache/`。
- `MybatisPlusInterceptor` 链调度——extension `interceptor-core` 叶子。
- JSQLParser AST 库——第三方依赖。

## 8. 与相邻子系统交互

- 上游：extension `MybatisPlusInterceptor` 链调用本域 InnerInterceptor。
- 本叶子 → 下游：继承 `JsqlParserSupport`（parser-cache）解析 AST，改写后 SQL 交回 MyBatis。
- 版本差异：4.9/5.0/5.2 三套实现类同，按 JSQLParser AST API 版本分别编译；`DynamicTableNameJsqlParserInnerInterceptor` 是 extension 正则版动态表名的 AST 增强替代。

## 9. 语言专项适配口径（JVM）

- **模板方法 + 钩子回调**：`BaseMultiTableInnerInterceptor` 定遍历骨架，租户/权限等覆写具体表处理；`TenantLineHandler` 等业务回调 SPI。
- **AST 改写**：基于 JSQLParser 抽象语法树做结构化 SQL 改写，区别于 extension 正则替换。
- **多版本矩阵**：三套 jsqlparser 版本产物共用 common。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 拦截器架构图 | `sql-interceptors-architecture.html` | architecture | showcase |
| 租户 SQL 改写数据流 | `sql-interceptors-dataflow.html` | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 dataflow：原始 SQL→解析 AST→逐表注入租户条件→序列化改写 SQL→执行，是数据变换管道；lifecycle 备选但本叶子无单实体状态机，故选 dataflow。
