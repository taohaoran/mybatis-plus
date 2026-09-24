# SQL 解析与缓存（parser-cache）

> 本文是 `jsqlparser` 域下的叶子子系统文档。域级总览见 `../jsqlparser.md`。
>
> 源码基准：`mybatis-plus-jsqlparser-support`（聚合模块 `mybatis-plus-jsqlparser` 用 jsqlparser 5.2；另有 `jsqlparser-4.9` 兼容 JDK8、`jsqlparser-5.0` 两套版本实现），分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径（聚合 5.2 模块） |
|---|---|---|
| 解析支持基类 | `JsqlParserSupport`：`parserSingle/parserMulti` 调全局解析器，按 Statement 类型分派 processInsert/Select/Update/Delete | `extension/parser/JsqlParserSupport.java:39` |
| 全局解析门面 | `JsqlParserGlobal`：持有解析函数 `parserSingleFunc/parserMultiFunc`、可选缓存 `JsqlParseCache`、解析线程池 | `extension/parser/JsqlParserGlobal.java:32` |
| 解析函数 SPI | `JsqlParserFunction`：函数式接口封装 parse 调用 | `extension/parser/JsqlParserFunction.java` |
| 解析缓存 SPI | `JsqlParseCache`：put/get Statement/Statements 四方法 | `extension/parser/cache/JsqlParseCache.java:27` |
| Caffeine 缓存抽象 | `AbstractCaffeineJsqlParseCache`：缓存 `String→byte[]`，序列化/反序列化抽象，支持异步写 | `extension/parser/cache/AbstractCaffeineJsqlParseCache.java:36` |
| 三种序列化实现 | JDK 原生 / FST / Fury 三种 Caffeine 缓存 | `cache/JdkSerialCaffeineJsqlParseCache.java` 等 |
| 解析线程池 | `JsqlParserThreadPool`（common 模块）：默认 `(CPU+1)/2` 线程，支持关闭钩子 | `jsqlparser/JsqlParserThreadPool.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `JsqlParserSupport` | `parser/JsqlParserSupport.java:39` | 模板方法分派；子类覆写 processXxx 做 AST 改写 |
| `JsqlParserGlobal` | `parser/JsqlParserGlobal.java:32` | 全局静态门面：parse/parseStatements 带缓存旁路 |
| `JsqlParseCache` | `parser/cache/JsqlParseCache.java:27` | 缓存 SPI |
| `AbstractCaffeineJsqlParseCache` | `parser/cache/AbstractCaffeineJsqlParseCache.java:36` | Caffeine + 序列化抽象；反序列化失败自动 invalidate |
| `JsqlParserThreadPool` | common `jsqlparser/JsqlParserThreadPool.java` | 解析线程池 |

## 3. 关键调用链

**链 1：带缓存的 SQL 解析**

1. 拦截器调 `JsqlParserSupport.parserSingle(sql, obj)`（`JsqlParserSupport.java:46`）。
2. 内部 `JsqlParserGlobal.parse(sql)`（`JsqlParserGlobal.java:115`）：若 `jsqlParseCache==null` 直接 `parserSingleFunc.apply(sql)`；否则先 `jsqlParseCache.getStatement(sql)`。
3. 缓存命中直接返回；未命中则 `parserSingleFunc.apply(sql)`（默认 `CCJSqlParserUtil.parse(sql, executorService, null)`）解析后 `putStatement` 回填（`JsqlParserGlobal.java:119-124`）。
4. 回到 `processParser` 按 `instanceof Insert/Select/Update/Delete` 分派子类覆写的 processXxx，最后 `statement.toString()` 输出改写后 SQL（`JsqlParserSupport.java:86-104`）。

**链 2：Caffeine 缓存读写**

1. `get(sql)`：`cache.getIfPresent(sql)` 取 byte[]，命中则 `deserialize`；反序列化异常 `cache.invalidate(sql)` 并记 error，返回 null（`AbstractCaffeineJsqlParseCache.java:81-92`）。
2. `put(sql, value)`：先 `serialize(value)` 成 byte[]；`async=true` 时 `CompletableFuture.runAsync` 异步写 Caffeine（可指定 executor），否则同步写（`AbstractCaffeineJsqlParseCache.java:100-111`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `jsqlParseCache` | null（不缓存）；用户可 set 一个 Caffeine 缓存实例 | `JsqlParserGlobal.java:64` |
| `parserSingleFunc/parserMultiFunc` | 默认 `CCJSqlParserUtil.parse(sql, executor, null)` | `JsqlParserGlobal.java:58-61` |
| 解析线程池 | 默认 `JsqlParserThreadPool.getDefaultThreadPoolExecutor()`，线程数 `(CPU+1)/2` | `JsqlParserGlobal.java:111-113` |
| `async` 写缓存 | false；true 时异步写 Caffeine | `AbstractCaffeineJsqlParseCache.java:40` |
| 序列化选型 | JDK/FST/Fury 三选一，Fury/FST 需引入对应依赖 | `cache/*SerialCaffeineJsqlParseCache.java` |

## 5. 错误与重试语义

- JSQLParserException 被包装成 `MybatisPlusException("Failed to process, Error SQL: ...")` 抛出（`JsqlParserSupport.java:53-55`）。
- 缓存反序列化失败：自动 `invalidate` 该 key 并记 error，下次重新解析（`AbstractCaffeineJsqlParseCache.java:86-89`）。
- 无重试；解析失败即中断该 SQL。

## 6. 并发细节

- `JsqlParserGlobal` 静态字段 `parserSingleFunc/parserMultiFunc/jsqlParseCache` 用 `@Setter`，启动期设置后运行期只读。
- 解析线程池 `ExecutorService` 由用户设置并自行关闭；`setExecutorService(..., shutdownHook)` 可注册 JVM 关闭钩子（`JsqlParserGlobal.java:98-103`）。
- Caffeine 缓存本身线程安全；异步写用 `CompletableFuture.runAsync`。
- `JsqlParseCache` 多线程并发 get/put 由 Caffeine 保证。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `parser/` 与 `parser/cache/` 全部解析/缓存类；common 模块 `JsqlParserThreadPool`。
- 三套版本实现（4.9/5.0/聚合5.2）共用本结构，仅依赖的 jsqlparser 库版本不同。

**Out-of-Scope（不在本仓库源码内）**

- JSQLParser 库本身（`net.sf.jsqlparser.*`）——第三方依赖，不在本仓库源码内。
- Caffeine/FST/Fury 库——第三方依赖。
- 具体 SQL 改写拦截器（租户/分页等）——见 `../sql-interceptors/`。

## 8. 与相邻子系统交互

- 上游：`sql-interceptors` 域各 InnerInterceptor 继承 `JsqlParserSupport`，调 `parserSingle/parserMulti`。
- 本叶子 → 下游：`CCJSqlParserUtil` 解析 SQL 成 AST；Caffeine 缓存序列化 AST。
- 版本差异：`jsqlparser-4.9`（兼容 JDK8）、`jsqlparser-5.0`、聚合 `jsqlparser`（5.2）三套实现类同，按 jsqlparser 版本 API 差异分别编译；`jsqlparser-common` 放跨版本共用的线程池与枚举。

## 9. 语言专项适配口径（JVM）

- **静态门面 + 可插拔函数**：`JsqlParserGlobal` 全局可替换解析函数与缓存，适配不同 jsqlparser 版本。
- **并发**：解析用独立线程池（JSQLParser 解析 CPU 密集）；缓存异步写。
- **多版本矩阵**：同一套源码按 jsqlparser 4.9/5.0/5.2 三套产物，common 模块抽共用部分。
- **依赖方向**：jsqlparser-support → extension（InnerInterceptor SPI）→ core。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 解析缓存架构图 | `parser-cache-architecture.html` | architecture | showcase |
| SQL 解析缓存数据流 | `parser-cache-dataflow.html` | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 dataflow：SQL 文本→查缓存→未命中解析→序列化→Caffeine→反序列化返回 AST，是典型数据管道。
