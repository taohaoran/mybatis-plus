# 原生镜像 AOT 与测试自动配置（boot-native-test）

> 本文是 `boot-starter` 域下的叶子子系统文档。域级总览见 `../boot-starter.md`。
> 本文展开 GraalVM 原生镜像 AOT 提示登记与测试切片自动配置两块能力。
>
> 源码基准：`spring-boot-starter` 分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| AOT 运行时提示登记 | 实现 `RuntimeHintsRegistrar`，为反射/动态代理/资源登记 GraalVM 原生提示 | `mybatis-plus-spring-boot-native-image/.../aot/MyBaitsRuntimeHintsRegistrar.java:29` |
| 原生镜像配置类 | `@Configuration` + `@ImportRuntimeHints`，条件在 classpath 有 `MapperFactoryBean` 时生效 | `aot/MyBatisPlusNativeImageConfiguration.java:72` |
| AOT 工具 | 收集类/方法/字段并登记反射提示的工具方法 | `aot/AotUtils.java`、`CollectUtils.java`、`FeatureUtils.java` |
| Bean 工厂 AOT 后处理 | 对 `MapperFactoryBean` 做 AOT 排除/贡献（`BeanFactoryInitializationAotProcessor`） | `MyBatisPlusNativeImageConfiguration.java:103` |
| 测试切片注解 | `@MybatisPlusTest` 切片测试自动装配 Mapper | `mybatis-plus-spring-boot-test-autoconfigure/.../test/autoconfigure/MybatisPlusTest.java` |
| 导入自动配置 | `@AutoConfigureMybatisPlus` 导入测试专用配置 | `test/autoconfigure/AutoConfigureMybatisPlus.java` |
| 测试引导器 | 自定义 `TestContextBootstrapper` | `test/autoconfigure/MybatisPlusTestContextBootstrapper.java` |
| 类型排除过滤器 | 测试切片类型过滤 | `test/autoconfigure/MybatisPlusTypeExcludeFilter.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MyBaitsRuntimeHintsRegistrar` | `aot/MyBaitsRuntimeHintsRegistrar.java:29` | 实现 `RuntimeHintsRegistrar`；`registerHints()`（行 32）建 `AotUtils` 登记提示 |
| `MyBatisPlusNativeImageConfiguration` | `aot/MyBatisPlusNativeImageConfiguration.java:72` | `@ConditionalOnClass(MapperFactoryBean.class)`；内含 `MyBatisRuntimeHintsRegistrar`（行 84）与 `MyBatisBeanFactoryInitializationAotProcessor`（行 103） |
| `AotUtils` | `aot/AotUtils.java` | 反射/资源提示登记工具 |
| `MybatisPlusTest` | `test/autoconfigure/MybatisPlusTest.java` | 切片测试组合注解 |
| `MybatisPlusTestContextBootstrapper` | `test/autoconfigure/MybatisPlusTestContextBootstrapper.java` | Spring TestContext 引导 |

## 3. 关键调用链

1. **AOT 提示登记链**：原生构建期 Spring 调 `MyBaitsRuntimeHintsRegistrar.registerHints(hints, classLoader)`（`MyBaitsRuntimeHintsRegistrar.java:32–33`）→ `new AotUtils(hints, classLoader)` → 收集实体/Mapper/类型处理器等需反射的类与方法，登记 `reflectionTypeQuery`/资源/动态代理提示，使 GraalVM 在原生镜像中保留这些元素。
2. **原生配置装配链**：`MyBatisPlusNativeImageConfiguration`（行 70–72：`@ConditionalOnClass(MapperFactoryBean.class)`、`@Configuration(proxyBeanMethods=false)`、`@ImportRuntimeHints(MyBatisRuntimeHintsRegistrar.class)`）→ 内部 `MyBatisBeanFactoryInitializationAotProcessor.processAheadOfTime(beanFactory)`（行 118–119）按 `getBeanNamesForType(MapperFactoryBean.class)` 遍历 Mapper bean，对其接口方法登记提示（行 143 起，排除 `Object` 声明的方法）。
3. **测试切片链**：测试类标 `@MybatisPlusTest` → 经 `AutoConfigureMybatisPlus` 导入测试自动配置 → `MybatisPlusTestContextBootstrapper` 构建测试上下文，`MybatisPlusTypeExcludeFilter` 限定仅装配 Mapper 相关 bean。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 原生配置生效条件 | classpath 存在 `MapperFactoryBean` | `MyBatisPlusNativeImageConfiguration.java:70` |
| 提示登记触发 | 原生镜像构建期（AOT），非运行期 | `MyBaitsRuntimeHintsRegistrar.java:32` |
| 测试切片 | `@MybatisPlusTest` 标注的测试类 | `MybatisPlusTest.java` |

## 5. 错误与重试语义

- AOT 登记为构建期一次性处理，提示缺失会导致原生镜像运行期反射失败（`ClassNotFoundException`/`NoSuchMethod`），不在本模块内重试。
- 测试切片装配失败抛 Spring TestContext 异常，测试即失败。
- 无运行期退避/重试：本叶子是构建期与测试期工具，错误在构建/测试阶段暴露。

## 6. 并发细节

- AOT 登记在原生构建单线程环境执行，无并发。
- 测试上下文由 Spring TestContext 框架按测试方法管理，无线程池。
- 无显式锁/异步回调。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `mybatis-plus-spring-boot-native-image`（aot 6 文件）与 `mybatis-plus-spring-boot-test-autoconfigure`（4 文件）。

**Out-of-Scope（不在本仓库源码内）**
- GraalVM `RuntimeHints`/`RuntimeHintsRegistrar`、Spring AOT 引擎、JUnit/Spring TestContext 框架均为第三方，不在本仓库源码内。
- 被登记提示的 Mapper/实体来自用户应用与 core/extension 模块。
- 运行期自动配置见 `../boot-autoconfigure/boot-autoconfigure.md`。

## 8. 与相邻子系统交互

- **上游**：GraalVM 原生构建流程 / Spring Boot 测试框架。
- **本叶子 → core/extension**：为 `MapperFactoryBean`、实体、类型处理器登记反射提示。
- **相邻**：与 `boot-autoconfigure` 互补——后者负责运行期 bean 装配，本叶子负责原生构建期提示与测试切片。

## 9. 语言专项适配口径（JVM）

- **GraalVM AOT**：通过 `RuntimeHintsRegistrar` 显式登记反射/资源/代理提示，弥补原生镜像无 classpath 扫描的限制，是 JVM 库原生适配的标准做法。
- **Spring Boot 测试切片**：`@MybatisPlusTest` + TypeExcludeFilter 是 Spring Boot Test 切片范式。
- **构建期 vs 运行期分离**：本叶子代码仅在原生构建或测试时生效，常规运行 jar 不触发。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| AOT 与测试配置架构图 | `boot-native-test-architecture.html` | architecture | showcase |
| AOT 提示登记数据流 | `boot-native-test-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子第二图选用 dataflow：AOT 是"扫描 Mapper/实体 → 收集类方法 → 登记 ReflectionHints → 写入原生镜像元数据"的数据加工管道；测试切片与 AOT 合并为一张架构图。
