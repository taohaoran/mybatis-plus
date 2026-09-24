# 扩展注入方法（injector-ext）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
> 本文只展开 extension 模块新增的 5 个通用 Mapper 方法注入器；core 的 `AbstractMethod`/`ISqlInjector` 体系见 core-mapping 域 `sql-injector` 叶子。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 批量自选字段插入 | `InsertBatchSomeColumn`：单条 `insert into t (cols) values (..),(..)` 多值插入，支持 `Predicate<TableFieldInfo>` 筛选字段 | `injector/methods/InsertBatchSomeColumn.java:62` |
| 按 id 总是更新指定列 | `AlwaysUpdateSomeColumnById`：update 时只更新筛选出的字段列 | `injector/methods/AlwaysUpdateSomeColumnById.java` |
| 逻辑删除批量 | `LogicDeleteBatchByIds`：批量逻辑删除 | `injector/methods/LogicDeleteBatchByIds.java` |
| 逻辑删除并填充 | `LogicDeleteByIdWithFill`：单条逻辑删除同时触发字段自动填充 | `injector/methods/LogicDeleteByIdWithFill.java` |
| Upsert | `Upsert`：存在则更新否则插入（数据库方言相关） | `injector/methods/Upsert.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `InsertBatchSomeColumn` | `injector/methods/InsertBatchSomeColumn.java:62` | 继承 core `AbstractMethod`；`injectMappedStatement` 拼接批量 insert SQL |
| `predicate` 字段 | `InsertBatchSomeColumn.java:69` | `Predicate<TableFieldInfo>` 字段过滤器，链式 setter |
| `AbstractMethod` | core `injector/AbstractMethod.java` | 方法注入基类（core-mapping 叶子） |
| `Upsert` | `injector/methods/Upsert.java` | 方言相关 upsert 方法注入 |

## 3. 关键调用链

**链 1：InsertBatchSomeColumn 注入 MappedStatement**

1. 注入器在 Mapper 初始化时被调 `injectMappedStatement(mapperClass, modelClass, tableInfo)`（`InsertBatchSomeColumn.java:100`）。
2. 取 `tableInfo.getFieldList()`，用 `filterTableFieldInfo(fieldList, predicate, ...)` 按用户 Predicate 筛出参与插入的列，拼成 `columnScript` 与 `insertSqlProperty`（`InsertBatchSomeColumn.java:103-109`）。
3. `SqlScriptUtils.convertForeach(insertSqlProperty, "list", null, ENTITY, COMMA)` 把实体属性段包成 `<foreach>` 多值循环（`InsertBatchSomeColumn.java:110`）。
4. 主键处理：`IdType.AUTO` 用 `Jdbc3KeyGenerator` 回写自增主键并 `SqlInjectionUtils.removeEscapeCharacter` 去转义；有序列则 `TableInfoHelper.genKeyGenerator`（`InsertBatchSomeColumn.java:114-127`）。
5. `sqlMethod.format(tableName, columnScript, valuesScript)` 拼最终 SQL，`createSqlSource` + `addInsertMappedStatement` 注册进 Configuration（`InsertBatchSomeColumn.java:129-131`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `methodName` | `insertBatchSomeColumn`（可在构造器改名） | `InsertBatchSomeColumn.java:75` |
| `predicate` | 默认不筛选（全字段）；常见如 `t -> !t.isLogicDelete()` 排除逻辑删除字段 | `InsertBatchSomeColumn.java:69` |
| 启用方式 | 用户自定义 SqlInjector 把这些方法 `addSqlInjector` 进 `AbstractSqlInjector.methodList` | core sql-injector 叶子 |

## 5. 错误与重试语义

- 注入期（启动时）失败直接抛异常，应用启动即失败，无重试。
- `InsertBatchSomeColumn` 注释明确警告：只在 MySQL 下测试过，自增主键未验证；非 MySQL 数据库不保证（`InsertBatchSomeColumn.java:39-41`）。
- 运行期 SQL 执行异常由 MyBatis 正常抛出，本叶子不做额外包装。

## 6. 并发细节

- 注入方法在 `SqlSessionFactory` 构建期一次性写入 Configuration，运行期只读，无线程问题。
- 无自建线程池/锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- extension 模块新增的 5 个 `AbstractMethod` 子类。

**Out-of-Scope（不在本仓库源码内）**

- `AbstractMethod`/`ISqlInjector`/`DefaultSqlInjector` 基类与默认 20 个方法——core-mapping 域 `sql-injector` 叶子。
- `SqlScriptUtils`/`SqlInjectionUtils`——core toolkit（core-foundation `toolkit` 叶子）。

## 8. 与相邻子系统交互

- 上游：用户自定义 `ISqlInjector`（core sql-injector）把本叶子方法加入注入列表。
- 本叶子 → 下游：生成的 `MappedStatement` 注册进 MyBatis `Configuration`，由 `mapper-runtime` 叶子代理执行。

## 9. 语言专项适配口径（JVM）

- **库型、启动期装配**：所有方法注入发生在应用启动 `SqlSessionFactory` 构建阶段，运行期零开销。
- **模板字符串生成 SQL**：用 `SqlScriptUtils.convertForeach` 生成 MyBatis动态 SQL，非注解 SQL。
- **依赖方向**：extension → core（`AbstractMethod`/`TableInfo`/`SqlScriptUtils`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 扩展注入方法架构图 | `injector-ext-architecture.html` | architecture | showcase |
| 方法注入流程图 | `injector-ext-workflow.html` | workflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 workflow：方法注入是"遍历 Mapper → 筛字段 → 拼 SQL → 注册 MS"的确定性步骤流程；省略 sequence 因与 core sql-injector 同构、信息重复。
