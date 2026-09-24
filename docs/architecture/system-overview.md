# mybatis-plus 系统级总览

> 源码基准：`mybatis-plus`（com.baomidou，Apache-2.0），版本 3.5.17，分支 3.0，commit `bf67d907`。
> 构建：Gradle 多模块，Java 21 编译 / release 8，少量 Kotlin（extension 模块 5 个 main .kt）。
> 规模：main 源码约 497 Java 文件 + 5 Kotlin，约 6.7 万行，覆盖 7 域 31 叶子子系统。

## 1. 项目概述

**定位**（README-zh 原文）："Mybatis 增强工具包 - 只做增强不做改变，简化 CRUD 操作。"

mybatis-plus 是一个基于 MyBatis 3.5.19 的增强框架，核心设计哲学是"只做增强不做改变"——在不修改 MyBatis 核心源码的前提下，通过继承、包装和 SPI 扩展点，为开发者提供通用 CRUD、条件构造器、分页、多租户、乐观锁、逻辑删除、代码生成等开箱即用的能力。项目为**库型**（无 main 入口），以依赖方式嵌入用户的 Spring/Spring Boot 应用。

### 代码规模

| 模块 | Java 文件 | Java 行数 | Kotlin 文件 | 说明 |
|---|---|---|---|---|
| mybatis-plus-annotation | 16 | 1,187 | 0 | 注解契约基础 |
| mybatis-plus-core | 170 | 22,975 | 0 | 核心增强（最大模块） |
| mybatis-plus-extension | 93 | 9,184 | 5 | 扩展插件与工具 |
| mybatis-plus-generator | 117 | 14,948 | 0 | 代码生成器 |
| mybatis-plus-spring | 11 | 1,584 | 0 | Spring 集成 |
| mybatis-plus-jsqlparser-support | 64 | 12,652 | 0 | SQL 解析（可选链路） |
| spring-boot-starter | 26 | 4,142 | 0 | Boot2/3/4 启动器 + AOT |
| **合计** | **约 497** | **约 6.7 万** | **5** | |

## 2. 功能总览

按 7 域 31 叶组织：

| 域 | 叶子数 | 核心能力 |
|---|---|---|
| **core-foundation** | 7 | 注解契约、工具库、Lambda 解析、主键生成、类型处理器、批量执行、核心支撑 |
| **core-mapping** | 5 | 条件构造器 Wrapper、SQL 注入器（20 方法）、表元数据、Mapper 代理运行时、配置与工厂引导 |
| **extension-plugins** | 10 | 拦截器核心、分页方言、扩展注入方法、DDL 执行、ActiveRecord/Repository、JSON 处理器、链式条件、Db 工具、数据库序列键、脚本语言与 p6spy |
| **jsqlparser** | 2 | SQL AST 解析与缓存（FST/Fury/Caffeine）、多租户/数据权限/防全表更新/非法 SQL 等智能拦截器 |
| **generator** | 4 | 生成器配置体系、数据库元数据查询、模板引擎（Velocity/Freemarker/Beetl/Enjoy）、AutoGenerator 主流程 |
| **spring** | 1 | IService/ServiceImpl、MybatisSqlSessionFactoryBean、SqlRunner、CrudRepository、Spring 兼容层 |
| **boot-starter** | 2 | Boot2/3/4 自动配置、native-image AOT、测试自动配置 |

### 核心功能清单（README Features 摘录）

1. 完全兼容 MyBatis（只做增强不做改变）
2. 启动即自动配置、自动注入基本 CRUD（BaseMapper/IService）
3. 通用 CRUD + 强大条件构造器（Wrapper/LambdaWrapper）
4. 多种主键策略（雪花 IdWorker / Sequence / 数据库序列）
5. Lambda 风格 API（序列化 Lambda 解析列名）
6. 高度可定制代码生成器（AutoGenerator/FastAutoGenerator）
7. 内置物理分页插件（14 种方言）
8. SQL 注入防御（SqlInjectionUtils、BlockAttack 全表 update/delete 阻断）
9. ActiveRecord 模式（实体继承 Model）
10. 可插拔自定义接口注入（ISqlInjector，Write once use anywhere）
11. 内置扩展：乐观锁、逻辑删除、多租户、数据权限、数据变更记录、动态表名、p6spy 性能分析、JSON 类型处理器

## 3. 解决的问题

| 用户痛点 | mybatis-plus 解法 | 对应叶子 |
|---|---|---|
| 每张表手写 CRUD SQL，重复劳动 | SqlInjector 自动注入 20+ 通用 MappedStatement，BaseMapper 开箱即用 | sql-injector |
| 动态查询条件拼接繁琐易错 | Wrapper/LambdaWrapper 条件构造器，MergeSegments 合并 SQL 片段 | conditions-wrapper |
| 分页需手写 count SQL 和 limit 方言 | PaginationInnerInterceptor 自动改写 SQL，14 种方言适配 | pagination |
| 多租户/数据权限需侵入业务 SQL | jsqlparser AST 解析后自动追加 tenant_id / 数据权限条件 | sql-interceptors |
| 实体类与数据库字段映射重复配置 | 注解驱动 TableInfo 元数据，驼峰自动映射，Lambda 引用列名 | table-metadata, lambda-parser |
| 主键生成需各自实现 | IdentifierGenerator SPI，内置雪花/序列/数据库键，可自定义 | id-generator, key-generators |
| 代码生成器模板固定不灵活 | AutoGenerator 多模板引擎 + 策略配置 + 自定义注入 | generator-core, generator-engine |
| 全表 update/delete 误操作风险 | BlockAttackInnerInterceptor 解析 SQL 阻断无 where 的修改 | sql-interceptors |
| Spring Boot 集成需大量配置 | boot-starter 自动配置，mybatis-plus.* 属性一键绑定 | boot-autoconfigure |

## 4. 系统边界

### 上边界（用户接入层）
- 用户通过 `BaseMapper<T>` / `IService<T>` / `Mapper` 接口调用，或使用 `Db` 静态工具、`ActiveRecord` 模式
- Spring Boot 用户引入 `mybatis-plus-boot-starter`（boot2/3/4），通过 `mybatis-plus.*` 配置属性驱动
- 代码生成器用户通过 `AutoGenerator` / `FastAutoGenerator` API 独立运行

### 下边界（基础设施层）
- **MyBatis 3.5.19**：核心依赖，mybatis-plus 通过继承 `Configuration`/`MapperRegistry`/`SqlSessionFactoryBuilder` 等 12 个类进行增强，不修改 MyBatis 源码（不在本仓库源码内）
- **JDBC / 数据源**：通过 MyBatis Executor 最终执行 JDBC；代码生成器直接使用 JDBC DatabaseMetaData
- **关系型数据库**：MySQL、Oracle、PostgreSQL、SQLServer、DB2、H2、SQLite、DM、Kingbase 等（不在本仓库源码内）

### 内边界（模块依赖方向）
```
annotation → core → extension → spring → generator
                                    ↘ boot-starter（自动装配）
extension → jsqlparser-support（可选独立链路，通过 InnerInterceptor SPI 接入）
```
- `annotation`：纯注解契约，无运行逻辑，仅依赖 MyBatis
- `core`：核心增强，api 依赖 annotation + MyBatis
- `extension`：扩展插件，api 依赖 core
- `spring`：Spring 集成，api 依赖 extension
- `generator`：代码生成器，依赖 spring
- `jsqlparser-support`：可选链路，依赖 extension 的插件接口，有 4.9/5.0/聚合 5.2 三套版本实现
- `boot-starter`：启动器，依赖 spring + 聚合 mybatis-plus 模块

### 侧边界
- **jsqlparser 为可选依赖**：不引入 jsqlparser-support 时，分页/多租户等基于 SQL AST 的拦截器不可用，但基础 CRUD/Wrapper/乐观锁等仍正常工作
- **多版本 starter 矩阵**：boot2（Spring Boot 2.7）、boot3（3.x，jakarta 命名空间）、boot4（4.x）三套近同构 `MybatisPlusAutoConfiguration`
- **native-image AOT**：GraalVM 原生镜像支持通过 `mybatis-plus-spring-boot-native-image` 模块提供

### 不做什么
- 不做连接池管理（依赖用户配置的 DataSource）
- 不做事务管理（依赖 Spring 事务管理器或 MyBatis 事务）
- 不做分库分表（需借助 ShardingSphere 等外部组件）
- 不做读写分离（需外部中间件）
- 不修改 MyBatis 核心源码（通过继承和包装实现增强）

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

架构图展示了 mybatis-plus 的模块分层与依赖方向：

1. **主依赖链**（annotation → core → extension → spring → generator）：单向 api 依赖，下层不感知上层，保证模块可独立使用
2. **可选链路**（extension → jsqlparser）：jsqlparser-support 通过 `InnerInterceptor` SPI 接入，不影响主链路
3. **启动装配**（spring → boot-starter）：boot-starter 在运行时自动装配 `MybatisSqlSessionFactoryBean` 和增强组件
4. **外部依赖**：MyBatis 3.5.19 为核心增强目标，Spring 框架为集成目标，关系型数据库为最终执行目标——均不在本仓库源码内

核心设计原则：**只做增强不做改变**——通过 `MybatisConfiguration` 等 12 个 MyBatis 类的子类进行功能扩展，而非 fork MyBatis。

## 6. 核心时序图说明

![CRUD 调用时序](system-sequence.html)

以 `BaseMapper.selectList(wrapper)` 为例的完整调用链：

1. **业务代码**调用 `mapper.selectList(wrapper)`，进入 JDK 动态代理 `MybatisMapperProxy`
2. **MapperMethod** 根据方法签名分发到 `execute()` → `SqlSession.selectList()`
3. **Executor** 执行 query 前经过 `MybatisPlusInterceptor` 拦截器链
4. **拦截器链**依次执行分页（count + limit 改写）、多租户（追加 tenant_id）、乐观锁（version 校验）、动态表名等改写
5. 改写后的 SQL 通过 JDBC 执行，`ResultSet` 经 MyBatis 结果映射返回实体列表
6. 返回路径沿代理链原路返回

关键设计：拦截器链采用 `InnerInterceptor` SPI，每个拦截器只关注自身逻辑，通过 `Invocation` 包装传递，支持用户自定义插入。

## 7. 系统数据流图说明

![SQL 生成与执行数据流](system-dataflow.html)

SQL 从实体定义到最终执行的数据流分为 5 个阶段：

1. **实体与注解**：用户实体类标注 `@TableName`/`@TableId`/`@TableField` 等注解
2. **元数据解析**：`TableInfoHelper` 在 Mapper 首次加载时反射解析注解，构建 `TableInfo`（表名、主键、字段列表、字段策略）
3. **条件与 SQL 构造**：`Wrapper` 条件段经 `MergeSegments` 拼接为 SQL 片段；`SqlInjector` 将 20 个通用方法注入为 `MappedStatement`
4. **拦截与改写**：`MybatisPlusInterceptor` 链对 SQL 进行分页/多租户/乐观锁等改写（可选 jsqlparser AST 解析）
5. **执行与映射**：最终 SQL 经 MyBatis Executor → JDBC 执行，`ResultSet` 映射为实体对象

## 8. 启动装配流程说明

![Boot 启动装配流程](system-workflow.html)

Spring Boot 应用启动时的自动装配流程（4 条泳道）：

1. **Spring Boot**：应用启动 → 加载 `AutoConfiguration.imports`
2. **自动配置**：`MybatisPlusAutoConfiguration` 条件装配（类路径有 SqlSessionFactory + 容器有单一 DataSource）→ 绑定 `MybatisPlusProperties`
3. **工厂构建**：创建 `MybatisSqlSessionFactoryBean` → 初始化 `GlobalConfig` → 应用 `ConfigurationCustomizer`/`SqlSessionFactoryBeanCustomizer`
4. **Mapper 装配**：Mapper 扫描注册 → `SqlInjector` 注入通用方法 → `InnerInterceptor` 链装配 → 就绪可执行 CRUD

## 9. 语言专项适配口径（JVM）

- **并发模型**：库型项目无自建线程池；`IdWorker` 雪花算法使用 `SystemClock` 定时线程更新时间戳；`JsqlParserThreadPool` 为 jsqlparser 解析提供线程池；`MybatisBatch` 批量执行复用 MyBatis `BatchExecutor`
- **库型无 main 入口**：所有模块为 jar 库，由用户应用启动触发；代码生成器 `AutoGenerator` 由用户代码调用执行
- **模块边界与依赖方向**：annotation → core → extension → spring 单向 api 依赖；jsqlparser 为可选独立链路；boot-starter 为运行时装配层
- **配置面**：`GlobalConfig`（核心全局配置，含 dbConfig/identifierGenerator/metaObjectHandler 等）+ `MybatisPlusProperties`（Boot `mybatis-plus.*` 属性绑定）
- **构建与代码生成**：Gradle 多模块构建，lombok 注解处理器简化实体代码；generator 模块为用户侧代码生成（非构建期生成）
- **事件驱动与状态机**：无服务端事件循环；`IdWorker` 位段构造、DDL 脚本版本记录、Mapper 代理生命周期为主要状态机场景

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 系统架构图 | `system-architecture.html` | architecture | showcase |
| CRUD 调用时序 | `system-sequence.html` | sequence | showcase |
| SQL 生成数据流 | `system-dataflow.html` | dataflow | showcase |
| Boot 启动装配流程 | `system-workflow.html` | workflow | showcase |

系统级 4 张图均为 showcase 档，HTML 均 >780KB（自包含交互式）。JSON IR 源文件位于 `json/` 目录。

域级与叶子级图见各域目录与 `README.md` 索引。
