# 启动器（boot-starter）域总览

> 本域包含 2 个叶子子系统；各叶子详情见对应文档。
> 源码基准：`spring-boot-starter` 分支 3.0，commit `bf67d907`（26 文件，约 4142 行）。

## 1. 域职责

启动器域让业务应用"引入依赖即自动装配 MyBatis-Plus"。核心是 `MybatisPlusAutoConfiguration`——基于 Spring Boot 条件注解，在容器就绪时创建 `MybatisSqlSessionFactoryBean`、绑定 `mybatis-plus.*` 属性、装配 `MybatisPlusInterceptor` 与各类增强组件。域内同时包含 GraalVM 原生镜像 AOT 提示登记模块与测试切片自动配置模块。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| 自动配置 | [boot-autoconfigure.md](boot-autoconfigure/boot-autoconfigure.md) | [架构图](boot-autoconfigure/boot-autoconfigure-architecture.html) | [流程](boot-autoconfigure/boot-autoconfigure-workflow.html) | 条件装配 SqlSessionFactory 与拦截器 |
| AOT 与测试 | [boot-native-test.md](boot-native-test/boot-native-test.md) | [架构图](boot-native-test/boot-native-test-architecture.html) | [数据流](boot-native-test/boot-native-test-dataflow.html) | 原生镜像反射提示登记 + 测试切片 |

## 3. 域级机制细节

- **三套 starter 版本矩阵**：`mybatis-plus-boot-starter`（Boot 2.x）、`mybatis-plus-spring-boot3-starter`、`mybatis-plus-spring-boot4-starter` 各自打包一份近同构的 `MybatisPlusAutoConfiguration`（392/400/400 行），对齐不同 Spring Boot 大版本与 jakarta 命名空间；共用 `mybatis-plus-spring-boot-autoconfigure` 基础模块提供 `MybatisPlusProperties` 与定制器 SPI。
- **条件装配**：`@ConditionalOnClass({SqlSessionFactory, SqlSessionFactoryBean})` + `@ConditionalOnSingleCandidate(DataSource)` + `@AutoConfigureAfter(DataSourceAutoConfiguration, ...)`。
- **可选定制注入**：用 `ObjectProvider<List<ConfigurationCustomizer>>` 惰性获取定制器，缺 Bean 不失败。
- **AOT 与运行期分离**：native-image 模块仅在原生构建期经 `RuntimeHintsRegistrar` 登记反射提示；test-autoconfigure 仅在 `@MybatisPlusTest` 切片测试时生效。

## 4. 域级图

![启动器域架构图](boot-starter-architecture.html)

![Boot 启动装配时序](boot-starter-sequence.html)

![启动器装配数据流](boot-starter-dataflow.html)

三张域级图分别表达：依赖→自动配置→工厂的静态拓扑（架构）、启动时序（时序）、属性绑定到工厂构建的数据管道（数据流），与叶子图互补。
