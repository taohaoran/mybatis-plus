# Spring 集成（spring-integration）

> 本文是 `spring` 域下唯一叶子子系统文档。域级总览见 `../spring.md`。
> 本文展开 mybatis-plus-spring 模块如何把核心增强能力接入 Spring 容器。
>
> 源码基准：`mybatis-plus-spring` 分支 3.0，commit `bf67d907`（11 个 Java 文件，约 1584 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| SqlSessionFactory 工厂 Bean | 继承/拷贝 mybatis-spring 的 `SqlSessionFactoryBean`，强制用 `MybatisSqlSessionFactoryBuilder` 加载自定义 MybatisConfiguration | `MybatisSqlSessionFactoryBean.java:83` |
| 全局配置注入 | 持有 `GlobalConfig`，把 MetaObjectHandler/ISqlInjector/IdentifierGenerator 等挂到全局 | `MybatisSqlSessionFactoryBean.java:147` |
| Service 层接口 | 通用 CRUD 服务契约，`saveBatch/saveOrUpdateBatch` 等默认实现带 `@Transactional` | `service/IService.java:32` |
| Service 实现基类 | 继承 `CrudRepository`，持有 `baseMapper` 实现 IService | `service/impl/ServiceImpl.java:28` |
| Repository 抽象 | `CrudRepository<M,T>` 抽象基类，暴露 `getBaseMapper()` | `repository/CrudRepository.java:40` |
| ActiveRecord 模型 | 实体继承 `Model<T>` 获得 insert/delete/updateById 等 AR 能力 | `activerecord/Model.java:32` |
| DDL 执行 | 实现 `IDdl`，按 DataSource 执行 SQL 脚本文件 | `ddl/SimpleDdl.java:31`（`runScript()` 行 37） |
| 静态 SQL 运行器 | 继承 `AbstractSqlRunner`，`SqlRunner.db()` 静态入口 | `toolkit/SqlRunner.java:47`（`db()` 行 86） |
| Spring 上下文感知 | 实现 `ApplicationContextAware`，持有静态上下文供静态工具取用 | `MybatisPlusApplicationContextAware.java` |
| Spring 兼容集合 | SPI 兼容集合 | `spi/SpringCompatibleSet.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MybatisSqlSessionFactoryBean` | `MybatisSqlSessionFactoryBean.java:83` | 实现 `FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ContextRefreshedEvent>, ApplicationContextAware`；`sqlSessionFactoryBuilder = new MybatisSqlSessionFactoryBuilder()`（行 102） |
| `IService<T>` | `service/IService.java:32` | 业务 Service 顶层接口，继承 `IRepository<T>` |
| `ServiceImpl<M,T>` | `service/impl/ServiceImpl.java:28` | `baseMapper` 注入与 IService 默认方法实现 |
| `CrudRepository<M,T>` | `repository/CrudRepository.java:40` | 暴露 `getBaseMapper()`（行 46），事务方法在子类 |
| `Model<T>` | `activerecord/Model.java:32` | AR 基类，继承 `AbstractModel<T>` |
| `SqlRunner` | `toolkit/SqlRunner.java:47` | 静态 CRUD 运行器，`DEFAULT` 单例（行 57） |
| `SimpleDdl` | `ddl/SimpleDdl.java:31` | `IDdl` 的 Spring 版脚本执行 |

## 3. 关键调用链

1. **SqlSessionFactory 装配链**：Spring 容器在 bean 初始化时调 `MybatisSqlSessionFactoryBean`（实现 `InitializingBean`）→ 其 `buildSqlSessionFactory()` 强制 `new MybatisSqlSessionFactoryBuilder()`（行 102，注释说明拷贝自 mybatis-spring 的 `SqlSessionFactoryBean` 并重写该方法加载自定义配置，行 75–78）→ 产出 `SqlSessionFactory`（行 104）；工厂同时是 `ApplicationContextAware`/`ApplicationListener`，在容器刷新时拿到 `GlobalConfig` 并装配增强组件。
2. **Service 调用链**：业务 `ServiceImpl<Mapper,Entity>` 注入 `baseMapper`（经 `CrudRepository.getBaseMapper()`，行 46）→ `IService.saveBatch()` 默认方法（`IService.java:39–40`）带 `@Transactional(rollbackFor=Exception.class)` 委托 baseMapper 批量执行。
3. **AR 调用链**：实体继承 `Model<T>`（行 32）→ 直接调 `insert()/deleteById()`，底层经 `AbstractModel` 取 `SqlSession` 执行。
4. **静态运行器链**：`SqlRunner.db()`（行 86）返回 `DEFAULT`（行 57）静态单例，经 `AbstractSqlRunner` 执行原生 SQL；多库场景按 class 取 `SqlQuery`（行 91 注释）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `defaultEnumTypeHandler` | 可设枚举类型处理器 | `MybatisSqlSessionFactoryBean.java:117` |
| `defaultScriptingLanguageDriver` | 可设自定义 LanguageDriver | 行 127 |
| `vfs` / `plugins` / `databaseIdProvider` | 透传给 MybatisConfiguration | 行 132/231/185 |
| `globalConfig` | 可显式注入全局配置；否则从容器/上下文获取 | 行 147 |
| `environment` | 强制取 `SqlSessionFactoryBean.class.getSimpleName()`（行 106），移除可配置 | 行 106 |

## 5. 错误与重试语义

- 工厂 bean 装配失败抛 Spring `BeanCreationException`，由容器上抛；本叶子无自定义重试。
- `IService.saveBatch` 等默认方法声明 `@Transactional(rollbackFor=Exception.class)`，运行期异常触发回滚。
- `SqlRunner` 静态工具异常直接上抛为运行时异常。
- 无退避/限流：本模块是 Spring 集成库，错误语义交给 Spring 事务与容器。

## 6. 并发细节

- **Spring 单例 bean**：`MybatisSqlSessionFactoryBean`、`SqlRunner.DEFAULT` 为单例，`SqlSessionFactory` 线程安全（MyBatis 保证）。
- `SqlSession` 由 Spring 事务管理器按事务线程绑定（`SqlSessionTemplate` 线程安全代理，mybatis-spring 提供，不在本仓库）。
- `MybatisPlusApplicationContextAware` 持静态 `ApplicationContext`，属容器启动后只读。
- 无线程池创建；无显式锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `mybatis-plus-spring` 全部 11 个文件：工厂 Bean、Service/Repository/AR/DDL/SqlRunner/上下文感知。

**Out-of-Scope（不在本仓库源码内）**
- mybatis-spring 的 `SqlSessionFactoryBean`/`SqlSessionTemplate`、Spring 容器与事务管理器均为第三方依赖，不在本仓库源码内。
- `GlobalConfig`、`BaseMapper`、`AbstractSqlRunner`、`AbstractModel`、`IDdl` 定义在 core/extension 模块（相邻分片分析）。
- Spring Boot 自动配置如何创建本叶子的工厂 Bean 见 `../boot-starter/boot-autoconfigure.md`。

## 8. 与相邻子系统交互

- **上游**：boot-starter 的 `MybatisPlusAutoConfiguration` 创建 `MybatisSqlSessionFactoryBean` 并设置 DataSource/全局配置。
- **本叶子 → core/extension**：依赖 `MybatisSqlSessionFactoryBuilder`、`GlobalConfig`、`BaseMapper`、`AbstractSqlRunner`、`IDdl`。
- **下游（业务）**：用户 Service 继承 `ServiceImpl`、实体继承 `Model`、Mapper 继承 `BaseMapper`。

## 9. 语言专项适配口径（JVM）

- **Spring 生命周期接口**：工厂 Bean 同时实现 `FactoryBean/InitializingBean/ApplicationListener/ApplicationContextAware`，是典型 Spring 集成 bean；依赖方向 extension → spring（facts.md §3）。
- **事务边界**：`IService` 默认方法用注解声明式事务，不手写编程式事务。
- **库型无 main**：本模块是被 Spring Boot 自动配置装配的库，不独立启动进程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Spring 集成架构图 | `spring-integration-architecture.html` | architecture | showcase |
| SqlSessionFactoryBean 装配时序 | `spring-integration-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子第二图选用 sequence：工厂 Bean 的容器装配是"容器→FactoryBean→MybatisSqlSessionFactoryBuilder→GlobalConfig→SqlSessionFactory"的时序调用链。
