# 表元数据（table-metadata）

> 本文是 `core-mapping` 域下的叶子子系统文档。域级总览见 `../core-mapping.md`。
> 本文展开 `core/metadata/`：实体类 ↔ 数据库表的反射元数据（TableInfo/TableFieldInfo）及其缓存初始化。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 表元数据模型 | `TableInfo`：表名/主键/字段列表/ResultMap 等 | `metadata/TableInfo.java:596` |
| 字段元数据模型 | `TableFieldInfo`：单个字段列名/类型/Fill/TypeHandler 等 | `metadata/TableFieldInfo.java:609` |
| 元数据初始化器 | `TableInfoHelper`：反射扫描实体、解析 @TableName/@TableId/@TableField、建缓存 | `metadata/TableInfoHelper.java:656` |
| 缓存 | `TABLE_INFO_CACHE`（实体类→TableInfo）、`TABLE_NAME_INFO_CACHE`（表名→TableInfo） | `TableInfoHelper.java:62,67` |
| 分页模型 | `IPage`：分页参数与记录 | `metadata/IPage.java:185` |
| 排序项 | `OrderItem`/`OrderFieldInfo`：排序字段 | `metadata/OrderItem.java:144` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `TableInfo` | `TableInfo.java:192` | 单表元数据，持 `keyProperty`（`:82`）、`fieldList`（`:94`） |
| `TableFieldInfo` | `TableFieldInfo.java` | 单字段元数据 |
| `TableInfoHelper` | `TableInfoHelper.java:185` | `initTableInfo` 同步初始化入口 |
| `IPage` | `IPage.java:185` | 分页接口 |
| `OrderItem` | `OrderItem.java:144` | 排序项 |

## 3. 关键调用链

**实体元数据初始化**：

1. sql-injector 调 `TableInfoHelper.initTableInfo(builderAssistant, modelClass)`（`AbstractSqlInjector.java:50`）→ 内部 `initTableInfo(configuration, namespace, clazz)`（`TableInfoHelper.java:185`，synchronized）。
2. 先查 `TABLE_INFO_CACHE.get`（`:88`）命中直接返回；未命中则 `new TableInfo(...)`（`TableInfo.java:192`）。
3. `initTableName`（`TableInfoHelper.java:221`）解析 @TableName 与全局表名前缀/后缀。
4. `initTableFields`（`:320`）反射遍历字段，@TableId 走 `initTableIdWithAnnotation`（`:478`），其余建 `TableFieldInfo`。
5. 经 `PostInitTableInfoHandler` 钩子后置处理（`:203`）→ 放入 `TABLE_INFO_CACHE`（`:203`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 默认主键名 | `id`（无 @TableId 时按字段名 id 推断） | `TableInfoHelper.java:72` |
| 表名前后缀 | GlobalConfig.DbConfig 全局配置 | `TableInfoHelper.java:295` |
| 是否开启驼峰转下划线 | 全局配置（toolkit/StringUtils.camelToUnderline） | 字段命名 |

## 5. 错误与重试语义

- 无 @TableId 时 `havePK()`（`TableInfo.java:216`）为 false，xxById 方法不注入（见 sql-injector）。
- 初始化失败抛异常导致启动失败；无运行期重试。
- 反射异常由 MyBatis/ReflectionKit 包装抛出。

## 6. 并发细节

- `TABLE_INFO_CACHE`/`TABLE_NAME_INFO_CACHE`（`TableInfoHelper.java:62,67`）为 ConcurrentHashMap，线程安全。
- `initTableInfo`（`:185`）加 synchronized 防并发重复初始化；缓存命中后无锁。
- TableInfo/TableFieldInfo 初始化后不可变，运行期只读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/metadata/` 全部文件。

**Out-of-Scope（不在本仓库源码内）**
- 注解定义（annotations 叶子，mybatis-plus-annotation 模块）。
- 反射工具（ReflectionKit，toolkit 叶子）。
- ResultMapping/Configuration（MyBatis，第三方）。

## 8. 与相邻子系统交互

- 上游：sql-injector 调 `initTableInfo`。
- 本叶子 → 下游：读取 annotations 叶子的 @TableName/@TableId/@TableField；输出 TableInfo 供 sql-injector/mapper-runtime/type-handlers 使用。
- 数据流：**实体类 → 反射扫描 → 注解解析 → TableInfo 缓存 → 供注入与运行期**。

## 9. 语言专项适配口径（Java/JVM）

- **反射**：`initTableFields` 反射遍历 Field。
- **ConcurrentHashMap + synchronized**：缓存并发初始化。
- **DTO 不可变**：TableInfo 构建后只读。
- **库型无 main**。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 表元数据架构图 | `table-metadata-architecture.html` | architecture | showcase |
| 元数据初始化时序 | `table-metadata-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：initTableInfo→initTableName→initTableFields→入缓存是清晰的多步初始化时序。
未生成 dataflow/lifecycle/workflow：初始化是一次性反射流程（已 sequence 化），无运行期管道、无状态机、无审批流，按资源节省原则省略。
