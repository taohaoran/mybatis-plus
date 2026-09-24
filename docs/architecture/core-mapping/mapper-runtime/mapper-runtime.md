# Mapper 运行时（mapper-runtime）

> 本文是 `core-mapping` 域下的叶子子系统文档。域级总览见 `../core-mapping.md`。
> 本文展开 `core/override/` + 顶层 Mybatis* 运行时类：覆盖 MyBatis 的 Mapper 代理/参数处理/XML 解析/配置装配，接入 MP 的注入器与填充 SPI。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Mapper 代理 | `MybatisMapperProxy`：JDK 动态代理 invoke 分发到 MybatisMapperMethod | `override/MybatisMapperProxy.java:88` |
| 方法缓存 | methodCache：Method→MapperMethodInvoker 缓存，避免重复构造 | `MybatisMapperProxy.java:53,99` |
| Mapper 方法 | `MybatisMapperMethod`：替代 MyBatis MapperMethod | `override/MybatisMapperMethod.java:261` |
| 参数处理 | `MybatisParameterHandler`：处理参数时触发 insertFill/updateFill 自动填充 | `MybatisParameterHandler.java:52,77` |
| Mapper 注解构建 | `MybatisMapperAnnotationBuilder`：parse 时除解析注解还调 inspectInject | `MybatisMapperAnnotationBuilder.java:88,126` |
| Mapper 注册 | `MybatisMapperRegistry`：注册 Mapper 接口 | `MybatisMapperRegistry.java:108` |
| 配置覆盖 | `MybatisConfiguration`：覆盖 MyBatis Configuration 关键默认 | `MybatisConfiguration.java:496` |
| 工厂构建 | `MybatisSqlSessionFactoryBuilder`：构建入口 | `MybatisSqlSessionFactoryBuilder.java:118` |
| XML 解析覆盖 | `MybatisXMLConfigBuilder/XMLMapperBuilder/XMLScriptBuilder/XMLLanguageDriver` | `MybatisXMLConfigBuilder.java:443` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `MybatisMapperProxy` | `MybatisMapperProxy.java:88` | Mapper 方法调用入口（动态代理） |
| `MybatisParameterHandler` | `MybatisParameterHandler.java:52` | 参数处理 + 自动填充触发 |
| `MybatisMapperAnnotationBuilder` | `MybatisMapperAnnotationBuilder.java:68` | Mapper 解析 + 注入器钩子 |
| `MybatisConfiguration` | `MybatisConfiguration.java:496` | MP 配置中心 |
| `MybatisSqlSessionFactoryBuilder` | `MybatisSqlSessionFactoryBuilder.java:118` | 构建入口 |

## 3. 关键调用链

**Mapper 方法执行**：

1. 用户调 `userMapper.selectById(1)` → JDK 代理 `MybatisMapperProxy.invoke`（`MybatisMapperProxy.java:88`）→ `cachedInvoker(method)`（`:99`）取/建 `PlainMethodInvoker(new MybatisMapperMethod(...))`（`:103`）。
2. invoker.invoke（`:155`）→ MybatisMapperMethod 执行 → MyBatis 走 SqlSession。
3. SQL 执行前参数处理：`MybatisParameterHandler.processParameter`（`MybatisParameterHandler.java:77`）→ 按 SqlCommandType 是 insert 调 `insertFill`（`:112,161`）→ `GlobalConfigUtils.getMetaObjectHandler(...).ifPresent(h -> h.insertFill(metaObject))`（`:162-164`）；update 走 `updateFill`（`:114,169`）。

**Mapper 解析期注入钩子**：

1. `MybatisMapperAnnotationBuilder.parse`（`MybatisMapperAnnotationBuilder.java:88`）→ loadXmlResource（`:91`）→ parseStatement 注解语句（`:108`）。
2. 末尾 `GlobalConfigUtils.getSqlInjector(configuration).inspectInject(assistant, type)`（`:126`）——接入 sql-injector 叶子。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| MetaObjectHandler | GlobalConfig 注入，无则不填充 | `MybatisParameterHandler.java:162` |
| SqlInjector | GlobalConfig 注入，默认 DefaultSqlInjector | `MybatisMapperAnnotationBuilder.java:126` |
| MybatisConfiguration | 覆盖 MyBatis 默认语言驱动/类型注册器等 | `MybatisConfiguration.java:496` |

## 5. 错误与重试语义

- 代理调用异常透传；填充 handler 缺失时 `ifPresent` 静默跳过。
- 无重试；SQL 执行异常由 MyBatis 抛。

## 6. 并发细节

- `methodCache`（`MybatisMapperProxy.java:53`）为 ConcurrentHashMap 语义，方法 invoker 缓存线程安全。
- 代理对象本身无状态；MybatisParameterHandler 每次执行新建。
- 无线程池；单条 SQL 在业务线程执行。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/override/` + 顶层 Mybatis* 运行时类。

**Out-of-Scope（不在本仓库源码内）**
- MyBatis 原生 `MapperProxy`/`MapperMethod`/`Configuration`/`DefaultParameterHandler`/`XMLConfigBuilder`（第三方，本叶子仅继承覆盖）。
- 注入器实现细节（sql-injector 叶子）。
- 填充 handler 实现（type-handlers 叶子）。

## 8. 与相邻子系统交互

- 上游：业务代码经 Mapper 接口调用；config-bootstrap 装配这些覆盖类。
- 本叶子 → 下游：回调 type-handlers 的 MetaObjectHandler、sql-injector 的 inspectInject；最终委托 MyBatis。
- 数据流：**Mapper 代理 → 参数处理+填充 → MyBatis 执行 SQL**。

## 9. 语言专项适配口径（Java/JVM）

- **JDK 动态代理**：`InvocationHandler.invoke` 分发。
- **继承覆盖 MyBatis 类**：`extends MapperAnnotationBuilder/DefaultParameterHandler/Configuration`，重写钩子接入 MP。
- **Optional 链**：`getMetaObjectHandler().ifPresent(...)`。
- **库型无 main**。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Mapper 运行时架构图 | `mapper-runtime-architecture.html` | architecture | showcase |
| Mapper 调用时序 | `mapper-runtime-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：Mapper 调用→代理分发→参数处理→填充→执行是清晰的多参与方调用时序。
未生成 dataflow/lifecycle/workflow：运行期是单次请求链（已 sequence 化），无数据管道、无状态机、无审批流，按资源节省原则省略。
