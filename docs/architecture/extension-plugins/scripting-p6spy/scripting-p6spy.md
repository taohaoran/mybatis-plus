# 模板语言驱动与 p6spy 日志（scripting-p6spy）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Velocity 语言驱动 | `MybatisVelocityLanguageDriver` 继承 mybatis-velocity，覆写 `createParameterHandler` 换 MP 的 `MybatisParameterHandler` | `scripting/MybatisVelocityLanguageDriver.java` |
| FreeMarker 语言驱动 | `MybatisFreeMarkerLanguageDriver` 同上，换 MP 参数处理器 | `scripting/MybatisFreeMarkerLanguageDriver.java` |
| Thymeleaf 语言驱动 | `MybatisThymeleafLanguageDriver` 同上 | `scripting/MybatisThymeleafLanguageDriver.java` |
| p6spy 日志工厂 | `MybatisPlusLogFactory` 实现 p6spy `P6Factory`，注册 MP 事件监听器 | `p6spy/MybatisPlusLogFactory.java:30` |
| SQL 日志格式化 | `P6SpyLogger` 实现 `MessageFormattingStrategy`，输出耗时+规整 SQL | `p6spy/P6SpyLogger.java:27` |
| 事件监听 | `MybatisPlusLoggingEventListener` 单例，拦截批量执行等事件 | `p6spy/MybatisPlusLoggingEventListener.java:29` |
| 标准输出 appender | `StdoutLogger` 继承 p6spy StdoutLogger | `p6spy/StdoutLogger.java:24` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MybatisVelocityLanguageDriver` 等 3 个 | `scripting/` | 模板语言驱动，统一换 `MybatisParameterHandler` |
| `MybatisPlusLogFactory` | `p6spy/MybatisPlusLogFactory.java:30` | p6spy `P6Factory` SPI |
| `P6SpyLogger` | `p6spy/P6SpyLogger.java:27` | SQL 消息格式化策略 |
| `MybatisPlusLoggingEventListener` | `p6spy/MybatisPlusLoggingEventListener.java:29` | JdbcEventListener 单例 |

## 3. 关键调用链

**链 1：模板 SQL 参数处理替换**

1. MyBatis 用语言驱动解析 `<script>` 模板后，调 `createParameterHandler(ms, parameter, boundSql)`。
2. `MybatisVelocityLanguageDriver` 不返回父类默认参数处理器，而是 `new MybatisParameterHandler(ms, parameter, boundSql)`（`MybatisVelocityLanguageDriver.java` 覆写方法），让模板 SQL 也走 MP 的参数填充/主键回填逻辑。

**链 2：p6spy SQL 日志输出**

1. p6spy 通过 `P6Factory` SPI 加载 `MybatisPlusLogFactory`，`getJdbcEventListener()` 返回单例 `MybatisPlusLoggingEventListener`（`MybatisPlusLogFactory.java:38-40`）。
2. SQL 执行后 `P6SpyLogger.formatMessage(connectionId, now, elapsed, category, prepared, sql, url)` 输出 `Consume Time：X ms` + 规整空白后的 SQL（`P6SpyLogger.java:30-33`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 启用方式 | MyBatis `languageDriver` 配置或 Mapper `lang=` 指定模板驱动 | mybatis 配置 |
| p6spy 启用 | spy.properties 指定 `modulelist=...` 与 `logMessageFormat=...P6SpyLogger` | p6spy 配置（外部） |

## 5. 错误与重试语义

- 模板渲染异常由对应模板引擎抛出；p6spy 日志失败不影响主流程。

## 6. 并发细节

- `MybatisPlusLoggingEventListener` 单例，p6spy 事件回调线程安全。
- 语言驱动实例由 MyBatis 单例化。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- 3 个模板语言驱动 + 4 个 p6spy 日志类。

**Out-of-Scope（不在本仓库源码内）**

- Velocity/FreeMarker/Thymeleaf 引擎本身、mybatis-* 语言驱动——第三方依赖。
- p6spy 引擎本身——第三方依赖，不在本仓库源码内。
- `MybatisParameterHandler`——core `override` 包（core-mapping mapper-runtime 叶子）。

## 8. 与相邻子系统交互

- 上游：MyBatis Configuration 装配语言驱动；p6spy 通过 JDBC 代理拦截 SQL。
- 本叶子 → 下游：参数处理委托 core `MybatisParameterHandler`。

## 9. 语言专项适配口径（JVM）

- **SPI 扩展**：分别接入 mybatis 语言驱动 SPI 与 p6spy `P6Factory` SPI。
- **依赖方向**：extension → core + 模板引擎/p6spy（optional）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| scripting-p6spy 架构图 | `scripting-p6spy-architecture.html` | architecture | showcase |
| p6spy 日志链路时序 | `scripting-p6spy-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 第二图选用 sequence：SQL 执行→p6spy 代理→事件监听→格式化→输出的消息时序。
