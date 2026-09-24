# 基础工具集（toolkit）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开 core/toolkit 包内"被排除项之外"的通用工具类集合；
> Lambda 解析（LambdaUtils/support/reflect）单列 lambda-parser 叶子，主键生成（IdWorker/Sequence/SystemClock）单列 id-generator 叶子，
> 条件构造入口 Wrappers 归 conditions-wrapper 叶子，批量工具 MybatisBatchUtils 归 batch 叶子。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

本叶子是 core 模块的静态工具类集散地，全部为 `final`/`abstract` 工具类，无状态实例，供 core 各子系统静态调用。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 字符串工具 | 驼峰↔下划线互转（`camelToUnderline:151`/`underlineToCamel:173`）、判空、SQL 拼接（`sqlParam:278`/`quotaMark:296`）、`sqlArgsFill:232` | `toolkit/StringUtils.java`（631 行） |
| 常量池 | MyBatis 动态 SQL 中 `et`/`ew`/`ew.sqlSegment`/`cm`/`coll` 等 OGNL 占位符常量，继承 `StringPool` | `toolkit/Constants.java:26` |
| 表名解析器 | 用词法切分从任意 SQL 中提取涉及的表名（`accept(visitor):103`/`tables():328`/`SqlToken:340`），供动态表名/多租户改写使用 | `toolkit/TableNameParser.java`（375 行） |
| SQL 注入校验 | 两条预编译正则 `SQL_SYNTAX_PATTERN:31`/`SQL_COMMENT_PATTERN:37`，`check():46` 判断参数是否含注入特征 | `toolkit/sql/SqlInjectionUtils.java` |
| 动态 SQL 片段生成 | `convertIf:41`/`convertTrim:61`/`convertChoose:88` 拼出 MyBatis `<if>/<trim>/<choose>` XML 片段（供 SQL 注入器拼接） | `toolkit/sql/SqlScriptUtils.java:31` |
| SQL 模板/转义 | `SqlTemplate` 占位渲染、`StringEscape` 反斜杠转义 | `toolkit/sql/SqlTemplate.java`、`sql/StringEscape.java` |
| 全局配置缓存 | `GLOBAL_CONFIG` ConcurrentHashMap 按 Configuration 地址缓存 GlobalConfig（`setGlobalConfig:77`/`getGlobalConfig:88`），暴露 getIdType/getSqlInjector/getMetaObjectHandler 等门面 | `toolkit/GlobalConfigUtils.java:41` |
| 枚举缓存 | `CACHE:45` 缓存 EnumMetadata、`ENUM_VALUE_FIELD_CACHE:46` 缓存 @EnumValue 字段名；`isMpEnums:68` 判定是否 MP 枚举（IEnum 或带 @EnumValue） | `toolkit/EnumCache.java:39` |
| 反射工具 | 字段列表/Map 收集（`getFieldList:127` 沿父类上溯）、`getFieldValue:75`、`setAccessible:200`、基本类型包装映射 | `toolkit/ReflectionKit.java:41` |
| MyBatis 桥接 | 从 Mapper 代理反查 SqlSessionFactory（`getSqlSessionFactory:116/128`）、`newJsonTypeHandler:83` 反射实例化 JSON 类型处理器 | `toolkit/MybatisUtils.java:45` |
| 杂项工具 | CollectionUtils/ClassUtils/BeanUtils/ArrayUtils/ObjectUtils/Assert/AopUtils/NetUtils/SerializationUtils/ParameterUtils/PluginUtils/AES/EncryptUtils/EnumUtils/ExceptionUtils/StringPool | `toolkit/` 根目录各 .java |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `Constants`（interface） | `Constants.java:26` | 常量接口，继承 StringPool；定义 `ENTITY="et"`、`WRAPPER="ew"` 及全部 `ew.*` OGNL 路径，被 SqlScriptUtils 等 implements 复用 |
| `TableNameParser` | `TableNameParser.java:40` | SQL 词法分析器：构造时按 `NON_SQL_TOKEN_PATTERN:80` 切词，`accept(TableNameVisitor)` 回调命中关键字后的表名 |
| `TableNameVisitor` | `TableNameParser.java:152` | 表名访问者接口，扩展点（插件域动态表名/多租户据此拿到原始表名） |
| `SqlInjectionUtils` | `SqlInjectionUtils.java:27` | 注入检测静态门面，两条 Pattern 预编译 |
| `SqlScriptUtils` | `SqlScriptUtils.java:31` | abstract implements Constants；把字段/条件片段包成 MyBatis 动态标签 |
| `GlobalConfigUtils` | `GlobalConfigUtils.java:41` | 全局配置门面：以 `Integer.toHexString(configuration.hashCode())` 为 key 缓存 GlobalConfig |
| `EnumCache`/`EnumMetadata` | `EnumCache.java:39`/`EnumMetadata.java` | MP 枚举元数据缓存，供 MybatisEnumTypeHandler（type-handlers 叶子）使用 |
| `ReflectionKit` | `ReflectionKit.java:41` | 反射字段收集，沿 superclass 链上溯（`getFieldList:134-138`） |

## 3. 关键调用链

1. **动态表名插件改写 SQL 前提取表名**：插件传入原始 SQL → `new TableNameParser(sql)`（`TableNameParser.java:91`）按 `NON_SQL_TOKEN_PATTERN` 切词 → `accept(visitor)`（`:103`）在命中 from/join/update/into 等关键字后回调 visitor 记录表名 → `tables()`（`:328`）返回表名集合。
2. **全局配置读取**：任意子系统调用 `GlobalConfigUtils.getGlobalConfig(configuration)`（`GlobalConfigUtils.java:88`）→ 以 `configuration.hashCode()` 为 key 查 `GLOBAL_CONFIG`（`:46`）→ 未命中则 `CollectionUtils.computeIfAbsent` 装载 `defaults()`（`:65`）→ 再门面取 `getIdType/getSqlInjector`。
3. **SQL 注入器拼动态脚本**：`AbstractMethod` 生成 SQL 时调 `SqlScriptUtils.convertIf(sqlScript, "entity != null", true)`（`SqlScriptUtils.java:41`）→ 包成 `<if test="entity != null">...</if>` 字符串，写入 MappedStatement 的 SqlSource。
4. **枚举判定与缓存**：类型处理器注册枚举时 `EnumCache.isMpEnums(clazz)`（`EnumCache.java:68`）→ 命中 IEnum 走 `:92` 分支，否则扫 `@EnumValue` 字段（`:110`）→ 结果写入 `CACHE`/`ENUM_VALUE_FIELD_CACHE`。

## 4. 配置项

本叶子无独立配置文件。其行为受全局配置影响：

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `GlobalConfig` 缓存 | 未显式 set 时 `defaults()` 产出空 DbConfig（`GlobalConfigUtils.java:65`） | `GlobalConfigUtils.java:65` |
| 驼峰转下划线 | 由 MyBatis `mapUnderscoreToCamelCase` 控制（MP 默认 true），`StringUtils.camelToUnderline:151` 执行 | `StringUtils.java:151` |
| SQL 注入校验正则 | 两条预编译 Pattern，CASE_INSENSITIVE，不可配 | `SqlInjectionUtils.java:31-37` |

## 5. 错误与重试语义

- `SqlInjectionUtils.check:46` 对入参 `Objects.requireNonNull`，null 直接 NPE（不吞异常）。
- `GlobalConfigUtils.getGlobalConfig:88` 未初始化 configuration 时抛 `Assert.notNull` 异常（"You need Initialize MybatisConfiguration"），不做静默兜底。
- 工具类普遍"快速失败"：`Assert.notNull/isTrue`（`Assert.java`）在参数非法时抛 `MybatisPlusException`，无重试、无退避。

## 6. 并发细节

- `GlobalConfigUtils.GLOBAL_CONFIG`（`GlobalConfigUtils.java:46`）与 `EnumCache.CACHE`/`ENUM_VALUE_FIELD_CACHE`（`EnumCache.java:45-46`）均为 `ConcurrentHashMap`，启动期多线程解析实体/配置时并发写入安全；读取为无锁读。
- `SqlInjectionUtils` 的两条 `Pattern` 为 `static final` 预编译，线程安全。
- 无线程池、无异步回调、无共享可变实例状态；纯静态方法本身线程安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/toolkit/` 根目录排除 LambdaUtils/support/reflect/IdWorker/Sequence/SystemClock/Wrappers/MybatisBatchUtils 后的全部工具类；`core/toolkit/sql/` 整个 sql 子包。

**Out-of-Scope（不在本仓库源码内）**
- 真正消费表名解析结果的动态表名/多租户插件（extension-plugins 域，分片 B）。
- MyBatis 的 `DefaultReflectorFactory`/`Configuration`/`SqlSessionFactory`（org.apache.ibatis，第三方）。
- Lambda 解析链（lambda-parser 叶子）、主键雪花（id-generator 叶子）、条件构造（conditions-wrapper 叶子）、批量（batch 叶子）。

## 8. 与相邻子系统交互

- 上游：core 几乎所有子系统（sql-injector / table-metadata / mapper-runtime / type-handlers）静态调用本叶子的 StringUtils、SqlScriptUtils、GlobalConfigUtils、ReflectionKit。
- 下游：本叶子调 MyBatis 的 Configuration/ReflectorFactory（第三方）；TableNameVisitor 被 extension 插件实现。
- 数据流：**SQL 文本 → TableNameParser 词法切分 → 表名集合 → 插件改写**；**configuration → GlobalConfigUtils → GlobalConfig 门面 → 各子系统取策略**。

## 9. 语言专项适配口径（Java/JVM）

- **库型无 main**：纯静态工具 jar，被业务应用类路径加载。
- **并发原语**：`ConcurrentHashMap`（GlobalConfigUtils/EnumCache 缓存）、预编译 `Pattern` 线程安全；无 synchronized/ReentrantLock。
- **反射边界**：`ReflectionKit`/`SetAccessibleAction` 封装 `setAccessible(true)`，绕过 JVM 模块访问限制（JDK9+ 需 `--add-opens`，不在本仓库处理）。
- **常量继承**：`Constants interface extends StringPool`，通过接口多继承复用常量（Java 惯例），避免类继承滥用。
- **lombok**：部分工具类用 lombok 注解生成样板代码；构建为 Gradle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 工具集架构图 | `toolkit-architecture.html` | architecture | showcase |
| SQL 安全与脚本数据流 | `toolkit-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 dataflow 图：TableNameParser 切词→表名提取→插件改写、SqlScriptUtils 片段拼接是清晰的"源→处理→目的"管道，画数据流图信息增益明显。
未生成 sequence/lifecycle/workflow：本叶子是无状态静态工具集合，无单实体状态机、无多角色审批流程、调用链为单次静态调用（已在第 3 节文字描述），按资源节省原则省略。
