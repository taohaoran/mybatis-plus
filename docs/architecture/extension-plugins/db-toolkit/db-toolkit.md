# 数据库便捷工具（db-toolkit）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 静态 CRUD 门面 | `Db` 全静态方法，无需注入 Mapper 即可按实体类做 save/getById/list/count/page 等 50+ 操作 | `toolkit/Db.java:61` 起 |
| 简单查询 | `SimpleQuery` 把实体列表快速转成 id 列表 / map / 分组 map | `toolkit/SimpleQuery.java` |
| SQL 辅助 | `SqlHelper`：会话获取、retBool/retCount、executeBatch 批量执行、saveOrUpdateBatch、getMapper | `toolkit/SqlHelper.java:45` |
| SQL 解析工具 | `SqlParserUtils`：从 SQL 解析表名等 | `toolkit/SqlParserUtils.java` |
| JDBC 工具 | `JdbcUtils.getDbType(url)` 从 jdbcUrl 判定 DbType | `toolkit/JdbcUtils.java` |
| 属性映射 | `PropertyMapper` 流式属性读取（见 interceptor-core 叶子） | `toolkit/PropertyMapper.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Db` | `toolkit/Db.java:61` | 静态门面；每个方法内部走 `SqlHelper.execute` 或链式包装 |
| `SqlHelper` | `toolkit/SqlHelper.java:45` | 会话/批量/结果转换公共辅助；静态 `FACTORY` |
| `SimpleQuery` | `toolkit/SimpleQuery.java` | 实体集合 → 常见 Map/List 转换 |
| `JdbcUtils` | `toolkit/JdbcUtils.java` | jdbcUrl → DbType |

## 3. 关键调用链

**链 1：Db.getById 静态调用**

1. `Db.getById(id, entityClass)`（`Db.java:238`）。
2. 内部经 `SqlHelper.execute(entityClass, sFunction)` 开 SqlSession、`getMapper` 取 BaseMapper、执行 `selectById`，finally 关会话（`SqlHelper.java:335-341`）。

**链 2：SqlHelper.executeBatch 批量提交**

1. `executeBatch(sqlSessionFactory, log, list, batchSize, biFunc)` 断言 `batchSize>=1`（`SqlHelper.java:205-206`）。
2. 逐条 `biFunc.apply(sqlSession, element)`；每满 `batchSize` 调 `sqlSession.flushStatements()` 提交，遍历 `BatchResult.updateCounts`：>0 累加行数、`SUCCESS_NO_INFO(-2)` 计成功、`EXECUTE_FAILED(-3)` 抛 `Batch execute failed`（`SqlHelper.java:212-229`）。

**链 3：getMapper 定位 Mapper**

1. `TableInfoHelper.getTableInfo(entityClass)` 取 TableInfo，无则抛异常；`tableInfo.getCurrentNamespace()` 反射取 Mapper 类（`SqlHelper.java:310-314`）。
2. 若有 `CompatibleSet`（Spring 环境）优先 `getBean(mapperClass)`，否则 `configuration.getMapper(mapperClass, sqlSession)`（`SqlHelper.java:315-322`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `SqlHelper.FACTORY` | 静态 SqlSessionFactory，Spring 启动时注入 | `SqlHelper.java:50` |
| `batchSize` | 默认 `Constants.DEFAULT_BATCH_SIZE` | `SqlHelper.executeBatch` |

## 5. 错误与重试语义

- `batchSize<1` 立即断言失败；批量执行遇 `EXECUTE_FAILED(-3)` 抛 RuntimeException（`SqlHelper.java:224-227`）。
- 找不到 TableInfo/Mapper 抛 `MybatisPlusException`。
- 无重试；会话 try-finally 必关。

## 6. 并发细节

- 每次操作新开 SqlSession 并在 finally 关闭，会话不跨线程。
- `SqlHelper.FACTORY` 静态只读。
- 批量用 BATCH ExecutorType，在单线程内 flush。
- 无自建长期线程池。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `Db`/`SimpleQuery`/`SqlHelper`/`SqlParserUtils`/`JdbcUtils`/`PropertyMapper`。

**Out-of-Scope（不在本仓库源码内）**

- `GlobalConfigUtils`/`TableInfoHelper`/`CompatibleHelper`——core 包。
- MyBatis SqlSession——第三方依赖。

## 8. 与相邻子系统交互

- 上游：业务代码无注入场景下静态调 `Db.xxx`。
- 本叶子 → 下游：`BaseMapper`/`SqlSession`；AR/仓储/Boot 自动装配复用 `SqlHelper`。

## 9. 语言专项适配口径（JVM）

- **静态门面（无状态工具类）**：`Db` 全静态方法，免注入 Mapper。
- **会话管理**：手动开/关 SqlSession，Spring 环境经 `CompatibleSet` 适配。
- **依赖方向**：extension → core。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| db-toolkit 架构图 | `db-toolkit-architecture.html` | architecture | showcase |
| Db 静态调用时序 | `db-toolkit-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 sequence：Db→SqlHelper→SqlSession→Mapper→DB 的调用链。
