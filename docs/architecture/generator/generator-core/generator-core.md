# 生成主流程编排（generator-core）

> 本文是 `generator` 域下的叶子子系统文档。域级总览见 `../generator.md`。
> 本文只展开 `AutoGenerator`/`FastAutoGenerator` 如何编排配置→查询→引擎全链路，以及表注解/Mapper 方法等扩展 SPI；配置见 `../generator-config/generator-config.md`，查询见 `../generator-query/generator-query.md`，引擎见 `../generator-engine/generator-engine.md`。
>
> 源码基准：`mybatis-plus-generator` 分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 生成器编排器 | 持有六类配置，`execute()` 组装 ConfigBuilder 并驱动引擎生命周期 | `AutoGenerator.java:37`（`execute()` 行 173） |
| 流式快速入口 | 五个 Builder 的流式 Consumer 填充，最终转成 AutoGenerator | `FastAutoGenerator.java:34`（`execute()` 行 226） |
| 命令行扫描 | 交互式 `scannerNext` 从控制台读配置 | `FastAutoGenerator.java:101` |
| 表注解处理 SPI | 为生成实体追加类级注解（如 `@TableName`） | `ITableAnnotationHandler.java:30`、`DefaultTableAnnotationHandler.java:35` |
| 字段注解处理 SPI | 为生成字段追加注解（`DefaultTableFieldAnnotationHandler`） | `DefaultTableFieldAnnotationHandler.java` |
| 表字段元信息定制 SPI | 在默认转换链上定制特殊列类型/字段 | `ITableFieldMetaInfoCustomizer.java:37`（`andThen` 行 55） |
| Mapper 方法生成 SPI | 自定义 Mapper 接口方法（Java/Kotlin） | `IGenerateMapperMethodHandler.java:30`、`index/AbstractMapperMethodHandler.java:25` |
| 默认 Mapper 方法处理器 | 默认生成 Lambda/普通 Mapper 方法实现 | `index/DefaultGenerateMapperMethodHandler.java`、`DefaultGenerateMapperLambdaMethodHandler.java` |
| 模板标记接口 | Entity/Mapper/Service/Controller 子策略实现的统一标记 | `ITemplate.java` |
| 自动填充接口 | 填充字段定义接口（实现见 engine 的 fill/） | `IFill.java:27` |
| 注解属性模型 | 生成注解的属性/类注解承载对象 | `model/AnnotationAttributes.java`、`ClassAnnotationAttributes.java`、`MapperMethod.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AutoGenerator` | `AutoGenerator.java:37` | 编排根；链式 `injection/strategy/packageInfo/global` 收配置，`execute()` 驱动 |
| `FastAutoGenerator` | `FastAutoGenerator.java:34` | fluent 门面；内部持有五个 `Builder`，`execute()` 构造并委托 `AutoGenerator` |
| `ITableAnnotationHandler` | `ITableAnnotationHandler.java:30` | 类级注解扩展点；`DefaultTableAnnotationHandler.handle()`（行 38）产出 `@TableName` |
| `ITableFieldMetaInfoCustomizer` | `ITableFieldMetaInfoCustomizer.java:37` | 字段级定制链；`andThen()`（行 55）支持组合 |
| `IGenerateMapperMethodHandler` | `IGenerateMapperMethodHandler.java:30` | Mapper 方法生成 SPI |
| `AbstractMapperMethodHandler` | `index/AbstractMapperMethodHandler.java:25` | 方法建造模板；`buildMethod()`（行 41）、`buildKotlinMethod()`（行 68） |
| `IFill` | `IFill.java:27` | 自动填充标记接口 |
| `ITemplate` | `ITemplate.java` | 模板路径供给标记接口 |

## 3. 关键调用链

1. **流式入口链**（`FastAutoGenerator.execute()`，行 226–235）：`new AutoGenerator(this.dataSourceConfigBuilder.build())` → 链式 `.global(globalConfigBuilder.build()).packageInfo(...).strategy(...).injection(...)` → 调 `AutoGenerator.execute(templateEngine)`。
2. **编排执行主链**（`AutoGenerator.execute(AbstractTemplateEngine)`，行 173–187）：若 `config==null` 则 `new ConfigBuilder(packageInfo, dataSource, strategy, template, globalConfig, injection)`（行 177，触发配置叶子的默认补全与路径计算）→ 若 `templateEngine==null` 默认 `new VelocityTemplateEngine()`（行 181）→ `templateEngine.setConfigBuilder(config)`（行 183）→ `templateEngine.init(config).batchOutput().open()`（行 185，驱动 engine 叶子的渲染输出）。
3. **表信息开放点**：`getAllTableInfoList(config)`（行 196–198）直接返回 `config.getTableInfoList()`，预留子类重写。
4. **注解处理链**：`DefaultTableAnnotationHandler.handle(tableInfo, entity)`（行 38）在策略 renderData 阶段据表名产出 `ClassAnnotationAttributes(TableName.class, displayName)`（行 71），供模板渲染类级注解。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 必传项 | 仅 `DataSourceConfig` 必传（构造器行 82–85），其余可选 | `AutoGenerator.java:82` |
| 默认模板引擎 | 未传时 Velocity | `AutoGenerator.java:181` |
| `template(TemplateConfig)` | 3.5.6 起 `@Deprecated`，迁移到 `StrategyConfig` 子策略 | `AutoGenerator.java:65/131` |
| `scannerNext(message)` | 交互式从 `System.in` 读一行（BiConsumer 重载时启用） | `FastAutoGenerator.java:101` |
| 五个 Builder 默认实例 | global/package/strategy/injection 在 `FastAutoGenerator` 构造时 `new`（行 75–78） | `FastAutoGenerator.java:75` |

## 5. 错误与重试语义

- 本叶子是薄编排，自身不捕获异常；`ConfigBuilder` 反射失败、引擎渲染失败均沿调用链上抛为 `RuntimeException`（见 config/engine 叶子）。
- 无重试/退避：一次性本地代码生成，失败即终止 JVM。
- 废弃方法（`template()`、`TemplateConfig`）保留兼容，标注 `@Deprecated` 指向新策略位置，不删。

## 6. 并发细节

- 单线程本地执行；`AutoGenerator`/`FastAutoGenerator` 实例为一次性使用，无线程池/锁。
- `execute()` 内 `ConfigBuilder` 构造与引擎驱动顺序执行，无并发切换。
- `FastAutoGenerator.scannerNext` 为同步控制台 IO，阻塞当前线程。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 顶层 `AutoGenerator`/`FastAutoGenerator`/`ITemplate`/`IFill`/各注解与方法 SPI、`index/` 方法处理器、`model/` 注解属性模型。

**Out-of-Scope（不在本仓库源码内）**
- 配置模型细节（config/）、JDBC 元数据采集（query/jdbc/）、模板渲染（engine/）分别归相邻叶子。
- 用户 `main` 方法与构建产物 jar 不在本仓库；模板资源文件随包发布。

## 8. 与相邻子系统交互

- **上游**：用户代码（测试/业务 main）调用 `FastAutoGenerator.create(url,user,pwd)...execute()`。
- **本叶子 → generator-config**：组装六类配置构造 `ConfigBuilder`。
- **本叶子 → generator-engine**：把 `ConfigBuilder` 交给模板引擎并驱动 `init→batchOutput→open`。
- **本叶子 ↔ generator-query**：经 `ConfigBuilder.getTableInfoList()` 间接触发元数据采集。

## 9. 语言专项适配口径（JVM）

- **门面 + 编排**：`FastAutoGenerator` 是 fluent builder 门面，`AutoGenerator` 是编排器，符合 Java 库"易用入口 + 可扩展内核"分层。
- **SPI 扩展点**：注解处理、字段定制、Mapper 方法生成均为接口 + 默认实现，用户可替换；`ITableFieldMetaInfoCustomizer.andThen()` 是函数式组合。
- **库型无 main**：本模块不启动进程，由用户 main 调用；Gradle 多模块中 generator implementation 依赖 spring（facts.md §3）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 生成主流程架构图 | `generator-core-architecture.html` | architecture | showcase |
| AutoGenerator 编排时序 | `generator-core-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子第二图选用 sequence：`execute()` 是"用户→FastAutoGenerator→AutoGenerator→ConfigBuilder→引擎→文件系统"的多方按时间顺序调用链，时序图最能表达其编排关系。
