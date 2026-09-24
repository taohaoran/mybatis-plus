# 注解契约（annotations）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文只展开 mybatis-plus-annotation 模块对外暴露的"实体-表映射注解与枚举契约"，
> 不展开这些注解被如何解析装配（见 core-mapping 域的 table-metadata / sql-injector 叶子）。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-annotation`（16 个源文件，约 1187 行）。

## 1. 功能清单

本叶子是整个 MyBatis-Plus 体系的**最底层契约模块**：只定义注解、枚举与常量，不含任何运行时逻辑。
它被 `mybatis-plus-core`（及更上层 extension/spring/starter）依赖，是"只做增强不做改变"声明的实体映射契约面。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 实体-表映射注解 | `@TableName` 标注实体类对应数据库表名、schema、前缀、resultMap 自动构建 | `annotation/TableName.java` |
| 主键注解 | `@TableId` 标注主键字段与主键类型 `IdType` | `annotation/TableId.java` |
| 字段注解 | `@TableField` 标注字段名、是否存在、where 条件、update 片段、insert/update/where 三种策略、自动填充、是否 select、typeHandler、jdbcType 等（最复杂的注解，188 行） | `annotation/TableField.java` |
| 逻辑删除注解 | `@TableLogic` 标注逻辑删除字段，配置未删除值/删除值 | `annotation/TableLogic.java` |
| 乐观锁注解 | `@Version` 标注乐观锁版本字段（支持 long/int/Date/LocalDateTime/Instant） | `annotation/Version.java` |
| 自动排序注解 | `@OrderBy` 字段级自动排序（asc 方向 + sort 优先级） | `annotation/OrderBy.java` |
| 序列主键注解 | `@KeySequence` 标注 Oracle 等序列名与 DbType，`@Inherited` 可继承 | `annotation/KeySequence.java` |
| 枚举值注解 | `@EnumValue` 标注普通枚举类中作为数据库存储值的字段 | `annotation/EnumValue.java` |
| 枚举接口 | `IEnum<T>` 自定义枚举接口，实现 `getValue()` 返回库存储值 | `annotation/IEnum.java` |
| 拦截器忽略注解 | `@InterceptorIgnore` 标注在 Mapper/Method 上，逐项关闭租户/动态表名/攻击阻断/垃圾SQL/数据权限等内置插件 | `annotation/InterceptorIgnore.java` |
| 主键策略枚举 | `IdType`：AUTO/NONE/INPUT/ASSIGN_ID/ASSIGN_UUID | `annotation/IdType.java` |
| 字段策略枚举 | `FieldStrategy`：ALWAYS/NOT_NULL/NOT_EMPTY/NEVER/DEFAULT（DEFAULT 跟随全局） | `annotation/FieldStrategy.java` |
| 填充策略枚举 | `FieldFill`：DEFAULT/INSERT/UPDATE/INSERT_UPDATE | `annotation/FieldFill.java` |
| 数据库类型枚举 | `DbType`：40+ 种数据库方言标识（mysql/oracle/postgresql/dm/kingbasees/oceanbase/clickhouse/trino 等），含 `getDbType()` 反查 | `annotation/DbType.java` |
| SQL 条件常量 | `SqlCondition`：`EQUAL/NOT_EQUAL/LIKE/ORACLE_LIKE/LIKE_LEFT/LIKE_RIGHT` 模板串（`%s=#{%s}` 形式） | `annotation/SqlCondition.java` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `@TableName` | `TableName.java:29` | 类级注解，value 表名、schema、keepGlobalPrefix、resultMap、autoResultMap、properties/excludeProperty（3.5.10 起支持白名单属性） |
| `@TableId` | `TableId.java:29` | 字段级注解，value 列名、type 指定 `IdType`（默认 NONE=跟随全局） |
| `@TableField` | `TableField.java:35` | 字段级注解，是 MP 字段映射的全部契约汇聚点：列名/存在性/condition/update 片段/三种策略/fill/select/keepGlobalFormat/property/jdbcType/typeHandler/javaType/numericScale |
| `IdType` | `IdType.java:27` | 主键策略枚举，带 `int key`；ASSIGN_ID 默认走雪花 `DefaultIdentifierGenerator`，ASSIGN_UUID 走去横线 UUID |
| `FieldStrategy` | `FieldStrategy.java:26` | 字段插入/更新/where 拼接策略；DEFAULT 在注解里代表"跟随全局"，在全局里代表 NOT_NULL |
| `FieldFill` | `FieldFill.java:30` | 自动填充时机枚举，优先级高于 FieldStrategy（填充字段断言必有值） |
| `IEnum<T>` | `IEnum.java:26` | 自定义枚举存储值接口，`getValue()` 返回落库值 |
| `@InterceptorIgnore` | `InterceptorIgnore.java:37` | 插件级开关注解，支持 TYPE/METHOD 两级（method 优先）；dataPermission 默认 `"1"`（默认关闭需显式打开），others 支持 `key@1` 格式 |
| `DbType` | `DbType.java` | 方言枚举，code+desc 两元组，是多数据库支持的身份标识 |

> 扩展点说明：`@TableField.typeHandler` 指向 MyBatis 原生 `TypeHandler`（`org.apache.ibatis.type`，不在本仓库）；
> `@InterceptorIgnore` 各属性注释里引用的 `TenantLineInnerInterceptor` 等实现在 extension 插件域（不在本叶子）。

## 3. 关键调用链

本叶子本身**无方法调用链**（纯声明）。其"被消费"的典型装配时序如下（与第二张时序图互相印证）：

1. 应用启动构建 `SqlSessionFactory` 时，`MybatisMapperAnnotationBuilder` 解析 Mapper 关联的实体类（见 mapper-runtime / table-metadata 叶子）。
2. `TableInfoHelper.initTableInfo(...)` 读取实体类上的 `@TableName`（`TableName.java:29`）确定表名/schema/autoResultMap；遍历字段读取 `@TableId`（`TableId.java:29`）、`@TableField`（`TableField.java:35`）、`@TableLogic`、`@Version`、`@OrderBy`、`@EnumValue`。
3. 读到 `@TableField.fill`（`TableField.java:120`）时登记需要自动填充的字段，交给 `MetaObjectHandler`（type-handlers 叶子）在 insert/update 时填充。
4. `@InterceptorIgnore`（`InterceptorIgnore.java:37`）在插件拦截期被 `InterceptorIgnoreHelper`（core-support 叶子）读取，决定是否跳过对应 InnerInterceptor。

> 上述行号均为本叶子注解定义处；真正的解析逻辑落在 table-metadata 叶子的 `TableInfoHelper`，不在本叶子源码内。

## 4. 配置项

注解本身即"配置面"，全部通过注解属性暴露，无外部配置文件：

| 配置项 | 默认值 | 行为 | 位置 |
|---|---|---|---|
| `@TableName.value` | `""` | 空则按全局 tablePrefix + 类名驼峰转下划线推导表名 | `TableName.java:34` |
| `@TableName.autoResultMap` | `false` | true 时 MP 自动构建 resultMap（配合字段级 typeHandler/jdbcType） | `TableName.java:68` |
| `@TableField.exist` | `true` | false 表示该属性非数据库字段 | `TableField.java:53` |
| `@TableField.fill` | `FieldFill.DEFAULT` | 填充时机，优先级高于 insert/updateStrategy | `TableField.java:120` |
| `@TableField.select` | `true` | false 时不加入默认 select 列（大字段） | `TableField.java:127` |
| `@TableId.type` | `IdType.NONE` | NONE=跟随全局主键策略 | `TableId.java:40` |
| `@InterceptorIgnore.dataPermission` | `"1"` | 数据权限默认关闭（需显式注解打开） | `InterceptorIgnore.java:64` |

## 5. 错误与重试语义

本叶子无运行时错误路径（纯声明）。语义约束由消费方在解析期校验：
- 注解属性非法组合（如 `@TableName.resultMap` 与 `autoResultMap` 同时生效）由 `TableInfoHelper` 裁决（resultMap 优先，autoResultMap 不生效，见 `TableName.java:63` 注释）。
- `@EnumValue` 标注在非枚举字段上、`@Version` 标注在不支持类型上，由消费方在解析/拦截期给出警告或异常，不在本叶子内。

## 6. 并发细节

本叶子无并发语义：注解与枚举在类加载期一次性加载，运行期只读，无线程切换、无锁、无共享可变状态。
`DbType` 为枚举常量池，`SqlCondition` 为 `public static final String`，天然不可变。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `mybatis-plus-annotation` 包全部 16 个文件：实体映射注解、策略枚举、方言枚举、SQL 条件常量、枚举接口。

**Out-of-Scope（不在本仓库源码内）**
- 注解的解析与装配：`TableInfoHelper`/`TableInfo`（core-mapping/table-metadata 叶子）。
- 注解驱动的 SQL 注入：`AbstractMethod` 系列（core-mapping/sql-injector 叶子）。
- 插件对 `@InterceptorIgnore` 的读取：`InterceptorIgnoreHelper`（core-support）与各 InnerInterceptor（extension-plugins 域，不在本分片）。
- MyBatis 原生 `TypeHandler`/`ResultMapping`/`ParameterMapping`（org.apache.ibatis，第三方依赖，不在本仓库）。
- 本叶子**不做**：不实现任何 ORM 运行逻辑，不依赖 spring/mybatis-spring，是依赖图最底端模块。

## 8. 与相邻子系统交互

- 上游（被依赖方向）：`mybatis-plus-core` 依赖本模块。core 在实体解析、SQL 注入、类型处理时读取本模块注解/枚举。
- 本叶子 → 下游第三方：仅 import MyBatis 的 `JdbcType`/`TypeHandler`/`ResultMapping`（`TableField.java:18-22`），不反向依赖业务代码。
- 数据流方向：**实体类（用户代码）→ 标注本模块注解 → core 解析注解 → 生成 TableInfo 元数据 → 注入 CRUD MappedStatement**。本叶子处于该链路最前的"声明"环节。

## 9. 语言专项适配口径（Java/JVM）

- **库型项目无 main 入口**：本模块是纯契约 jar，无任何 `main` 方法，作为依赖被打包进业务应用。
- **依赖方向**：annotation 模块仅依赖 mybatis（api），是整个依赖链 `annotation → core → extension → spring → generator` 的最底端，无反向依赖。
- **注解Retention**：全部 `RUNTIME`（`@Retention(RetentionPolicy.RUNTIME)`），运行期反射读取；`@KeySequence` 额外 `@Inherited` 支持继承。
- **构建**：Gradle 多模块，lombok `@Getter` 用于 `IdType`/`DbType` 枚举生成 getter（见 `IdType.java:26`），不手写 getter。
- **并发模型**：无——纯声明模块，无需线程池/锁分析。
- **配置面**：本模块即"声明式配置面"的源头，运行期默认值最终在 core 的 `GlobalConfig`/`MybatisConfiguration` 落地（见 config-bootstrap 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 注解契约架构图 | `annotations-architecture.html` | architecture | showcase |
| 注解被消费装配时序 | `annotations-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：虽为纯声明模块，但"注解声明 → core 反射解析 → 元数据构建"是清晰的多参与方时序交互，
画出来可把注解与 table-metadata 叶子的衔接关系可视化，信息增益明显。
未生成 dataflow/lifecycle/workflow：本叶子无数据管道、无单实体状态机、无多角色审批流程语义，按资源节省原则省略。
