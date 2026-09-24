# core-foundation 域总览

> 本域是 mybatis-plus-core 的基础设施层，提供注解契约、工具、Lambda 解析、主键生成、类型处理、批量执行与支撑工具。
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。

## 1. 域职责

core-foundation 域为整个 MyBatis-Plus 提供**与 SQL 映射无直接耦合的底层能力**：注解定义实体契约、字符串/SQL 工具、Lambda 列解析、分布式主键、枚举/JSON 类型处理、批量执行、统一异常与 SPI。这些能力被 core-mapping 域与 extension 域复用。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|---|---|---|---|---|
| annotations | [annotations.md](annotations/annotations.md) | [架构图](annotations/annotations-architecture.html) | [时序](annotations/annotations-sequence.html) | 实体/表/字段注解契约 |
| toolkit | [toolkit.md](toolkit/toolkit.md) | [架构图](toolkit/toolkit-architecture.html) | [数据流](toolkit/toolkit-dataflow.html) | 字符串/SQL/全局配置工具 |
| lambda-parser | [lambda-parser.md](lambda-parser/lambda-parser.md) | [架构图](lambda-parser/lambda-parser-architecture.html) | [时序](lambda-parser/lambda-parser-sequence.html) | SFunction→列名解析 |
| id-generator | [id-generator.md](id-generator/id-generator.md) | [架构图](id-generator/id-generator-architecture.html) | [状态机](id-generator/id-generator-lifecycle.html) | 雪花分布式主键 |
| type-handlers | [type-handlers.md](type-handlers/type-handlers.md) | [架构图](type-handlers/type-handlers-architecture.html) | [时序](type-handlers/type-handlers-sequence.html) | 枚举/JSON 类型处理与字段填充 |
| batch | [batch.md](batch/batch.md) | [架构图](batch/batch-architecture.html) | [时序](batch/batch-sequence.html) | BATCH 模式批量执行 |
| core-support | [core-support.md](core-support/core-support.md) | [架构图](core-support/core-support-architecture.html) | [数据流](core-support/core-support-dataflow.html) | SQL 方法枚举/异常/SPI/拦截忽略 |

## 3. 域级机制细节

- **GlobalConfig 挂载**：toolkit/GlobalConfigUtils 用 ConcurrentHashMap 按 Configuration 缓存全局配置，本域各叶子（type-handlers/id-generator）经它取 SPI 实现。
- **注解驱动**：annotations 叶子定义契约，table-metadata（core-mapping）反射读取。
- **缓存并发**：EnumCache/WRITE_REPLACE_CACHE/MP_ENUM_CACHE 均为线程安全缓存。

## 4. 域级图

![core-foundation 架构图](core-foundation-architecture.html)
![core-foundation 时序](core-foundation-sequence.html)
![core-foundation 数据流](core-foundation-dataflow.html)
