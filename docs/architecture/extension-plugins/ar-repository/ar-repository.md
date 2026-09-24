# ActiveRecord 与仓储（ar-repository）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
> 本文覆盖 ActiveRecord 模式 `AbstractModel`、仓储 `IRepository`/`AbstractRepository`，以及 spring 模块的 `Model` 实体基类。
>
> 源码基准：`mybatis-plus-extension` + `mybatis-plus-spring`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| ActiveRecord CRUD | `AbstractModel<T>` 实体自身带 `insert/updateById/deleteById/selectById/selectList/selectCount/selectPage` 等方法 | `activerecord/AbstractModel.java:45` |
| AR 会话管理 | 每个 AR 操作开 `SqlSession`、try-finally 关闭，按实体类定位 Mapper 命名空间 | `activerecord/AbstractModel.java:242-280` |
| 仓储接口 | `IRepository<T>` 默认方法封装单表 CRUD、批量、链式入口 `query()/lambdaQuery()/update()` | `repository/IRepository.java:24` |
| 仓储抽象基类 | `AbstractRepository` 实现 `saveBatch/updateBatchById/saveOrUpdateBatch` 等批量方法 | `repository/AbstractRepository.java` |
| Spring AR 模型 | `com.baomidou.mybatisplus.spring.activerecord.Model` 继承 `AbstractModel`，供 Spring 环境实体继承 | `mybatis-plus-spring/.../activerecord/Model.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AbstractModel<T>` | `activerecord/AbstractModel.java:45` | AR 基类；`sqlSession()` 开会话、`sqlStatement(SqlMethod)` 拼 Mapper 方法 id |
| `IRepository<T>` | `repository/IRepository.java:24` | 仓储接口，default 方法委托 `getBaseMapper()` |
| `AbstractRepository<T>` | `repository/AbstractRepository.java` | 批量方法实现基类 |
| `Model<T>` | spring `activerecord/Model.java` | Spring 环境 AR 实体基类 |
| `SqlHelper` | extension `toolkit/SqlHelper.java` | 会话/批量执行公共辅助（见 db-toolkit 叶子） |

## 3. 关键调用链

**链 1：AR insert 调用链**

1. 业务实体继承 `Model`，调 `entity.insert()`（`AbstractModel.java:54`）。
2. `sqlSession()` 经 `SqlHelper.sqlSession(entityClass)` 从全局 `SqlSessionFactory` 开会话（`AbstractModel.java:242-244`）。
3. `sqlStatement(SqlMethod.INSERT_ONE)` → `SqlHelper.table(entityClass).getSqlStatement("insert")` 取 Mapper 方法 id（`AbstractModel.java:260-263`）。
4. `sqlSession.insert(statement, this)`，`SqlHelper.retBool` 转 boolean，`finally closeSqlSession`（`AbstractModel.java:55-60`）。

**链 2：AR 插入或更新判断**

- `insertOrUpdate()`：主键为空或 `selectById(pkVal())` 为 null 则 `insert()`，否则 `updateById()`（`AbstractModel.java:66-68`）。

**链 3：IRepository 链式入口**

- `IRepository.query()` → `ChainWrappers.queryChain(getBaseMapper())` 返回 `QueryChainWrapper`，链式 `.eq(...).one()` 直接执行（`IRepository.java:492-494`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `DEFAULT_BATCH_SIZE` | `Constants.DEFAULT_BATCH_SIZE`（批量默认条数） | `IRepository.java:29` |
| AR 前置条件 | 必须存在对应 Mapper 且继承 `BaseMapper` 并可用 | `AbstractModel.java:37-39` 注释 |

## 5. 错误与重试语义

- `updateById/selectById/deleteById` 主键为空时 `Assert.isFalse` 立即抛断言（`AbstractModel.java:117,175,88`）。
- `closeSqlSession` 委托 `CompatibleHelper.getCompatibleSet().closeSqlSession`，确保 Spring 环境正确处理会话事务绑定。
- 无重试；DB 异常直接抛出。

## 6. 并发细节

- 每次 AR 操作新开并关闭 `SqlSession`，会话不跨线程共享。
- `SqlHelper.FACTORY` 是静态全局 `SqlSessionFactory`，由 Spring 初始化后赋值，运行期只读。
- `AbstractModel.entityClass` 是 `transient final`，每个实体实例独立。
- 无线程池；批量方法在调用线程用 BATCH 执行器流式 flush。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `AbstractModel`、`IRepository`/`AbstractRepository`、spring `Model`。

**Out-of-Scope（不在本仓库源码内）**

- `ServiceImpl`/`IService`（Service 层封装）——spring 域 `spring-integration` 叶子。
- `BaseMapper`——core `mapper` 包（core-mapping）。
- Spring 容器/事务管理——第三方依赖。

## 8. 与相邻子系统交互

- 上游：业务实体继承 `Model`；业务 Service 继承 `AbstractRepository`。
- 本叶子 → 下游：经 `SqlSession`/`BaseMapper` 执行 MappedStatement；批量委托 `SqlHelper.executeBatch`。
- 与 `db-toolkit`：共用 `SqlHelper`；与 `chain-conditions`：仓储链式入口构造 `QueryChainWrapper`。

## 9. 语言专项适配口径（JVM）

- **ActiveRecord 模式**：实体即自身带持久化方法，区别于 DataMapper（Mapper 接口分离）。
- **会话生命周期**：AR 模式手动开/关 SqlSession，Spring 环境经 `CompatibleSet` 适配事务同步。
- **依赖方向**：extension(AR/repository) → spring(Model) → core。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| AR 与仓储架构图 | `ar-repository-architecture.html` | architecture | showcase |
| AR insert 调用时序 | `ar-repository-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 sequence：AR insert 是"实体→SqlHelper→SqlSession→Mapper→DB"的多方时序。
