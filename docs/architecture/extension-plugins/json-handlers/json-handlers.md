# JSON 类型处理器（json-handlers）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| JSON 类型处理器抽象基类 | `AbstractJsonTypeHandler<T>` 继承 MyBatis `BaseTypeHandler`，实现参数 set 与结果 get 的通用模板，子类只实现 `toJson/parse` | `handlers/AbstractJsonTypeHandler.java:37` |
| Jackson 实现 | `JacksonTypeHandler`（jackson 2.x） | `handlers/JacksonTypeHandler.java` |
| Jackson3 实现 | `Jackson3TypeHandler`（jackson 3.x） | `handlers/Jackson3TypeHandler.java` |
| Fastjson 实现 | `FastjsonTypeHandler` / `Fastjson2TypeHandler` | `handlers/FastjsonTypeHandler.java`、`Fastjson2TypeHandler.java` |
| Gson 实现 | `GsonTypeHandler` | `handlers/GsonTypeHandler.java` |
| Map 包装工厂 | `MybatisMapWrapperFactory`/`MybatisMapWrapper` 让 `Map` 结果支持点路径取值 | `handlers/MybatisMapWrapper.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AbstractJsonTypeHandler<T>` | `handlers/AbstractJsonTypeHandler.java:37` | 模板方法：`setNonNullParameter` 调 `toJson` 写字符串；`getNullableResult` 读字符串调 `parse` 反序列化 |
| `IJsonTypeHandler<T>` | core `handlers/IJsonTypeHandler` | JSON 处理器 SPI（toJson/parse 契约） |
| 各 `*TypeHandler` | `handlers/` | 具体 JSON 库序列化/反序列化实现 |

## 3. 关键调用链

**链 1：JSON 字段写入数据库**

1. MyBatis 预处理语句时调 `setNonNullParameter(ps, i, parameter, jdbcType)`（`AbstractJsonTypeHandler.java:74`）。
2. `ps.setString(i, toJson(parameter))`——`toJson` 由子类用对应 JSON 库实现。

**链 2：JSON 字段读取**

1. `getNullableResult(rs, columnName/columnIndex/cs)` 三处重载统一 `rs.getString`，空白返回 null，否则 `parse(json)`（`AbstractJsonTypeHandler.java:79-94`）。
2. `getFieldType()` 优先返回泛型 `genericType`（3.5.6 起支持字段泛型），否则退化 `type`（`AbstractJsonTypeHandler.java:96-98`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 启用方式 | 实体字段 `@TableField(typeHandler = JacksonTypeHandler.class)` 指定 | 注解在 core annotation |
| `genericType` | 由字段反射 `field.getGenericType()` 注入，支持泛型集合反序列化 | `AbstractJsonTypeHandler.java:68-71` |

## 5. 错误与重试语义

- JSON 序列化/反序列化异常由各 JSON 库抛出，TypeHandler 不额外包装。
- 空字符串结果集返回 null（`StringUtils.isBlank` 判断），不抛错。

## 6. 并发细节

- TypeHandler 实例由 MyBatis 单例化，必须线程安全；各 JSON 库的 ObjectMapper/Gson 实例通常静态共享，本身线程安全。
- 无自建线程池。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `AbstractJsonTypeHandler` 与 5 个 JSON 库实现、`MybatisMapWrapper(Factory)`。

**Out-of-Scope（不在本仓库源码内）**

- Jackson/Fastjson/Gson 库本身——第三方依赖，不在本仓库源码内。
- `BaseTypeHandler`——MyBatis 第三方依赖。

## 8. 与相邻子系统交互

- 上游：MyBatis 执行参数映射/结果映射时按 `@TableField.typeHandler` 选用。
- 本叶子 → 下游：写 DB 为 JSON 字符串列；读时反序列化为实体字段类型。

## 9. 语言专项适配口径（JVM）

- **策略 + 模板方法**：抽象基类定模板，子类换 JSON 库实现；多 JSON 库可选，按需引入依赖。
- **依赖方向**：extension → core（`IJsonTypeHandler`）+ 各 JSON 库（optional）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| JSON 处理器架构图 | `json-handlers-architecture.html` | architecture | showcase |

- JSON IR 源文件位于 `json/` 目录。
- **第二图省略**：本叶子是策略/模板方法的静态类结构，无独立时序交互、无数据管道、无状态机、无审批流程；写入/读取链路与 MyBatis TypeHandler 机制重复，按资源节省原则不补第二图（architecture 已表达组件拓扑）。
