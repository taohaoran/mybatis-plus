# 代码生成（generator）域总览

> 本域包含 4 个叶子子系统；各叶子详情见对应文档。
> 源码基准：`mybatis-plus-generator` 分支 3.0，commit `bf67d907`（117 文件，约 1.5 万行）。

## 1. 域职责

代码生成域是 MyBatis-Plus 的"反向工程"模块：用户配置数据源与生成策略后，自动读取数据库表元数据，经模板引擎渲染出 Entity / Mapper / Mapper.xml / Service / Controller 全套代码文件。其核心范式是**编排器（AutoGenerator）驱动三段式管线：配置汇总 → 元数据采集 → 模板渲染落盘**。模块 `implementation` 依赖 mybatis-plus-spring，并内嵌 Velocity/FreeMarker/Beetl/Enjoy 四种可选模板引擎。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| 配置体系 | [generator-config.md](generator-config/generator-config.md) | [架构图](generator-config/generator-config-architecture.html) | [数据流](generator-config/generator-config-dataflow.html) | 六类配置定义、ConfigBuilder 汇总与输出路径计算 |
| 元数据查询 | [generator-query.md](generator-query/generator-query.md) | [架构图](generator-query/generator-query-architecture.html) | [数据流](generator-query/generator-query-dataflow.html) | JDBC 表/列/索引读取、过滤与列类型转换 |
| 模板引擎 | [generator-engine.md](generator-engine/generator-engine.md) | [架构图](generator-engine/generator-engine-architecture.html) | [流程](generator-engine/generator-engine-workflow.html) | 四种模板引擎实现与逐表批量渲染落盘 |
| 主流程编排 | [generator-core.md](generator-core/generator-core.md) | [架构图](generator-core/generator-core-architecture.html) | [时序](generator-core/generator-core-sequence.html) | AutoGenerator/FastAutoGenerator 编排与注解/方法 SPI |

## 3. 域级机制细节

- **三段式管线**：`AutoGenerator.execute()`（`AutoGenerator.java:173`）先建 `ConfigBuilder`（补默认值 + 算输出路径 + 反射建 `IDatabaseQuery`），再驱动 `AbstractTemplateEngine.init(config).batchOutput().open()`。
- **懒加载表信息**：`ConfigBuilder.getTableInfoList()`（`ConfigBuilder.java:170`）首次被引擎调用时才触发 `queryTables()`，避免不必要的数据库连接。
- **可插拔扩展点**：元数据查询类（`databaseQueryClass`）、列类型转换（`ITypeConvertHandler`）、模板引擎、表/字段注解处理器、Mapper 方法生成器均为接口 + 默认实现。
- **连接生命周期**：JDBC `Connection` 在 `DefaultQuery.queryTables()` 的 `finally` 中关闭，杜绝泄漏。
- **文件覆盖语义**：已存在文件且未开 `fileOverride` 时仅告警跳过，不覆盖用户手写代码。

## 4. 域级图

![代码生成域架构图](generator-architecture.html)

![生成域全流程时序](generator-sequence.html)

![生成域数据流](generator-dataflow.html)

域级三张图分别表达：四段静态拓扑（架构）、多方按时间顺序的编排链（时序）、配置→采集→渲染→输出的数据管道（数据流），与各叶子图互补。
