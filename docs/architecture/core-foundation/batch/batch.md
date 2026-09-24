# 批量执行器（batch）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开 `core/batch/` + `toolkit/MybatisBatchUtils`：基于 MyBatis `ExecutorType.BATCH` 的批量插入/更新封装。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 批量执行核心 | `MybatisBatch<T>`：持有 SqlSessionFactory+数据列表+batchSize，`execute` 开启 BATCH 会话分批执行 | `batch/MybatisBatch.java:59,141` |
| 分批切片 | `CollectionUtils.split(dataList, batchSize)` 按批次切分，每批 flushStatements 后 commit | `batch/MybatisBatch.java:144-152` |
| 批量存或更新 | `saveOrUpdate`：用 `BatchSqlSession` 包装，predicate 判定每条走 insert 还是 update | `batch/MybatisBatch.java:187-205` |
| 批量会话包装 | `BatchSqlSession`：混合查询时每次 select 前先 `flushStatements`，避免一级缓存脏读 | `batch/BatchSqlSession.java:32,42` |
| 批量方法描述 | `BatchMethod<T>`：statementId + ParameterConvert 二元组 | `batch/BatchMethod.java:28` |
| 参数转换 SPI | `ParameterConvert<T>` 函数式接口：entity→mapper 方法参数 | `batch/ParameterConvert.java:23` |
| 方法构造器 | `MybatisBatch.Method<T>`：便捷构造 insert/updateById/update/deleteById 的 BatchMethod | `batch/MybatisBatch.java:223` |
| 静态门面 | `MybatisBatchUtils`：全静态重载 execute/saveOrUpdate，内部 new MybatisBatch 委托 | `toolkit/MybatisBatchUtils.java:33` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `MybatisBatch<T>` | `MybatisBatch.java:59` | 批量执行编排器，try-with-resources 管理 SqlSession |
| `MybatisBatch.Method<T>` | `MybatisBatch.java:223` | 按 Mapper 类名空间构造常用 BatchMethod |
| `BatchMethod<T>` | `BatchMethod.java:28` | 待执行方法的描述（statementId + 参数转换） |
| `ParameterConvert<T>` | `ParameterConvert.java:23` | entity→mapper 参数的转换函数式接口 |
| `BatchSqlSession` | `BatchSqlSession.java:32` | 包装 SqlSession，select 前强制 flushStatements |
| `MybatisBatchUtils` | `MybatisBatchUtils.java:33` | 静态门面，避免用户直接 new MybatisBatch |

## 3. 关键调用链

**批量插入（execute）**：

1. 用户 `new MybatisBatch<>(sqlSessionFactory, dataList).execute(method.insert())`（`MybatisBatch.java:117`）→ `execute(false, batchMethod)`（`:129`）→ `execute(false, statementId, parameterConvert)`（`:141`）。
2. try-with-resources 开 `ExecutorType.BATCH` 的 SqlSession（`:143`）→ `CollectionUtils.split` 按 batchSize 切片（`:144`）。
3. 每片内逐条 `sqlSession.update(statement, 参数)`（`:147`）→ 片末 `flushStatements()` 实际下推 JDBC batch（`:149`）→ 非 autoCommit 时 `commit()`（`:151`）。

**批量 saveOrUpdate**：

1. `saveOrUpdate(autoCommit, insertMethod, predicate, updateMethod)`（`:187`）→ 开 BATCH 会话，包成 `BatchSqlSession`（`:190`）。
2. 逐条 `insertPredicate.test(session, data)`（`:192`）：真→`sqlSession.insert`（`:193`），假→`sqlSession.update`（`:195`）。predicate 里若查库，`BatchSqlSession.selectOne` 会先 flushStatements（`BatchSqlSession.java:43`）保证看到已更新数据。
3. 最后 `flushStatements()` + 汇总 `session.getResultBatchList()`（`:198-199`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| batchSize | 默认 `Constants.DEFAULT_BATCH_SIZE`，构造可覆盖 | `MybatisBatch.java:70,76` |
| autoCommit | 依赖事务管理器；Spring 下 `SpringManagedTransaction` 控制无效，只能靠 datasource | `MybatisBatch.java:44,103` |
| 事务 | 需自行控制（注释 `<li>事务需要自行控制</li>`），flushStatements 才让 batch 有意义 | `MybatisBatch.java:38-40` |

## 5. 错误与重试语义

- SqlSession 用 try-with-resources 关闭（`:143`），异常时自动 close；无重试/退避。
- JDBC batch 执行失败由 `BatchExecutor` 抛异常，不吞；`flushStatements` 返回 `List<BatchResult>` 供调用方判断每条影响行数。
- 文档明确提示：saveOrUpdate 里混合 select 会触发一级缓存问题，故用 BatchSqlSession 包装（`:160-163`）。

## 6. 并发细节

- `MybatisBatch` 实例方法在单线程内顺序执行，非线程安全设计（一个实例一次批量任务）。
- `BatchSqlSession.resultBatchList`（`BatchSqlSession.java:36`）为普通 ArrayList，单线程内追加。
- BATCH 模式下 MyBatis `BatchExecutor` 内部攒 SQL 批量下推（第三方），本叶子不涉及线程切换。
- 无共享可变静态状态；`MybatisBatchUtils` 纯静态委托。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/batch/` 全部 4 文件 + `toolkit/MybatisBatchUtils.java`。

**Out-of-Scope（不在本仓库源码内）**
- `ExecutorType.BATCH`/`BatchExecutor`/`BatchResult`/`SqlSession`/`SqlSessionFactory`（MyBatis，第三方）。
- 事务管理（Spring `SpringManagedTransaction`/`JdbcTransaction`，第三方）。
- 实际 Mapper 语句（BaseMapper.insert 等，mapper-runtime/sql-injector 叶子）。

## 8. 与相邻子系统交互

- 上游：业务代码（或 service 层）调用 MybatisBatch/MybatisBatchUtils。
- 本叶子 → 下游：`sqlSessionFactory.openSession(ExecutorType.BATCH)`（`:143`）→ MyBatis BatchExecutor → JDBC batch。
- 数据流：**数据列表 → split 切片 → 逐条 update 攒批 → flushStatements 下推 → commit**。

## 9. 语言专项适配口径（Java/JVM）

- **MyBatis 扩展**：复用 MyBatis `ExecutorType.BATCH` 而非自研批量；通过 SqlSession 编程式 API（非 Mapper 代理）执行。
- **try-with-resources**：SqlSession 实现 AutoCloseable（`:143`），自动关闭。
- **函数式接口**：`ParameterConvert`、`BiPredicate<BatchSqlSession,T>`（saveOrUpdate 判定）是 JDK lambda 用法。
- **lombok**：`BatchMethod` 用 `@Getter`。
- **库型无 main**：嵌入业务应用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 批量执行架构图 | `batch-architecture.html` | architecture | showcase |
| 批量提交会话时序 | `batch-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：execute 内"开 BATCH 会话→切片→逐条攒批→flush→commit"是清晰的多步会话交互时序。
未生成 dataflow/lifecycle/workflow：分批+提交本质是会话时序（已 sequence 化），无单实体状态机、无多角色审批，按资源节省原则省略。
