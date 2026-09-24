# SQL 注入器（sql-injector）

> 本文是 `core-mapping` 域下的叶子子系统文档。域级总览见 `../core-mapping.md`。
> 本文展开 `core/injector/`：启动期为每个 Mapper 接口自动注册 BaseMapper 内置 CRUD 的 MappedStatement。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 注入器 SPI | `ISqlInjector`：`inspectInject` 入口 | `injector/ISqlInjector.java:36` |
| 注入器基类 | `AbstractSqlInjector`：解析 Mapper 泛型实体、初始化 TableInfo、遍历方法注入、去重缓存 | `injector/AbstractSqlInjector.java:39,44` |
| 默认注入器 | `DefaultSqlInjector`：列出 Insert/Delete/Update/Select* 等内置方法，按是否有主键决定是否加 xxById | `injector/DefaultSqlInjector.java:36,39` |
| 方法注入基类 | `AbstractMethod`：`inject` 入口 + `addMappedStatement` 委托 builderAssistant | `injector/AbstractMethod.java:53,82,396` |
| 各 CRUD 方法 | `methods/Insert/Update/Delete/SelectById/...` 20+ 个，各自实现 injectMappedStatement | `injector/methods/Insert.java` |
| SqlRunner 注入 | `SqlRunnerInjector`：为裸 SQL 运行注入语句 | `injector/SqlRunnerInjector.java:185` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `ISqlInjector` | `ISqlInjector.java:36` | 注入器 SPI |
| `AbstractSqlInjector` | `AbstractSqlInjector.java:39` | 注入编排 |
| `DefaultSqlInjector` | `DefaultSqlInjector.java:36` | 默认方法清单 |
| `AbstractMethod` | `AbstractMethod.java:53` | 单方法注入基类，`injectMappedStatement` 抽象（`:431`） |
| `methods.Insert` 等 | `methods/` | 各内置 SQL 方法 |

## 3. 关键调用链

**启动期注入流程**：

1. MyBatis 解析 Mapper 时回调 `AbstractSqlInjector.inspectInject(builderAssistant, mapperClass)`（`AbstractSqlInjector.java:44`）。
2. `ReflectionKit.getSuperClassGenericType(mapperClass, Mapper.class, 0)` 取实体类（`:45`）；查 `mapperRegistryCache` 去重（`:48-49`）。
3. `TableInfoHelper.initTableInfo(...)` 初始化表元数据（`:50`，table-metadata 叶子）。
4. `getMethodList(configuration, mapperClass, tableInfo)` 得方法列表（`:51`）；`DefaultSqlInjector` 按有无主键决定是否加 xxById（`DefaultSqlInjector.java:50`）。
5. 遍历 `methodList.forEach(m -> m.inject(...))`（`AbstractSqlInjector.java:58`）→ `AbstractMethod.inject`（`:82`）→ 子类 `injectMappedStatement`（`:431`）用 SqlMethod 模板造 SqlSource → `addMappedStatement`（`:396`）→ `builderAssistant.addMappedStatement`（`:407`）注册到 MyBatis Configuration。
6. 注入完加入 `mapperRegistryCache`（`:62`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 注入器实现 | 默认 `DefaultSqlInjector`，可 GlobalConfig 替换 | `GlobalConfig`（config-bootstrap 叶子） |
| 是否注入 xxById | 实体须有 @TableId，否则告警且不注入 | `DefaultSqlInjector.java:50-57` |
| insert 是否忽略自增列 | `dbConfig.isInsertIgnoreAutoIncrementColumn()` | `DefaultSqlInjector.java:42` |

## 5. 错误与重试语义

- 无 @TableId 时仅 `logger.warn` 不注入 xxById（`DefaultSqlInjector.java:57`），不中断。
- 注入在启动期一次性完成，失败抛异常导致启动失败；无运行期重试。
- methodList 为空时 debug 日志跳过（`AbstractSqlInjector.java:60`）。

## 6. 并发细节

- 注入发生在启动期单线程；`mapperRegistryCache` 是 Set 去重（`:48`）。
- 无运行期并发；注入完成后 MappedStatement 只读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/injector/` 全部文件。

**Out-of-Scope（不在本仓库源码内）**
- `MapperBuilderAssistant.addMappedStatement`（MyBatis，第三方）。
- TableInfo 初始化细节（table-metadata 叶子）。
- SqlMethod 模板定义（core-support 叶子）。
- Mapper 解析入口（mapper-runtime 叶子）。

## 8. 与相邻子系统交互

- 上游：mapper-runtime 叶子在解析 Mapper 时调 `inspectInject`。
- 本叶子 → 下游：读 core-support 的 `SqlMethod` 模板；依赖 table-metadata 的 `TableInfo`；最终调 MyBatis `MapperBuilderAssistant` 注册 MappedStatement。
- 数据流：**Mapper 接口 → 解析实体 → TableInfo → 方法列表 → 各方法造 SqlSource → 注册 MappedStatement**。

## 9. 语言专项适配口径（Java/JVM）

- **模板方法模式**：`AbstractMethod.injectMappedStatement` 抽象，子类实现；`addMappedStatement` 基类复用。
- **SPI 可替换**：ISqlInjector 可被用户自定义覆盖默认清单。
- **反射**：`ReflectionKit.getSuperClassGenericType` 取 Mapper 泛型实参。
- **Stream API**：`DefaultSqlInjector.getMethodList` 用 Stream.Builder 组装。
- **库型无 main**。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| SQL 注入器架构图 | `sql-injector-architecture.html` | architecture | showcase |
| 注入流程时序 | `sql-injector-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：inspectInject→initTableInfo→getMethodList→各方法 inject→addMappedStatement 是清晰的多步启动期调用时序。
未生成 dataflow/lifecycle/workflow：注入是启动期一次性流程（已 sequence 化），无运行期数据管道、无状态机、无多角色泳道审批，按资源节省原则省略。
