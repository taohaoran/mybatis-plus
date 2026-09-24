# 数据库序列键生成（key-generators）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 序列键生成 SPI | 实现 core `IKeyGenerator`，`executeSql(序列名)` 返回取序列值 SQL，`dbType()` 标识方言 | `incrementer/OracleKeyGenerator.java:27` |
| 9 种数据库序列 | Oracle/Postgre/DB2/DM/Firebird/H2/Kingbase/Lealone/SapHana | `incrementer/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `IKeyGenerator` | core `incrementer/IKeyGenerator.java` | 序列取数 SPI（core id-generator 叶子） |
| `OracleKeyGenerator` | `incrementer/OracleKeyGenerator.java:27` | `SELECT 序列.NEXTVAL FROM DUAL` |
| 其余 8 个 `*KeyGenerator` | `incrementer/` | 各自方言的序列 nextval SQL |

## 3. 关键调用链

**链 1：序列主键取值**

1. 实体主键 `@KeySequence` 指定序列名，MyBatis 启动时 `TableInfoHelper.genKeyGenerator` 按 DbType 选对应 `IKeyGenerator`。
2. 插入前调用 `executeSql(incrementerName)` 生成 `SELECT xxx.NEXTVAL FROM DUAL`（OracleKeyGenerator.java:30-32），MyBatis 执行该 SQL 取回序列值回填主键。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 启用方式 | 实体 `@KeySequence(value="序列名", dbType=...)` | core annotation |

## 5. 错误与重试语义

- 序列不存在由数据库报错透传，无重试。

## 6. 并发细节

- 各 KeyGenerator 无状态、线程安全；序列取数由数据库保证原子性。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- 9 个数据库序列 SQL 生成器。

**Out-of-Scope（不在本仓库源码内）**

- `IKeyGenerator` SPI 与 `@KeySequence` 注解——core。
- 内置雪花 ID `DefaultIdentifierGenerator`/`IdWorker`——core id-generator 叶子。
- 数据库序列对象本身——外部系统。

## 8. 与相邻子系统交互

- 上游：`TableInfoHelper` 按 `@KeySequence` 选用本叶子生成器。
- 本叶子 → 下游：输出取序列 SQL 交 MyBatis KeyGenerator 执行。

## 9. 语言专项适配口径（JVM）

- **策略模式**：一种数据库一个无状态实现类，SQL 模板化。
- **依赖方向**：extension → core（IKeyGenerator）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 序列键生成架构图 | `key-generators-architecture.html` | architecture | showcase |

- JSON IR 源文件位于 `json/` 目录。
- **第二图省略**：本叶子是 9 个同构无状态 SQL 模板类，无独立时序/管道/状态机/流程语义，按资源节省原则省略第二图。
