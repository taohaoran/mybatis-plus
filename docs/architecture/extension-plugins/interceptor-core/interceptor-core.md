# 拦截器核心（interceptor-core）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
> 本文只展开 MyBatis 插件链调度与三个内置内部拦截器（乐观锁、动态表名、占位符替换），分页拦截器与 SQL 智能拦截器分别见 `../pagination/` 与 `../../jsqlparser/sql-interceptors/`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 统一插件入口 | `MybatisPlusInterceptor` 实现 MyBatis `Interceptor`，在 `Executor.update/query` 与 `StatementHandler.prepare/getBoundSql` 五个签名上织入，按顺序调度所有 `InnerInterceptor` | `plugins/MybatisPlusInterceptor.java:50` |
| 内部拦截器 SPI | `InnerInterceptor` 接口定义 6 个默认钩子：`willDoQuery/beforeQuery/willDoUpdate/beforeUpdate/beforePrepare/beforeGetBoundSql`，全部默认空实现，按需覆写 | `plugins/inner/InnerInterceptor.java:38` |
| 编程式注册 | `addInnerInterceptor(InnerInterceptor)` 手动追加到链尾；`getInterceptors()` 返回不可变视图 | `plugins/MybatisPlusInterceptor.java:117` |
| Properties 规则装配 | `setProperties` 按 `@别名=全类名` + `别名:属性=值` 的内部规则反射实例化并注入属性 | `plugins/MybatisPlusInterceptor.java:138` |
| 乐观锁 | `OptimisticLockerInnerInterceptor` 在 update 前把 `@Version` 字段旧值写入 where、新值 set 进实体，支持实体式与 Wrapper 式两种更新 | `plugins/inner/OptimisticLockerInnerInterceptor.java:73` |
| 动态表名 | `DynamicTableNameInnerInterceptor` 用 `TableNameParser` 切出 SQL 中所有表名 token，回调 `TableNameHandler` 逐个替换 | `plugins/inner/DynamicTableNameInnerInterceptor.java:45` |
| 表名替换回调 | `TableNameHandler` 函数式接口，`dynamicTableName(sql, tableName)` 返回新表名 | `plugins/handler/TableNameHandler.java:24` |
| 占位符运行时替换 | `ReplacePlaceholderInnerInterceptor` 在查询前把 `${...}` 占位符按转义符规则替换为字面值 | `plugins/inner/ReplacePlaceholderInnerInterceptor.java:40` |
| 属性映射工具 | `PropertyMapper` 流式 `whenNotBlank` 取值与 `@` 分组，供拦截器与 Boot 自动装配复用 | `toolkit/PropertyMapper.java:37` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MybatisPlusInterceptor` | `plugins/MybatisPlusInterceptor.java:50` | 唯一对外的 MyBatis `@Intercepts` 插件；持有 `List<InnerInterceptor>`，`intercept()` 按 target 类型分发 |
| `InnerInterceptor` | `plugins/inner/InnerInterceptor.java:38` | 内部拦截器扩展点；6 个 default 钩子，子拦截器只覆写关心的时机 |
| `OptimisticLockerInnerInterceptor` | `plugins/inner/OptimisticLockerInnerInterceptor.java:73` | 乐观锁；`beforeUpdate` 改写参数 Map；`FieldEqFinder` 状态机在 Wrapper 中定位 version 等值条件 |
| `DynamicTableNameInnerInterceptor` | `plugins/inner/DynamicTableNameInnerInterceptor.java:45` | 动态表名；`beforeQuery` 与 `beforePrepare` 双入口，用附加参数标记防重复解析 |
| `TableNameHandler` | `plugins/handler/TableNameHandler.java:24` | 表名替换回调（函数式接口） |
| `ReplacePlaceholderInnerInterceptor` | `plugins/inner/ReplacePlaceholderInnerInterceptor.java:40` | 查询前占位符替换 |
| `PropertyMapper` | `toolkit/PropertyMapper.java:37` | 链式属性读取与 `@` 分组反射装配辅助 |

## 3. 关键调用链

**链 1：一次 SELECT 经过拦截器链的完整时序**

1. MyBatis `Executor.query` 被代理，进入 `MybatisPlusInterceptor.intercept`（`MybatisPlusInterceptor.java:56`）。
2. 判断 `target instanceof Executor` 且 `SqlCommandType.SELECT`，遍历 `interceptors`：先调 `willDoQuery`，任一返回 `false` 立即 `return Collections.emptyList()` 短路（`MybatisPlusInterceptor.java:74-77`）；再调 `beforeQuery` 做 SQL 改写。
3. 全部放行后 `executor.createCacheKey(...)` 重建缓存 key，再 `executor.query(...)` 走到真实数据库（`MybatisPlusInterceptor.java:80-81`）。
4. 若 `target` 是 `StatementHandler`：`args==null` 走 `beforeGetBoundSql`（仅 Batch/ReuseExecutor），否则走 `beforePrepare(sh, connection, transactionTimeout)`（`MybatisPlusInterceptor.java:94-104`）。

**链 2：乐观锁 updateById 改写**

1. `Executor.update` 被拦截，遍历拦截器调 `beforeUpdate`（`MybatisPlusInterceptor.java:83-87`）。
2. `OptimisticLockerInnerInterceptor.beforeUpdate` 仅处理 `UPDATE` 命令且参数为 Map（`OptimisticLockerInnerInterceptor.java:110-118`）。
3. `doOptimisticLocker` 从 Map 取 `Constants.ENTITY`，经 `getVersionFieldInfo` 查 `@Version` 字段；反射读旧值、`VersionFactory` 算新值（数值 +1 / 时间戳取当前时间），把旧值塞进 where、新值 `versionField.set(et, newVal)`（`OptimisticLockerInnerInterceptor.java:120-163`）。

**链 3：动态表名双入口防重**

1. `beforeQuery` 先标记附加参数 `_MP_DYNAMIC_TABLE_NAME_ALREADY_PARSED=true`，再 `changeTable`（`DynamicTableNameInnerInterceptor.java:72-79`）。
2. 若某条 SQL 只走到 `StatementHandler.prepare` 而未走 `Executor` 阶段（如存储过程/特殊执行器），`beforePrepare` 检查该附加参数不存在才补做一次替换（`DynamicTableNameInnerInterceptor.java:82-93`），避免重复改写。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `interceptors` 列表 | 空 `ArrayList`，由 `addInnerInterceptor` 或 `setProperties` 填充 | `MybatisPlusInterceptor.java:53` |
| `@page=全类名` 规则 | `@` 开头定义一个 InnerInterceptor 别名，value 为实现类全限定名 | `MybatisPlusInterceptor.java:138-145` |
| `别名:属性=值` | 反射设置该拦截器的同名 setter 属性 | `PropertyMapper.group` |
| `OptimisticLockerInnerInterceptor.wrapperMode` | 默认 `false`；`true` 时额外支持 `update(wrapper)` 形式的乐观锁 | `OptimisticLockerInnerInterceptor.java:99-107` |
| `OptimisticLockerInnerInterceptor.exception` | 自定义运行期异常，version 为 null 时抛出 | `OptimisticLockerInnerInterceptor.java:78-79` |
| `DynamicTableNameInnerInterceptor.tableNameHandler` | 默认 `(sql, t) -> sql`（不替换）；必须显式注入业务回调 | `DynamicTableNameInnerInterceptor.java:55` |
| `ReplacePlaceholderInnerInterceptor.escapeSymbol` | 默认 null，由 `SqlUtils.replaceSqlPlaceholder` 决定转义符 | `ReplacePlaceholderInnerInterceptor.java:44` |
| `InterceptorIgnoreHelper` 忽略条件 | 通过 `@InterceptorIgnore` 注解按 msId 跳过某拦截器 | 被各拦截器 `willIgnoreXxx` 调用 |

## 5. 错误与重试语义

- 拦截器链本身不做重试；SQL 改写失败（JSQL 解析、反射读写）直接抛 `ExceptionUtils.mpe`，由上层事务回滚。
- 乐观锁 version 为 null：若配置了自定义 `exception` 则抛出，否则静默 return（不附加 version 条件）——见 `OptimisticLockerInnerInterceptor.java:135-140`。
- Wrapper 模式下 `FieldEqFinder` 找不到 `version = ?` 等值条件时静默 return，不注入新值（`OptimisticLockerInnerInterceptor.java:192-194`）。
- `willDoQuery`/`willDoUpdate` 返回 false 时分别返回空列表 / 影响行数 -1，短路掉后续执行，不抛错。
- 动态表名 `changeTable` 的 `finally` 块无条件执行 `hook.run()`，即使 `processTableName` 抛异常也触发回调（`DynamicTableNameInnerInterceptor.java:95-103`）。

## 6. 并发细节

- `interceptors` 是普通 `ArrayList`，生命周期在 MyBatis 插件初始化阶段装配完成，运行期只读；`getInterceptors()` 返回 `Collections.unmodifiableList` 视图（`MybatisPlusInterceptor.java:121-123`）。装配期若并发 `addInnerInterceptor` 需调用方自行保证 happens-before。
- `OptimisticLockerInnerInterceptor.ENTITY_CLASS_CACHE` 是 `static final ConcurrentHashMap`，缓存 `msId -> entityClass`，线程安全（`OptimisticLockerInnerInterceptor.java:84`）。
- `VersionFactory.VERSION_FUNCTION_MAP` 是静态 `HashMap`，类加载后只读，无并发写。
- 无独立线程池/异步任务；所有改写都在调用线程（MyBatis Executor 线程）同步完成。
- `DynamicTableNameInnerInterceptor` 用 `BoundSql.additionalParameters` 传递"已解析"标记，是每请求隔离的，无线程共享状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `MybatisPlusInterceptor` 插件链调度、`InnerInterceptor` SPI。
- 三个内置内部拦截器：乐观锁、动态表名、占位符替换；`TableNameHandler` 回调；`PropertyMapper` 装配工具。

**Out-of-Scope（不在本仓库源码内）**

- 物理分页拦截器 `PaginationInnerInterceptor`、多租户/数据权限/全表阻断等 SQL 改写拦截器——位于 `mybatis-plus-jsqlparser-support` 模块，见 `../../jsqlparser/sql-interceptors/`。
- MyBatis 原生 `Plugin`/`Interceptor` 代理机制（`org.apache.ibatis.plugin.*`）——第三方依赖，不在本仓库源码内。
- `@InterceptorIgnore` 注解与 `InterceptorIgnoreHelper` 位于 `mybatis-plus-core`（core-support 叶子）。

## 8. 与相邻子系统交互

- 上游：MyBatis `Executor` / `StatementHandler` 代理对象调用 `intercept`；Boot 自动装配（`boot-autoconfigure` 叶子）把 `MybatisPlusInterceptor` 注册进 `Configuration`。
- 本叶子 → 下游：改写后的 `BoundSql.sql` 交回 MyBatis 执行；乐观锁读写实体字段经 `TableInfoHelper`/`TableFieldInfo`（`table-metadata` 叶子）；动态表名用 core 的 `TableNameParser`/`PluginUtils`。
- 与 `jsqlparser` 域的关系：本叶子的内置拦截器只用正则/字符串替换，不依赖 JSQLParser；需要 AST 级 SQL 改写的拦截器全部在 jsqlparser 域实现，二者都挂载在同一条 `MybatisPlusInterceptor` 链上。

## 9. 语言专项适配口径（JVM）

- **库型项目无 main 入口**：本叶子是被 MyBatis/Spring Boot 加载的插件库，无独立进程；生命周期由 `SqlSessionFactory` 构建时一次性装配。
- **代理模式**：复用 MyBatis `@Intercepts` + `Plugin.wrap`（JDK 动态代理），5 个 `@Signature` 锚点（`MybatisPlusInterceptor.java:41-48`）。
- **并发**：无自建线程池；可变共享状态仅限装配期的 `interceptors` 列表与两个静态缓存（ConcurrentHashMap）。
- **配置面**：`setProperties` 内部规则 + Spring `@Bean` 编程式注册两条路径；`InterceptorIgnoreHelper` 提供按 msId 的运行时跳过。
- **依赖方向**：extension → core（`TableInfoHelper`/`PluginUtils`/`InterceptorIgnoreHelper`）→ mybatis（`Interceptor`/`Executor`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 拦截器核心架构图 | `interceptor-core-architecture.html` | architecture | showcase |
| 拦截器链调用时序图 | `interceptor-core-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 lifecycle/dataflow/workflow：本叶子是同步插件链调度，无单实体状态机、无数据管道、无审批泳道，第二图选用最贴合的 sequence（调用时序）。
