# core-mapping 域总览

> 本域是 mybatis-plus-core 的 SQL 映射层：条件构造、SQL 注入、表元数据、Mapper 运行时、配置引导。
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。

## 1. 域职责

core-mapping 域把实体注解翻译成 MyBatis 的 MappedStatement 并在运行期执行：条件构造器拼 where 片段、SQL 注入器启动期注册内置 CRUD、表元数据缓存实体↔表映射、Mapper 运行时代理分发与参数填充、配置引导装配全局组件。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|---|---|---|---|---|
| conditions-wrapper | [conditions-wrapper.md](conditions-wrapper/conditions-wrapper.md) | [架构图](conditions-wrapper/conditions-wrapper-architecture.html) | [数据流](conditions-wrapper/conditions-wrapper-dataflow.html) | 链式 where/orderBy 条件构造 |
| sql-injector | [sql-injector.md](sql-injector/sql-injector.md) | [架构图](sql-injector/sql-injector-architecture.html) | [时序](sql-injector/sql-injector-sequence.html) | 启动期注入内置 CRUD |
| table-metadata | [table-metadata.md](table-metadata/table-metadata.md) | [架构图](table-metadata/table-metadata-architecture.html) | [时序](table-metadata/table-metadata-sequence.html) | 实体↔表元数据缓存 |
| mapper-runtime | [mapper-runtime.md](mapper-runtime/mapper-runtime.md) | [架构图](mapper-runtime/mapper-runtime-architecture.html) | [时序](mapper-runtime/mapper-runtime-sequence.html) | Mapper 代理分发与参数填充 |
| config-bootstrap | [config-bootstrap.md](config-bootstrap/config-bootstrap.md) | [架构图](config-bootstrap/config-bootstrap-architecture.html) | [数据流](config-bootstrap/config-bootstrap-dataflow.html) | GlobalConfig 装配与构建入口 |

## 3. 域级机制细节

- **启动期→运行期两阶段**：config-bootstrap 装配 → sql-injector 经 table-metadata 生成 MappedStatement → mapper-runtime 运行期代理执行。
- **可插拔 SPI**：GlobalConfig 持有 sqlInjector/metaObjectHandler/identifierGenerator，默认实现兜底。
- **缓存**：TABLE_INFO_CACHE 实体元数据、methodCache 方法 invoker。

## 4. 域级图

![core-mapping 架构图](core-mapping-architecture.html)
![core-mapping 时序](core-mapping-sequence.html)
![core-mapping 数据流](core-mapping-dataflow.html)
