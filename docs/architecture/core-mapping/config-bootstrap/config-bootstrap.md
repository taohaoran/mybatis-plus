# 配置引导（config-bootstrap）

> 本文是 `core-mapping` 域下的叶子子系统文档。域级总览见 `../core-mapping.md`。
> 本文展开 `core/config/` + 配置装配类：GlobalConfig 全局配置、SqlSessionFactory 构建入口、XML 配置解析覆盖、注入器解析器。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 全局配置 | `GlobalConfig`：持有 sqlInjector/metaObjectHandler/identifierGenerator/dbConfig | `config/GlobalConfig.java:48` |
| 注入器默认值 | `sqlInjector = new DefaultSqlInjector()` | `GlobalConfig.java:64` |
| 数据库配置 | `GlobalConfig.DbConfig`：表前缀/下划线/逻辑删除等 | `GlobalConfig.java:104` |
| 会话工厂构建 | `MybatisSqlSessionFactoryBuilder`：替代 MyBatis SqlSessionFactoryBuilder | `MybatisSqlSessionFactoryBuilder.java:118` |
| 配置中心 | `MybatisConfiguration`：覆盖 MyBatis Configuration，装配 MP 组件 | `MybatisConfiguration.java:496` |
| XML 配置解析 | `MybatisXMLConfigBuilder`：解析 mybatis-config.xml 时挂入 GlobalConfig | `MybatisXMLConfigBuilder.java:443` |
| XML Mapper/Script 解析 | `MybatisXMLMapperBuilder`/`MybatisXMLScriptBuilder`/`MybatisXMLLanguageDriver` | `MybatisXMLMapperBuilder.java:405` |
| 注入器解析器 | `InjectorResolver`：resolve() 调 parserInjector 触发注入 | `InjectorResolver.java:26,36` |
| 版本号 | `MybatisPlusVersion`：MP 版本信息 | `MybatisPlusVersion.java:68` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `GlobalConfig` | `GlobalConfig.java:48` | 全局可插拔组件容器 |
| `MybatisSqlSessionFactoryBuilder` | `MybatisSqlSessionFactoryBuilder.java:118` | 构建入口 |
| `MybatisConfiguration` | `MybatisConfiguration.java:496` | MP 配置中心 |
| `InjectorResolver` | `InjectorResolver.java:26` | 触发注入的 MethodResolver |
| `MybatisPlusVersion` | `MybatisPlusVersion.java:68` | 版本号 |

## 3. 关键调用链

**启动装配流程**：

1. 用户调 `new MybatisSqlSessionFactoryBuilder().build(...)`（`MybatisSqlSessionFactoryBuilder.java:118`）。
2. 构建过程中 `MybatisXMLConfigBuilder`（`MybatisXMLConfigBuilder.java:443`）解析配置，把 `GlobalConfig` 挂到 `MybatisConfiguration`（`GlobalConfigUtils` 持 GLOBAL_CONFIG map）。
3. GlobalConfig 默认 `sqlInjector = new DefaultSqlInjector()`（`GlobalConfig.java:64`），用户可覆盖。
4. Mapper 解析时 `InjectorResolver.resolve()`（`InjectorResolver.java:36`）→ `annotationBuilder.parserInjector()` → 触发 sql-injector 叶子注入内置 CRUD。
5. 运行期各叶子经 `GlobalConfigUtils.getMetaObjectHandler/getSqlInjector/getIdentifierGenerator` 取配置组件。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| sqlInjector | DefaultSqlInjector，可替换 | `GlobalConfig.java:64` |
| metaObjectHandler | null（不填充） | `GlobalConfig.java:83` |
| identifierGenerator | null（用默认雪花） | `GlobalConfig.java:97` |
| DbConfig | 表前缀/是否驼峰转下划线/逻辑删除字段 | `GlobalConfig.java:104` |

## 5. 错误与重试语义

- 启动期配置错误抛异常终止启动；无运行期重试。
- GlobalConfig 缺省组件时运行期按默认或 null-safe 处理（如填充 ifPresent 跳过）。

## 6. 并发细节

- `GlobalConfigUtils.GLOBAL_CONFIG`（toolkit）为 ConcurrentHashMap，按 Configuration 缓存。
- GlobalConfig 启动期装配、运行期只读。
- 无线程池。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/config/` + 配置装配类。

**Out-of-Scope（不在本仓库源码内）**
- MyBatis 原生 XMLConfigBuilder/Configuration/MethodResolver（第三方，仅继承覆盖）。
- Spring Boot starter 自动装配（extension/starter，不在 core）。

## 8. 与相邻子系统交互

- 上游：用户/Spring Boot starter 触发 build。
- 本叶子 → 下游：GlobalConfig 向 sql-injector、type-handlers、id-generator、mapper-runtime 提供可插拔组件实例。
- 数据流：**build → 解析 XML → 装配 GlobalConfig → 运行期各叶子取用**。

## 9. 语言专项适配口径（Java/JVM）

- **建造者模式**：SqlSessionFactoryBuilder。
- **可插拔组件容器**：GlobalConfig 持有 SPI 接口引用，默认实现兜底。
- **继承覆盖 MyBatis**：XMLConfigBuilder/Configuration。
- **库型无 main**（Spring Boot starter 才是进程入口，不在 core）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 配置引导架构图 | `config-bootstrap-architecture.html` | architecture | showcase |
| 启动装配数据流 | `config-bootstrap-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 dataflow 图：build→解析XML→装配GlobalConfig→注入CRUD是典型"触发→加工→就绪"的启动数据流。
未生成 sequence/lifecycle/workflow：装配是启动期一次性流程（已 dataflow 化），无多参与方消息时序主链、无状态机，按资源节省原则省略。
