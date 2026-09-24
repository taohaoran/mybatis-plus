# 类型处理器与元对象填充（type-handlers）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开 `core/handlers/`：枚举类型处理器、元对象自动填充 SPI、JSON 类型处理器接口、TableInfo 后置钩子。
> JSON 类型处理器的具体实现（Jackson/Gson/Fastjson）在 extension 域，不在本叶子。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 枚举类型处理器 | `MybatisEnumTypeHandler<E>`：基于 `EnumMetadata` 在枚举值↔库值间转换， extends MyBatis `BaseTypeHandler` | `handlers/MybatisEnumTypeHandler.java:35` |
| 枚举类型处理器组合 | `CompositeEnumTypeHandler`：构造时判定是否 MP 枚举（IEnum/@EnumValue），是则委托 MybatisEnumTypeHandler，否则走 MyBatis 默认 EnumTypeHandler | `handlers/CompositeEnumTypeHandler.java:37,47` |
| 元对象填充 SPI | `MetaObjectHandler`：用户实现 `insertFill`/`updateFill` 自动写入创建/更新时间等公共字段 | `handlers/MetaObjectHandler.java:36,85,92` |
| 严格填充链 | `strictInsertFill`/`strictUpdateFill`/`strictFill`：只填充标注了 @TableField(fill) 且类型匹配、当前值为 null 的字段 | `handlers/MetaObjectHandler.java:154,181,195` |
| 填充模型 | `StrictFill<T,E>`：fieldName+fieldType+fieldVal(Supplier) 三元组，lombok @Data | `handlers/StrictFill.java:31` |
| 填充策略 | `fillStrategy`/`strictFillStrategy`：默认"有值不覆盖、提供 null 不填充" | `handlers/MetaObjectHandler.java:218,234` |
| JSON 类型处理器接口 | `IJsonTypeHandler<T>`：parse/toJson 序列化 SPI，多例实现，配合 @TableName(autoResultMap) | `handlers/IJsonTypeHandler.java:37` |
| 注解查找 SPI | `AnnotationHandler`：类/字段/方法注解查找门面，默认委托 AnnotationUtils（支持注解嵌套） | `handlers/AnnotationHandler.java:28` |
| TableInfo 后置钩子 | `PostInitTableInfoHandler`：参与 TableInfo/TableFieldInfo 初始化增强（creteTableInfo/postTableInfo/postFieldInfo） | `handlers/PostInitTableInfoHandler.java:28,37,48,60` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `MetaObjectHandler` | `MetaObjectHandler.java:36` | 填充 SPI 核心，用户实现此接口注册到 GlobalConfig |
| `StrictFill` | `StrictFill.java:31` | 严格填充参数模型 |
| `MybatisEnumTypeHandler` | `MybatisEnumTypeHandler.java:35` | MP 枚举 ↔ 库值转换，持 `EnumMetadata`（`:37`） |
| `CompositeEnumTypeHandler` | `CompositeEnumTypeHandler.java:37` | 枚举 TypeHandler 路由器，`MP_ENUM_CACHE`（`:39`）缓存判定结果 |
| `IJsonTypeHandler` | `IJsonTypeHandler.java:37` | JSON 序列化 SPI（实现见 extension） |
| `AnnotationHandler` | `AnnotationHandler.java:28` | 注解查找扩展点 |
| `PostInitTableInfoHandler` | `PostInitTableInfoHandler.java:28` | TableInfo 构建期钩子 |

## 3. 关键调用链

**插入时自动填充（@TableField(fill=INSERT)）**：

1. MyBatis 执行 insert 前，`MybatisParameterHandler`（mapper-runtime 叶子）在参数处理期回调 `MetaObjectHandler.insertFill(metaObject)`（`MetaObjectHandler.java:85`）。
2. 用户实现里调 `strictInsertFill(metaObject, "createTime", Date.class, new Date())`（`:136`）→ 包装成 `StrictFill.of`（`:137`）→ `strictFill(true, tableInfo, metaObject, strictFills)`（`:155,195`）。
3. `strictFill` 先判 `tableInfo.isWithInsertFill()`（`:196`），再遍历字段流过滤"字段名匹配 && 类型匹配 && 该字段标注了 insertFill"（`:200-202`）→ 命中则 `strictFillStrategy`（`:203`）：当前值为 null 才 setValue（`:235-239`）。

**枚举字段读写**：

1. MyBatis 注册枚举 TypeHandler 时构造 `CompositeEnumTypeHandler(enumClass)`（`CompositeEnumTypeHandler.java:43`）→ `MP_ENUM_CACHE.computeIfAbsent` 判 `EnumUtils.isMpEnums`（`:47`）。
2. 是 MP 枚举 → `delegate = new MybatisEnumTypeHandler`（`:48`）；否则走 MyBatis 默认 `EnumTypeHandler`（`:50`）。
3. SQL 写入时 `MybatisEnumTypeHandler.setNonNullParameter`（`:64`）→ `metadata.getValue(parameter)` 取库值 setObject；读取时 `getNullableResult`（`:76`）→ `metadata.valueOf(value)` 反解枚举。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `openInsertFill/openUpdateFill` | 默认 true（开启填充），可按 MappedStatement 关闭 | `MetaObjectHandler.java:55,76` |
| 默认枚举 TypeHandler | 默认 MyBatis `EnumTypeHandler`，可 `setDefaultEnumTypeHandler` 替换 | `CompositeEnumTypeHandler.java:40,54` |
| 填充策略 | 默认"有值不覆盖、null 不填充" | `MetaObjectHandler.java:218,234` |
| MetaObjectHandler 注册 | 通过 GlobalConfig 注入（见 config-bootstrap 叶子） | `GlobalConfigUtils.java:110` |

## 5. 错误与重试语义

- `MybatisEnumTypeHandler` 构造时 enumClass 为 null 抛 `IllegalArgumentException`（`:45`）。
- `CompositeEnumTypeHandler.getInstance` 反射构造 TypeHandler 失败抛 MyBatis `TypeException`（`:89,96`）。
- 填充方法本身不吞异常；`strictFill` 中 `findFirst` 未命中则不填充（静默跳过，`:203`）。
- 无重试/退避；类型转换失败由 JDBC 层抛 SQLException。

## 6. 并发细节

- `CompositeEnumTypeHandler.MP_ENUM_CACHE`（`:39`）为 `ConcurrentHashMap`，按枚举类缓存"是否 MP 枚举"判定结果，线程安全。
- `MybatisEnumTypeHandler`/`CompositeEnumTypeHandler` 为多例（每枚举类一个实例），构造后不可变，线程安全。
- `MetaObjectHandler` 由用户实现，单例注册到 GlobalConfig；其 `insertFill`/`updateFill` 在参数处理线程调用，需用户实现自身保证线程安全（无状态推荐）。
- 无线程池；填充发生在单条 SQL 执行线程内，无线程切换。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/handlers/` 全部 8 个文件。

**Out-of-Scope（不在本仓库源码内）**
- `BaseTypeHandler`/`TypeHandler`/`MetaObject`/`ResultSet`/`PreparedStatement`（MyBatis 与 JDBC，第三方）。
- JSON TypeHandler 具体实现（Jackson/Fastjson/Gson/Jackson3，extension/handlers，分片 B）。
- 触发填充的参数处理链（MybatisParameterHandler，mapper-runtime 叶子）。
- EnumMetadata/EnumUtils/EnumCache（toolkit 叶子）。

## 8. 与相邻子系统交互

- 上游：mapper-runtime 叶子的 `MybatisParameterHandler` 在 insert/update 参数处理时调 `MetaObjectHandler`。
- 本叶子 → 下游：`MybatisEnumTypeHandler` 依赖 toolkit 叶子的 `EnumUtils.metadata`/`EnumMetadata`；`CompositeEnumTypeHandler` 委托 MyBatis TypeHandler。
- 数据流：**实体参数 → MybatisParameterHandler → MetaObjectHandler 填充公共字段 → TypeHandler 把枚举/JSON 转库值 → JDBC 写入**。

## 9. 语言专项适配口径（Java/JVM）

- **MyBatis 扩展点**：`TypeHandler` 是 MyBatis SPI，MP 通过继承 `BaseTypeHandler` 接入；`MetaObject` 是 MyBatis 的对象反射包装。
- **并发原语**：`ConcurrentHashMap`（MP_ENUM_CACHE）；无锁。
- **反射**：`CompositeEnumTypeHandler.getInstance` 反射调用 TypeHandler 构造器（`:84,93`）。
- **lombok**：`StrictFill` 用 `@Data/@AllArgsConstructor`。
- **库型无 main**：嵌入业务应用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 类型处理器架构图 | `type-handlers-architecture.html` | architecture | showcase |
| 元对象填充时序 | `type-handlers-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：insert 时"ParameterHandler→MetaObjectHandler→strictFill 链→TypeHandler→JDBC"是清晰的多参与方调用时序。
未生成 dataflow/lifecycle/workflow：填充为单次方法调用链（已 sequence 化），无数据管道、无单实体状态机、无审批流程，按资源节省原则省略。
