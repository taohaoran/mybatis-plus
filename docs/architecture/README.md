# mybatis-plus 系统架构文档

> 基于 mybatis-plus 源码（com.baomidou，Apache-2.0，v3.5.17，分支 3.0，commit `bf67d907`）深度分析产出。
> 覆盖系统级、7 个域、31 个叶子子系统的功能、问题域、系统边界、架构图、时序图与数据流图。
> 所有图表由 archify 渲染为自包含交互式 HTML（单文件 780KB+）。
> 主语言 Java（JVM 口径），少量 Kotlin；Gradle 多模块构建；约 497 Java 文件 + 5 Kotlin，6.7 万行。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 项目概述、功能总览、解决的问题、系统边界、架构/时序/数据流/启动流程说明 | [架构图](system-architecture.html) · [时序图](system-sequence.html) · [数据流图](system-dataflow.html) · [启动流程图](system-workflow.html) |

---

### core-foundation 域（核心基础域，7 叶）

域总览：[core-foundation.md](core-foundation/core-foundation.md) · [域架构图](core-foundation/core-foundation-architecture.html) · [域时序图](core-foundation/core-foundation-sequence.html) · [域数据流图](core-foundation/core-foundation-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| annotations | [annotations.md](core-foundation/annotations/annotations.md) | [架构图](core-foundation/annotations/annotations-architecture.html) | [时序图](core-foundation/annotations/annotations-sequence.html) | 16 个注解/枚举契约 |
| toolkit | [toolkit.md](core-foundation/toolkit/toolkit.md) | [架构图](core-foundation/toolkit/toolkit-architecture.html) | [数据流图](core-foundation/toolkit/toolkit-dataflow.html) | 基础工具库（SQL 安全/反射/加密） |
| lambda-parser | [lambda-parser.md](core-foundation/lambda-parser/lambda-parser.md) | [架构图](core-foundation/lambda-parser/lambda-parser-architecture.html) | [时序图](core-foundation/lambda-parser/lambda-parser-sequence.html) | SFunction 序列化 Lambda 解析 |
| id-generator | [id-generator.md](core-foundation/id-generator/id-generator.md) | [架构图](core-foundation/id-generator/id-generator-architecture.html) | [生命周期图](core-foundation/id-generator/id-generator-lifecycle.html) | 主键生成策略（雪花/序列） |
| type-handlers | [type-handlers.md](core-foundation/type-handlers/type-handlers.md) | [架构图](core-foundation/type-handlers/type-handlers-architecture.html) | [时序图](core-foundation/type-handlers/type-handlers-sequence.html) | 枚举类型处理器/元对象填充 |
| batch | [batch.md](core-foundation/batch/batch.md) | [架构图](core-foundation/batch/batch-architecture.html) | [时序图](core-foundation/batch/batch-sequence.html) | 批量执行会话 |
| core-support | [core-support.md](core-foundation/core-support/core-support.md) | [架构图](core-foundation/core-support/core-support-architecture.html) | [数据流图](core-foundation/core-support/core-support-dataflow.html) | 枚举/异常/SPI/兼容层支撑 |

---

### core-mapping 域（核心映射域，5 叶）

域总览：[core-mapping.md](core-mapping/core-mapping.md) · [域架构图](core-mapping/core-mapping-architecture.html) · [域时序图](core-mapping/core-mapping-sequence.html) · [域数据流图](core-mapping/core-mapping-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| conditions-wrapper | [conditions-wrapper.md](core-mapping/conditions-wrapper/conditions-wrapper.md) | [架构图](core-mapping/conditions-wrapper/conditions-wrapper-architecture.html) | [数据流图](core-mapping/conditions-wrapper/conditions-wrapper-dataflow.html) | 条件构造器 Wrapper/LambdaWrapper |
| sql-injector | [sql-injector.md](core-mapping/sql-injector/sql-injector.md) | [架构图](core-mapping/sql-injector/sql-injector-architecture.html) | [时序图](core-mapping/sql-injector/sql-injector-sequence.html) | SQL 注入器（20 个通用方法） |
| table-metadata | [table-metadata.md](core-mapping/table-metadata/table-metadata.md) | [架构图](core-mapping/table-metadata/table-metadata-architecture.html) | [时序图](core-mapping/table-metadata/table-metadata-sequence.html) | TableInfo 表元数据解析 |
| mapper-runtime | [mapper-runtime.md](core-mapping/mapper-runtime/mapper-runtime.md) | [架构图](core-mapping/mapper-runtime/mapper-runtime-architecture.html) | [时序图](core-mapping/mapper-runtime/mapper-runtime-sequence.html) | Mapper 代理运行时与注解解析 |
| config-bootstrap | [config-bootstrap.md](core-mapping/config-bootstrap/config-bootstrap.md) | [架构图](core-mapping/config-bootstrap/config-bootstrap-architecture.html) | [数据流图](core-mapping/config-bootstrap/config-bootstrap-dataflow.html) | GlobalConfig 与 SqlSessionFactory 引导 |

---

### extension-plugins 域（扩展插件域，10 叶）

域总览：[extension-plugins.md](extension-plugins/extension-plugins.md) · [域架构图](extension-plugins/extension-plugins-architecture.html) · [域时序图](extension-plugins/extension-plugins-sequence.html) · [域数据流图](extension-plugins/extension-plugins-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| interceptor-core | [interceptor-core.md](extension-plugins/interceptor-core/interceptor-core.md) | [架构图](extension-plugins/interceptor-core/interceptor-core-architecture.html) | [时序图](extension-plugins/interceptor-core/interceptor-core-sequence.html) | 拦截器链调度 + 乐观锁/动态表名 |
| pagination | [pagination.md](extension-plugins/pagination/pagination.md) | [架构图](extension-plugins/pagination/pagination-architecture.html) | [数据流图](extension-plugins/pagination/pagination-dataflow.html) | 分页模型与 14 种方言 |
| injector-ext | [injector-ext.md](extension-plugins/injector-ext/injector-ext.md) | [架构图](extension-plugins/injector-ext/injector-ext-architecture.html) | [流程图](extension-plugins/injector-ext/injector-ext-workflow.html) | 扩展注入方法（批量插入/upsert 等） |
| ddl | [ddl.md](extension-plugins/ddl/ddl.md) | [架构图](extension-plugins/ddl/ddl-architecture.html) | [流程图](extension-plugins/ddl/ddl-workflow.html) | DDL 脚本版本化执行 |
| ar-repository | [ar-repository.md](extension-plugins/ar-repository/ar-repository.md) | [架构图](extension-plugins/ar-repository/ar-repository-architecture.html) | [时序图](extension-plugins/ar-repository/ar-repository-sequence.html) | ActiveRecord + Repository 模式 |
| json-handlers | [json-handlers.md](extension-plugins/json-handlers/json-handlers.md) | [架构图](extension-plugins/json-handlers/json-handlers-architecture.html) | —（省略，见 MD 第 10 节） | JSON 类型处理器（Jackson/Fastjson/Gson） |
| chain-conditions | [chain-conditions.md](extension-plugins/chain-conditions/chain-conditions.md) | [架构图](extension-plugins/chain-conditions/chain-conditions-architecture.html) | —（省略，见 MD 第 10 节） | 链式条件包装 |
| db-toolkit | [db-toolkit.md](extension-plugins/db-toolkit/db-toolkit.md) | [架构图](extension-plugins/db-toolkit/db-toolkit-architecture.html) | [时序图](extension-plugins/db-toolkit/db-toolkit-sequence.html) | Db 静态 CRUD/SimpleQuery 工具 |
| key-generators | [key-generators.md](extension-plugins/key-generators/key-generators.md) | [架构图](extension-plugins/key-generators/key-generators-architecture.html) | —（省略，见 MD 第 10 节） | 数据库序列键生成器（9 种方言） |
| scripting-p6spy | [scripting-p6spy.md](extension-plugins/scripting-p6spy/scripting-p6spy.md) | [架构图](extension-plugins/scripting-p6spy/scripting-p6spy-architecture.html) | [时序图](extension-plugins/scripting-p6spy/scripting-p6spy-sequence.html) | 模板语言驱动 + p6spy SQL 日志 |

---

### jsqlparser 域（SQL 解析域，2 叶）

域总览：[jsqlparser.md](jsqlparser/jsqlparser.md) · [域架构图](jsqlparser/jsqlparser-architecture.html) · [域时序图](jsqlparser/jsqlparser-sequence.html) · [域数据流图](jsqlparser/jsqlparser-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| parser-cache | [parser-cache.md](jsqlparser/parser-cache/parser-cache.md) | [架构图](jsqlparser/parser-cache/parser-cache-architecture.html) | [数据流图](jsqlparser/parser-cache/parser-cache-dataflow.html) | SQL AST 解析与缓存（FST/Fury/Caffeine） |
| sql-interceptors | [sql-interceptors.md](jsqlparser/sql-interceptors/sql-interceptors.md) | [架构图](jsqlparser/sql-interceptors/sql-interceptors-architecture.html) | [数据流图](jsqlparser/sql-interceptors/sql-interceptors-dataflow.html) | 多租户/数据权限/防全表更新智能拦截器 |

---

### generator 域（代码生成域，4 叶）

域总览：[generator.md](generator/generator.md) · [域架构图](generator/generator-architecture.html) · [域时序图](generator/generator-sequence.html) · [域数据流图](generator/generator-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| generator-config | [generator-config.md](generator/generator-config/generator-config.md) | [架构图](generator/generator-config/generator-config-architecture.html) | [数据流图](generator/generator-config/generator-config-dataflow.html) | 生成器配置体系（策略/模板/包/注入） |
| generator-query | [generator-query.md](generator/generator-query/generator-query.md) | [架构图](generator/generator-query/generator-query-architecture.html) | [数据流图](generator/generator-query/generator-query-dataflow.html) | 数据库元数据查询与类型转换 |
| generator-engine | [generator-engine.md](generator/generator-engine/generator-engine.md) | [架构图](generator/generator-engine/generator-engine-architecture.html) | [流程图](generator/generator-engine/generator-engine-workflow.html) | 模板引擎（Velocity/Freemarker/Beetl/Enjoy） |
| generator-core | [generator-core.md](generator/generator-core/generator-core.md) | [架构图](generator/generator-core/generator-core-architecture.html) | [时序图](generator/generator-core/generator-core-sequence.html) | AutoGenerator 主流程编排 |

---

### spring 域（Spring 集成域，1 叶）

域总览：[spring.md](spring/spring.md) · [域架构图](spring/spring-architecture.html)（域级图与叶子图合并计 3 张）

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| spring-integration | [spring-integration.md](spring/spring-integration/spring-integration.md) | [架构图](spring/spring-integration/spring-integration-architecture.html) | [时序图](spring/spring-integration/spring-integration-sequence.html) | IService/FactoryBean/SqlRunner/CrudRepository |

---

### boot-starter 域（启动器域，2 叶）

域总览：[boot-starter.md](boot-starter/boot-starter.md) · [域架构图](boot-starter/boot-starter-architecture.html) · [域时序图](boot-starter/boot-starter-sequence.html) · [域数据流图](boot-starter/boot-starter-dataflow.html)

| 叶子 | 文档 | 架构图 | 第二图 | 职责 |
|------|------|--------|--------|------|
| boot-autoconfigure | [boot-autoconfigure.md](boot-starter/boot-autoconfigure/boot-autoconfigure.md) | [架构图](boot-starter/boot-autoconfigure/boot-autoconfigure-architecture.html) | [流程图](boot-starter/boot-autoconfigure/boot-autoconfigure-workflow.html) | Boot2/3/4 自动配置与属性绑定 |
| boot-native-test | [boot-native-test.md](boot-starter/boot-native-test/boot-native-test.md) | [架构图](boot-starter/boot-native-test/boot-native-test-architecture.html) | [数据流图](boot-starter/boot-native-test/boot-native-test-dataflow.html) | GraalVM AOT 原生镜像 + 测试自动配置 |

---

## 产出统计

| 层级 | MD 文档 | HTML 图 | JSON IR | 说明 |
|------|---------|---------|---------|------|
| 系统级 | 2（README + system-overview） | 4 | 4 | architecture/sequence/dataflow/workflow |
| 域级 | 7 | 19 | 19 | 每域 ≥3 张（spring 域与叶子图合并计算） |
| 叶子级 | 31 | 59 | 59 | 每叶 ≥2 张（3 叶按资源节省原则省略第二图并已披露） |
| **合计** | **40** | **82** | **82** | HTML 与 JSON 一一对应 |

## 质量档位说明

| 层级 | showcase | standard | 说明 |
|------|----------|----------|------|
| 系统级 | 4 | 0 | 全部 showcase |
| 域级 | 19 | 0 | 全部 showcase |
| 叶子级 | 57 | 2 | id-generator-lifecycle（label-route-clearance）、boot-native-test-dataflow，均已在对应 MD 第 10 节披露失败检查名与修复动作 |

所有 HTML 均 >700KB（自包含交互式），render 退出码均为 0。

## 覆盖范围与说明

- **覆盖**：7 域 31 叶，覆盖 mybatis-plus 全部 main 源码模块（annotation/core/extension/generator/spring/jsqlparser-support/boot-starter）
- **语言适配口径**：JVM（Java 为主，少量 Kotlin），已按 language-java.md 清单落到各叶第 9 节（并发模型/库型无 main/模块边界/配置面/构建与代码生成）
- **省略说明**：extension-plugins 域的 json-handlers、chain-conditions、key-generators 三叶为扁平结构（基类模板方法/CRTP 代理/序列实现），无独立时序/管道/状态机语义，按 diagram-policy 资源节省原则省略第二图，原因已写入各叶第 10 节
- **jsqlparser 三版本**：4.9（兼容 jdk8）/5.0/聚合 5.2 三套实现 + 共用 common 模块，已在 parser-cache、sql-interceptors 及域总览中对比说明
- **boot2/3/4 矩阵**：三套近同构 MybatisPlusAutoConfiguration（392/400/400 行），差异在 Spring Boot 大版本与 jakarta 命名空间，已在 boot-autoconfigure 说明
- **未覆盖**：测试代码（src/test）、构建脚本（build.gradle 细节）、CHANGELOG 等非 main 源码内容
- **源码基准**：分支 3.0，commit `bf67d907478c724120bf76292da54abf9e73c2b3`
