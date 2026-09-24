# jsqlparser 域（jsqlparser）域总览

> 本域基于 JSQLParser 做 SQL AST 级解析与多表拦截改写，包含 2 个叶子子系统；各叶子详情见对应文档。
> 源码基准：mybatis-plus-jsqlparser-support 分支 3.0，commit `bf67d907478c724120bf76292da54abf9e73c2b3`。

## 1. 域职责

jsqlparser 域负责「把 SQL 文本解析成 AST，再按规则改写后执行」：

- **解析与缓存**：`JsqlParserGlobal` 作为全局门面，先查 `JsqlParseCache`（默认 Caffeine 实现，可配 JDK/FST/Fury 序列化），未命中时经 `JsqlParserThreadPool`（来自 `mybatis-plus-jsqlparser-common`）异步提交给 JSQLParser 解析，结果序列化回填缓存。
- **智能拦截改写**：`BaseMultiTableInnerInterceptor` 及其子类（多租户 TenantLine、物理分页、全表攻击阻断 BlockAttack、数据权限）遍历 AST 的表名，逐表注入 where 条件或分页片段，再 `toString` 还原 SQL。

**三版本兼容**：本域有三套实现——`mybatis-plus-jsqlparser`（聚合版，基于 jsqlparser 5.2，主分析对象）、`mybatis-plus-jsqlparser-5.0`、`mybatis-plus-jsqlparser-4.9`（兼容 JDK8）；三者共用 `mybatis-plus-jsqlparser-common`（线程池与枚举）。接口与改写逻辑一致，差异仅在 JSQLParser 库版本与 AST API 细节。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流/时序图 | 职责一句话 |
|------|------|--------|--------------|-----------|
| 解析缓存 | [parser-cache.md](parser-cache/parser-cache.md) | [架构图](parser-cache/parser-cache-architecture.html) | [数据流](parser-cache/parser-cache-dataflow.html) | 全局解析门面、Caffeine 缓存与序列化 |
| 智能拦截器 | [sql-interceptors.md](sql-interceptors/sql-interceptors.md) | [架构图](sql-interceptors/sql-interceptors-architecture.html) | [数据流](sql-interceptors/sql-interceptors-dataflow.html) | 多表遍历注入：租户/分页/阻断/数据权限 |

## 3. 域级机制细节

- **解析缓存命中率是性能关键**：同一条 SQL 文本第二次起直接命中缓存的 AST 字节数组，避免重复解析；缓存 key 为 SQL 文本本身。
- **改写入口统一在 `beforeQuery`**：所有多表拦截器继承 `BaseMultiTableInnerInterceptor`，对 select/insert/update/delete 的表名逐表应用 `TenantLineHandler` 等策略；忽略表清单（`@InterceptorIgnore`）在遍历前判定。
- **版本隔离**：4.9 / 5.0 / 聚合 5.2 三个 Maven 模块各自打包对应 JSQLParser 版本，业务按 JDK 与需求选其一引入，common 模块不绑定 JSQLParser API。

## 4. 域级图

![jsqlparser 域架构](jsqlparser-architecture.html)
![jsqlparser 域数据流](jsqlparser-dataflow.html)
![解析缓存时序](jsqlparser-sequence.html)
