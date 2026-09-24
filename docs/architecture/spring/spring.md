# Spring 集成（spring）域总览

> 本域包含 1 个叶子子系统；详情见对应文档。
> 源码基准：`mybatis-plus-spring` 分支 3.0，commit `bf67d907`（11 个 Java 文件，约 1584 行，为最小域）。

## 1. 域职责

Spring 集成域把 MyBatis-Plus 的核心增强能力桥接到 Spring 容器：核心是 `MybatisSqlSessionFactoryBean`——它拷贝 mybatis-spring 的 `SqlSessionFactoryBean` 并重写 `buildSqlSessionFactory()`，强制使用 `MybatisSqlSessionFactoryBuilder` 加载增强配置。同时提供 `IService/ServiceImpl`、ActiveRecord `Model`、`CrudRepository`、静态 `SqlRunner` 与 `SimpleDdl`，让业务代码以 Spring 惯用法使用增强 CRUD。模块 `api` 依赖 mybatis-plus-extension。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| Spring 集成 | [spring-integration.md](spring-integration/spring-integration.md) | [架构图](spring-integration/spring-integration-architecture.html) | [时序](spring-integration/spring-integration-sequence.html) | 工厂 Bean 桥接核心 + Service/AR/SqlRunner |

> 本域仅 1 叶，按 diagram-policy 规则叶子图与域级图合并计算：域总览引用下方域级架构图与叶子两张图，共 3 张，满足域级 ≥3 张要求。

## 3. 域级机制细节

- **工厂 Bean 多接口实现**：`MybatisSqlSessionFactoryBean` 同时实现 `FactoryBean<SqlSessionFactory>`、`InitializingBean`、`ApplicationListener<ContextRefreshedEvent>`、`ApplicationContextAware`，在容器生命周期各阶段接入。
- **声明式事务**：`IService.saveBatch` 等默认方法带 `@Transactional(rollbackFor=Exception.class)`，由 Spring 事务管理器回滚。
- **单例共享**：`SqlSessionFactory`、`SqlRunner.DEFAULT` 为容器单例，`SqlSession` 由 mybatis-spring 按线程/事务绑定。

## 4. 域级图

![Spring 集成域架构图](spring-architecture.html)

配合叶子图 [spring-integration-architecture.html](spring-integration/spring-integration-architecture.html) 与 [spring-integration-sequence.html](spring-integration/spring-integration-sequence.html)，覆盖域内静态拓扑与工厂 Bean 装配时序。
