# 条件构造器（conditions-wrapper）

> 本文是 `core-mapping` 域下的叶子子系统文档。域级总览见 `../core-mapping.md`。
> 本文展开 `core/conditions/` + `toolkit/Wrappers`：链式 where/orderBy/groupBy 条件构造，最终输出 SQL 片段与参数。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 条件构造基类 | `AbstractWrapper`：eq/ne/gt/ge/lt/le/between/in/like/isNull/and/or/nested 等全部条件方法 | `conditions/AbstractWrapper.java:751` |
| 片段合并器 | `MergeSegments`：把条件分发到 normal/groupBy/having/orderBy 四个片段列表，缓存拼接结果 | `conditions/segments/MergeSegments.java:34` |
| 片段接口 | `ISqlSegment`：所有 SQL 片段统一接口（`getSqlSegment()`） | `conditions/ISqlSegment.java` |
| 查询包装 | `QueryWrapper`/`LambdaQueryWrapper`：select 条件 | `conditions/query/QueryWrapper.java:153` |
| 更新包装 | `UpdateWrapper`/`LambdaUpdateWrapper`：set + where 条件 | `conditions/update/UpdateWrapper.java:157` |
| Lambda 列解析 | `AbstractLambdaWrapper`：把 SFunction 列解析成数据库列 | `conditions/AbstractLambdaWrapper.java:158` |
| 比较/函数接口 | `Compare`/`Func`/`Nested`/`Join`：条件方法的接口分组 | `conditions/interfaces/Compare.java:415` |
| 片段列表 | `NormalSegmentList`/`GroupBySegmentList`/`HavingSegmentList`/`OrderBySegmentList` | `conditions/segments/NormalSegmentList.java:98` |
| 工厂门面 | `Wrappers`：`new QueryWrapper/LambdaQueryWrapper/UpdateWrapper/LambdaUpdateWrapper` 静态工厂 | `toolkit/Wrappers.java:273` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `Wrapper` | `Wrapper.java:200` | 顶层包装抽象，持有 paramNameSeq/paramNameValuePairs |
| `AbstractWrapper` | `AbstractWrapper.java:751` | 全部条件方法实现，持 `MergeSegments expression`（`:80`） |
| `MergeSegments` | `MergeSegments.java:34` | 片段路由与拼接缓存 |
| `ISqlSegment` | `ISqlSegment.java` | SQL 片段统一接口 |
| `QueryWrapper`/`LambdaQueryWrapper` | `query/` | 查询条件 |
| `UpdateWrapper`/`LambdaUpdateWrapper` | `update/` | 更新条件 + set |
| `Wrappers` | `Wrappers.java:273` | 工厂 |

## 3. 关键调用链

**链式条件构造与 SQL 生成**：

1. 用户 `Wrappers.<User>lambdaQuery().eq(User::getName,"a").ge(User::getAge,18)` → `eq`（`AbstractWrapper.java:329`）→ `maybeDo(true, () -> appendSqlSegments(columnToSqlSegment, EQ, ...))`。
2. `appendSqlSegments`（`:628`）把列/关键字/参数占位符一组 ISqlSegment 交给 `expression.add(...)`。
3. `MergeSegments.add`（`MergeSegments.java:46`）按首段 `MatchSegment` 路由：ORDER_BY→orderBy、GROUP_BY→groupBy、HAVING→having，其余进 normal（`:49-57`），并置 `cacheSqlSegment=false`（`:58`）。
4. 运行期注入器取 `getSqlSegment`（`AbstractWrapper.java:647`）→ `MergeSegments.getSqlSegment`（`:62`）：缓存未失效直接返回；否则按 normal+groupBy+having+orderBy 顺序拼接（`:72`）并缓存（`:66`）。

**Lambda 列解析**：

- `AbstractLambdaWrapper.columnToString` 经 `LambdaUtils` 把 SFunction 解析成数据库列名（lambda-parser 叶子）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 条件开关 | 每个条件方法首参 `boolean condition`，false 则不拼接 | `AbstractWrapper.java:155` 等 |
| 参数占位 | `#{ew.paramNameValuePairs.MPGENVALx}` OGNL 占位，changeParamAlias 改别名 | `MergeSegments.java:86` |
| SQL 缓存 | `cacheSqlSegment` 位标记，add 后置 false，get 时重算 | `MergeSegments.java:44,63` |

## 5. 错误与重试语义

- 条件构造纯内存拼接，无外部 IO；无重试。
- 条件不满足（condition=false）时 `maybeDo` 直接跳过，不报错。
- SQL 拼接错误（列名缺失等）延迟到运行期由 MyBatis 解析抛出。

## 6. 并发细节

- Wrapper 实例非线程安全（普通 ArrayList 片段列表 + 可变缓存），一次请求一个实例。
- `MergeSegments` 内部列表非并发集合；`sqlSegment` 缓存位（`:44`）单线程内可见。
- 无线程池；Wrapper 在业务线程构造、随参数传入 Mapper。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/conditions/` 全部 28 文件 + `toolkit/Wrappers.java`。

**Out-of-Scope（不在本仓库源码内）**
- Lambda 列解析底层（LambdaUtils/反射，lambda-parser 叶子）。
- 注入器把片段拼进 SQL（sql-injector 叶子）。
- MyBatis 参数解析（`#{}` OGNL，第三方）。

## 8. 与相邻子系统交互

- 上游：业务代码经 `Wrappers` 工厂构造 Wrapper，传入 BaseMapper 方法。
- 本叶子 → 下游：`getSqlSegment()`/`getParamNameValuePairs()` 被 sql-injector/mapper-runtime 读取，拼成最终 SQL 与参数 Map。
- 数据流：**链式条件 → appendSqlSegments → MergeSegments 路由分发 → 缓存拼接 → SQL 片段**。

## 9. 语言专项适配口径（Java/JVM）

- **建造者模式**：所有条件方法返回 `this`（子类泛型自指），链式调用。
- **函数式**：`maybeDo(boolean, Supplier)` 条件化执行；`ISqlSegment` 可用 lambda。
- **lombok**：`@Getter`（MergeSegments）。
- **泛型自指**：AbstractWrapper 用 `<Children extends AbstractWrapper>` 链式返回子类。
- **库型无 main**。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 条件构造器架构图 | `conditions-wrapper-architecture.html` | architecture | showcase |
| MergeSegments 拼接数据流 | `conditions-wrapper-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 dataflow 图：链式条件 → appendSqlSegments → 四片段列表路由 → 缓存拼接是典型"源→加工→目的地"数据流。
未生成 sequence/lifecycle/workflow：条件构造是单线程内内存拼装（已 dataflow 化），无多参与方时序、无状态机、无审批流，按资源节省原则省略。
