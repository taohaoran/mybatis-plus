# 模板引擎与文件输出（generator-engine）

> 本文是 `generator` 域下的叶子子系统文档。域级总览见 `../generator.md`。
> 本文只展开"把 TableInfo 渲染数据经模板引擎输出为代码文件"的职责；配置汇总见 `../generator-config/generator-config.md`，元数据采集见 `../generator-query/generator-query.md`，主流程编排见 `../generator-core/generator-core.md`。
>
> 源码基准：`mybatis-plus-generator` 分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 模板引擎抽象 | 固化 init/batchOutput/open 流程与七类产物输出模板方法，子类实现 writer | `engine/AbstractTemplateEngine.java:45` |
| Velocity 引擎 | 默认引擎；初始化 VelocityEngine，文件/字符串两种资源加载 | `engine/VelocityTemplateEngine.java:38`（`init()` 行 51） |
| FreeMarker 引擎 | FreeMarker 实现 | `engine/FreemarkerTemplateEngine.java` |
| Beetl 引擎 | Beetl 实现 | `engine/BeetlTemplateEngine.java` |
| Enjoy 引擎 | Enjoy（JFinal）实现 | `engine/EnjoyTemplateEngine.java` |
| 逐表批量输出 | 遍历表列表，组装 objectMap，依次输出 entity/mapper/xml/service/controller + 自定义文件 | `AbstractTemplateEngine.batchOutput()` 行 231 |
| 渲染数据组装 | 合并四类子策略 renderData + 全局变量（author/kotlin/swagger/date/table/package） | `AbstractTemplateEngine.getObjectMap()` 行 309 |
| 文件覆盖判定 | 已存在且未开 override 则告警跳过 | `AbstractTemplateEngine.isCreate()` 行 362 |
| 自动开目录 | 全局 open=true 时用系统命令打开输出目录 | `AbstractTemplateEngine.open()` 行 282 |
| 自动填充定义 | `IFill` 两个实现 Column（按列名）/Property（按属性名）+ FieldFill 策略 | `fill/Column.java:28`、`fill/Property.java:28` |
| 文件名转换 SPI | 自定义生成类命名 | `function/ConverterFileName.java:28` |
| 工具类 | 类加载、文件创建、Kotlin 类型映射、打开目录 | `util/ClassUtils.java`、`FileUtils.java`、`KotlinTypeUtils.java`、`RuntimeUtils.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AbstractTemplateEngine` | `engine/AbstractTemplateEngine.java:45` | 模板方法基类；持有 `configBuilder`，定义 `init→batchOutput→open` 生命周期 |
| `VelocityTemplateEngine` | `engine/VelocityTemplateEngine.java:38` | 默认引擎；`writer(objectMap,templatePath,outputFile)`（行 78）merge 到 UTF-8 文件 |
| `writer(Map,String,String)` / `writer(Map,String,File)` | `AbstractTemplateEngine.java:266/277` | 抽象渲染接口；前者渲染为字符串，后者落盘 |
| `ITemplate` | `ITemplate.java`（顶层） | 子策略标记接口（Entity/Mapper/Service/Controller 实现它，提供模板路径） |
| `IFill` | `IFill.java`（顶层） | 自动填充字段定义接口；`Column`/`Property` 为其实现 |
| `ConverterFileName` | `function/ConverterFileName.java:28` | 文件名转换函数式接口 |
| `OutputFile` 七槽位 | `config/OutputFile.java`（被 engine 消费） | entity/mapper/xml/service/serviceImpl/controller/parent |

## 3. 关键调用链

1. **批量输出主链**（`batchOutput()`，`AbstractTemplateEngine.java:231–256`）：取 `config.getTableInfoList()`（行 234，触发 generator-query 采集）→ 逐表 `getObjectMap(config, tableInfo)`（行 236）→ 若有 `injectionConfig` 先调 `t.beforeOutputFile(tableInfo, objectMap)` 加自定义变量、再 `outputCustomFile(t.getCustomFiles(),...)`（行 237–242）→ 依次 `outputEntity`（行 244）、`outputMapper`（行 246，含 .java 与 .xml）、`outputService`（行 248，含接口与 Impl）、`outputController`（行 250）；任何异常包成 `RuntimeException("An exception occurred in the output file: ", e)`（行 253）。
2. **单类落盘链**（以 entity 为例，`outputEntity()` 行 89–98）：取 `getPathInfo(OutputFile.entity)`（行 91）→ 读 `strategyConfig.entity()` 子策略判 `isGenerate()`（行 94）→ 拼 `entityPath/EntityName.java`（Kotlin 时拼 `.kt`，行 95）→ `getOutputFile(...)` 经策略 `OutputFile.createFile` 建文件（行 96、`getOutputFile()` 行 100–102）→ 进 `outputFile()`。
3. **落盘与覆盖判定链**（`outputFile()`，行 180–195）：`isCreate(file, fileOverride)`（行 181、行 362–367：已存在且未开覆盖则告警返回 false）→ 文件不存在则 `FileUtils.forceMkdir(parentFile)`（行 187）→ `writer(objectMap, templatePath, file)`（行 190）真正渲染写盘。
4. **Velocity 渲染链**（`VelocityTemplateEngine.writer(Map,String,File)`，行 78–85）：`velocityEngine.getTemplate(templatePath, UTF8)` → try-with-resources `FileOutputStream→OutputStreamWriter(UTF8)→BufferedWriter` → `template.merge(new VelocityContext(objectMap), writer)`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 默认模板引擎 | `VelocityTemplateEngine`（`AutoGenerator.execute()` 未传引擎时 new） | `AutoGenerator.java:181` |
| `TemplateLoadWay` | FILE（文件模板）；Velocity 切 TEXT 走字符串资源加载 | `VelocityTemplateEngine.java:56–64` |
| 编码 | 全程 UTF-8（`ConstVal.UTF8`） | `VelocityTemplateEngine.java:54–55` |
| 各产物 `isGenerate*` | entity/mapper/xml/service/ServiceImpl/controller 各自开关，默认 true | `AbstractTemplateEngine.outputXxx()` 各处 |
| `fileOverride` | 默认 false；命中已存在文件仅 warn 不覆盖 | `AbstractTemplateEngine.java:362–367` |
| `GlobalConfig.open` | false；true 时 `open()` 用 `RuntimeUtils.openDir` 打开目录 | `AbstractTemplateEngine.java:292–294` |
| Kotlin 输出 | `GlobalConfig.isKotlin()` 决定后缀 `.kt` 与 Kotlin 模板 | `AbstractTemplateEngine.suffixJavaOrKt()` 行 372 |

## 5. 错误与重试语义

- 渲染/写盘异常在 `outputFile()` 内 try-catch 包成 `RuntimeException`（`AbstractTemplateEngine.java:191–193`）；`batchOutput()` 再包一层文案（行 252–253）。无重试——本地生成失败即终止。
- 输出目录未配置或不存在时 `open()` 仅 warn 并 return，不报错（行 284–291）。
- Velocity 1.x 检测（静态块 `Class.forName("org.apache.velocity.util.DuckType")` 失败）仅 warn 提示升级 2.x（`VelocityTemplateEngine.java:41–48`），不阻断。
- 文件覆盖是"跳过而非报错"：`isCreate()` 返回 false 时 warn 并继续下一个产物（行 363–365）。

## 6. 并发细节

- 单线程本地渲染，无锁/无线程池；模板引擎实例一次 `execute()` 内复用。
- `objectMap` 为每表新建 `HashMap`（行 310），无跨表共享可变状态。
- 文件写入用 try-with-resources 保证流关闭（`VelocityTemplateEngine.java:80–84`）。
- 无异步回调；`RuntimeUtils.openDir` 为系统调用，失败仅 `LOGGER.error`（`AbstractTemplateEngine.java:295–297`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `engine/` 四种引擎实现与抽象基类、`fill/` 自动填充定义、`function/` 文件名转换、`util/` 文件/运行时工具。

**Out-of-Scope（不在本仓库源码内）**
- Velocity / FreeMarker / Beetl / Enjoy 模板引擎本体均为第三方依赖，不在本仓库源码内（仅封装其渲染调用）。
- 模板 `.vm`/`.ftl` 文件资源随 jar 打包，非本叶子 Java 源码；渲染数据 `TableInfo` 由 generator-query 产出。
- 配置项来源（六类配置）见 generator-config；主流程 `execute()` 编排见 generator-core。

## 8. 与相邻子系统交互

- **上游**：generator-core 的 `AutoGenerator.execute()` 调 `templateEngine.setConfigBuilder(config).init(config).batchOutput().open()`。
- **本叶子 → generator-config**：读取 `configBuilder.getStrategyConfig()/getGlobalConfig()/getPathInfo()/getInjectionConfig()` 决定输出什么、输出到哪、是否覆盖。
- **本叶子 ← generator-query**：`batchOutput()` 经 `config.getTableInfoList()` 消费采集到的 `TableInfo`。
- **下游**：产物为磁盘上的 `.java`/`.xml`/`.kt` 文件。

## 9. 语言专项适配口径（JVM）

- **模板方法模式**：`AbstractTemplateEngine` 固化"组装数据→分派输出→落盘"骨架，子类只实现 `writer/templateFilePath/init`，是经典 Java 模板方法。
- **策略/可插拔引擎**：四种引擎实现同一抽象，用户经 `FastAutoGenerator.templateEngine(...)`（`FastAutoGenerator.java:221`）切换。
- **资源管理**：文件流 try-with-resources，编码统一 UTF-8，符合 JVM 文件 IO 最佳实践。
- **构建**：Velocity/FreeMarker/Beetl/Enjoy 为 generator 模块 optional/implementation 依赖（facts.md §3），选用某引擎需引入对应依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 模板引擎架构图 | `generator-engine-architecture.html` | architecture | showcase |
| 逐表渲染输出流程 | `generator-engine-workflow.html` | workflow | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子第二图选用 workflow：`batchOutput` 是带分支（各产物开关判定、文件是否存在）的确定性分步流程，泳道可表达"引擎/策略/文件系统"三方协作；未选 sequence（无多方消息往返）与 lifecycle（无单实体状态机）。
