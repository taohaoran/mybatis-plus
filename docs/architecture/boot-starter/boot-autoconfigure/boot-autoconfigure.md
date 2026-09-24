# Spring Boot 自动配置（boot-autoconfigure）

> 本文是 `boot-starter` 域下的叶子子系统文档。域级总览见 `../boot-starter.md`。
> 本文展开 Spring Boot 起步器如何自动装配 MyBatis-Plus。
>
> 源码基准：`spring-boot-starter` 分支 3.0，commit `bf67d907`（autoconfigure 模块 13 文件 + 三套 starter 各自的 MybatisPlusAutoConfiguration）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 核心自动配置 | 条件装配 `SqlSessionFactory`/`SqlSessionTemplate`/`MapperFactoryBean`，创建 `MybatisSqlSessionFactoryBean` | `mybatis-plus-boot-starter/.../MybatisPlusAutoConfiguration.java:100` |
| 配置属性绑定 | `mybatis-plus.*` 属性绑定（configuration/globalConfig/typeAliasesPackage/mapperLocations 等） | `autoconfigure/MybatisPlusProperties.java:56`（`@ConfigurationProperties(prefix=Constants.MYBATIS_PLUS)`） |
| 配置定制器 SPI | `ConfigurationCustomizer`/`SqlSessionFactoryBeanCustomizer`/`MybatisPlusPropertiesCustomizer` 扩展点 | `autoconfigure/ConfigurationCustomizer.java`、`SqlSessionFactoryBeanCustomizer.java`、`MybatisPlusPropertiesCustomizer.java` |
| 拦截器自动装配 | 容器存在 `InnerInterceptor` 时自动组装 `MybatisPlusInterceptor` | `autoconfigure/MybatisPlusInnerInterceptorAutoConfiguration.java:32` |
| 主键生成器装配 | `IdentifierGenerator` 自动配置 | `autoconfigure/IdentifierGeneratorAutoConfiguration.java` |
| DDL 自动执行 | `DdlApplicationRunner` 在启动后跑 DDL 脚本 | `autoconfigure/DdlAutoConfiguration.java`、`DdlApplicationRunner.java` |
| 语言驱动装配 | 自定义 LanguageDriver 自动配置 | `autoconfigure/MybatisPlusLanguageDriverAutoConfiguration.java` |
| 安全加解密 | `SafetyEncryptProcessor` 配置值加解密 | `autoconfigure/SafetyEncryptProcessor.java` |
| Spring Boot VFS | 适配 Spring Boot 的 VFS 实现 | `autoconfigure/SpringBootVFS.java` |
| 数据库初始化依赖 | 确保 MyBatis 在 DataSource 初始化后处理 | `autoconfigure/MybatisDependsOnDatabaseInitializationDetector.java` |
| 三套 starter | boot2/boot3/boot4 各自打包一份近同构 `MybatisPlusAutoConfiguration` | `mybatis-plus-boot-starter`、`mybatis-plus-spring-boot3-starter`、`mybatis-plus-spring-boot4-starter` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MybatisPlusAutoConfiguration` | `mybatis-plus-boot-starter/.../MybatisPlusAutoConfiguration.java:100` | `@AutoConfiguration`，`@ConditionalOnClass`、`@ConditionalOnSingleCandidate(DataSource)`、`@EnableConfigurationProperties`；`sqlSessionFactory(DataSource)`（行 164）建工厂 |
| `MybatisPlusProperties` | `autoconfigure/MybatisPlusProperties.java:56` | `mybatis-plus.*` 属性载体 |
| `MybatisPlusInnerInterceptorAutoConfiguration` | `autoconfigure/MybatisPlusInnerInterceptorAutoConfiguration.java:32` | `@ConditionalOnBean(InnerInterceptor.class)` 时 `defaultMybatisPlusInterceptor(...)`（行 37） |
| `ConfigurationCustomizer` 等 | `autoconfigure/*.java` | 用户定制 MybatisConfiguration/工厂 Bean/属性的函数式 SPI |

## 3. 关键调用链

1. **Boot 启动装配链**：Spring Boot 加载 `META-INF/spring/...AutoConfiguration.imports` → `MybatisPlusAutoConfiguration`（行 96–100：`@ConditionalOnClass({SqlSessionFactory,SqlSessionFactoryBean})`、`@ConditionalOnSingleCandidate(DataSource)`、`@AutoConfigureAfter({DataSourceAutoConfiguration, MybatisPlusLanguageDriverAutoConfiguration})`）→ 容器就绪后 `sqlSessionFactory(DataSource)`（行 164）：`new MybatisSqlSessionFactoryBean()`（行 165）→ `setVfs(SpringBootVFS.class)`（行 167）→ `applyConfiguration`（行 247，把 `properties.getConfiguration()` 灌入并跑 `ConfigurationCustomizer`，行 257）→ `applySqlSessionFactoryBeanCustomizers`（行 264–266）→ 从容器取 `TransactionFactory/MetaObjectHandler/AnnotationHandler/ISqlInjector/IdentifierGenerator` 等挂到 `globalConfig`（行 197–213）。
2. **拦截器装配链**：`MybatisPlusInnerInterceptorAutoConfiguration`（行 32–34：`@ConditionalOnBean(InnerInterceptor.class)` 且 `@ConditionalOnMissingBean(MybatisPlusInterceptor.class)`）→ `defaultMybatisPlusInterceptor(List<InnerInterceptor>)`（行 37–38）把所有 InnerInterceptor 加进主拦截器。
3. **版本差异链**：boot2（392 行）/boot3（400 行）/boot4（400 行）三份 `MybatisPlusAutoConfiguration` 包名一致、条件注解一致（均 `@ConditionalOnClass`、`@AutoConfigureAfter`），差异在各自 starter 模块对齐的 Spring Boot / jakarta 命名空间版本；共用 autoconfigure 模块提供 Properties 与各定制器。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `mybatis-plus.*` | 配置属性前缀（`Constants.MYBATIS_PLUS`） | `MybatisPlusProperties.java:56` |
| 装配条件 | 类路径存在 `SqlSessionFactory`/`SqlSessionFactoryBean` 且容器有单一 `DataSource` | `MybatisPlusAutoConfiguration.java:96–97` |
| 装载顺序 | 在 `DataSourceAutoConfiguration` 与语言驱动配置之后 | 行 99 |
| `configuration` 定制 | 存在 `ConfigurationCustomizer` Bean 时依次应用 | 行 257 |
| 日志前缀 | `mybatis-plus.configuration.log-prefix` 可配 | `MybatisPlusProperties.java:286` |

## 5. 错误与重试语义

- 自动配置类实现 `InitializingBean`（行 100），`afterPropertiesSet()`（行 147）做配置校验；校验失败抛 Bean 初始化异常，Boot 启动失败。
- 工厂创建异常（`sqlSessionFactory` 声明 `throws Exception`，行 164）由 Spring 包装为 `BeanCreationException`，启动即失败。
- 无运行期重试：装配期问题快速失败；运行期 SQL 错误由 MyBatis/事务处理。

## 6. 并发细节

- 自动配置在容器启动单线程刷新阶段完成，无并发装配。
- `SqlSessionFactory`/`MybatisPlusInterceptor` 为容器单例，运行期线程安全（MyBatis 层保证）。
- 注入用 `ObjectProvider<List<...>>`（行 130–132）惰性可选获取定制器列表，避免缺 Bean 时启动失败。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `mybatis-plus-spring-boot-autoconfigure` 13 文件与三套 starter 的 `MybatisPlusAutoConfiguration`。

**Out-of-Scope（不在本仓库源码内）**
- Spring Boot `@AutoConfiguration` 机制、`DataSourceAutoConfiguration`、条件注解实现均为 Spring Boot 框架，不在本仓库源码内。
- 实际 `MybatisSqlSessionFactoryBean` 实现见 `../spring-integration/spring-integration.md`；`MybatisPlusInterceptor`/`InnerInterceptor` 定义在 extension 模块（相邻分片）。
- GraalVM AOT 原生镜像支持见 `../boot-native-test/boot-native-test.md`。

## 8. 与相邻子系统交互

- **上游**：用户引入 starter 依赖后由 Spring Boot 触发自动配置。
- **本叶子 → spring 域**：创建 `com.baomidou.mybatisplus.spring.MybatisSqlSessionFactoryBean`。
- **本叶子 → extension/core**：装配 `IdentifierGenerator`/`ISqlInjector`/`MetaObjectHandler`/`InnerInterceptor` 等增强组件。
- **下游**：业务 Mapper/Service 由装配出的 `SqlSessionFactory` 代理。

## 9. 语言专项适配口径（JVM）

- **Spring Boot 自动配置范式**：`@AutoConfiguration` + 条件注解 + `ObjectProvider` 可选注入 + Customizer SPI，是 Spring Boot Starter 标准写法。
- **多产物版本矩阵**：boot2/boot3/boot4 三个 starter 模块各自打同构自动配置类，对齐不同 Spring Boot 大版本（jakarta 命名空间），共用 autoconfigure 基础模块。
- **库型装配**：无 main，由用户 Boot 应用启动类触发。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 自动配置架构图 | `boot-autoconfigure-architecture.html` | architecture | showcase |
| Boot 启动装配流程 | `boot-autoconfigure-workflow.html` | workflow | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子第二图选用 workflow：Boot 启动自动配置是"条件判定→建工厂→灌配置→挂增强组件"带分支的分步流程。
