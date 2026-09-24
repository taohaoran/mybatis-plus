# 链式条件构造（chain-conditions）

> 本文是 `extension-plugins` 域下的叶子子系统文档。域级总览见 `../extension-plugins.md`。
> 本文只展开把 Wrapper 与 Mapper 执行绑在一起的链式包装；核心 Wrapper 条件拼接见 core-mapping `conditions-wrapper` 叶子。
>
> 源码基准：`mybatis-plus-extension`，分支 3.0，commit `bf67d907`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 链式包装抽象基类 | `AbstractChainWrapper` 实现 Compare/Func/Join/Nested 接口，所有条件方法代理给内部 `wrapperChildren` 并返回 `typedThis` 支持链式 | `conditions/AbstractChainWrapper.java:43` |
| 查询链式 | `QueryChainWrapper`/`LambdaQueryChainWrapper`/`ChainQuery`：条件写完直接 `one()/list()/count()/page()` | `conditions/query/` |
| 更新链式 | `UpdateChainWrapper`/`LambdaUpdateChainWrapper`/`ChainUpdate`：条件写完直接 `update()/remove()` | `conditions/update/` |
| 链式工厂 | `ChainWrappers.queryChain/lambdaQueryChain/updateChain/...` 静态工厂 | `toolkit/ChainWrappers.java` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AbstractChainWrapper<T,R,Children,Param>` | `conditions/AbstractChainWrapper.java:43` | 泛型自类型 `typedThis`；代理所有 where 方法到 `wrapperChildren` |
| `QueryChainWrapper` | `conditions/query/QueryChainWrapper.java` | 持有 BaseMapper，条件后直接触发查询 |
| `LambdaQueryChainWrapper` | `conditions/query/LambdaQueryChainWrapper.java` | Lambda 列名版本 |
| `ChainWrappers` | `toolkit/ChainWrappers.java` | 静态工厂入口 |

## 3. 关键调用链

**链 1：链式条件代理**

1. `AbstractChainWrapper.eq(condition, column, val)` 直接 `getWrapper().eq(...)` 后 `return typedThis`（`AbstractChainWrapper.java:85-88`）。
2. 所有 `eq/ne/gt/like/in/orderBy/groupBy/and/or` 都是同一代理模式（`AbstractChainWrapper.java:85-425`）。
3. 终态方法（如 `one()/list()`）在具体子类上调用 `baseMapper.selectXxx(getWrapper())` 触发执行。

**链 2：禁用 Wrapper 原生方法**

- `getSqlSegment/getEntity/getExpression/clear/clone` 等被覆写为抛 `can not use this method`，防止用户绕过链式直接操作内部 segment（`AbstractChainWrapper.java:427-479`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 启用方式 | `new QueryChainWrapper<>(baseMapper)` 或 `mapper.selectList()` 风格链式；Kotlin 用 `KtQueryChainWrapper` | 构造器 |

## 5. 错误与重试语义

- 误用被禁方法立即抛 `MybatisPlusException`。
- DB 异常透传，无重试。

## 6. 并发细节

- 链式对象一次请求内使用，无线程共享。
- 无锁/无线程池。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `conditions/` 包全部链式包装 + `ChainWrappers` 工厂。

**Out-of-Scope（不在本仓库源码内）**

- `AbstractWrapper`/`QueryWrapper`/`LambdaQueryWrapper` 条件拼接核心——core-mapping `conditions-wrapper` 叶子。
- Kotlin 链式包装 `KtQueryChainWrapper`——extension `kotlin` 包。

## 8. 与相邻子系统交互

- 上游：业务代码 / `IRepository`/`Db` 工厂创建链式包装。
- 本叶子 → 下游：持有 `BaseMapper`，终态方法触发 Mapper 执行。

## 9. 语言专项适配口径（JVM）

- **自类型泛型（CRTP）**：`Children extends AbstractChainWrapper<...>` 实现链式返回类型协变。
- **装饰器模式**：在 core Wrapper 外包一层"条件收集 + 终态执行"。
- **依赖方向**：extension → core（AbstractWrapper 等）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 链式条件架构图 | `chain-conditions-architecture.html` | architecture | showcase |

- JSON IR 源文件位于 `json/` 目录。
- **第二图省略**：本叶子纯代理 core Wrapper，无独立时序/管道/状态机/流程语义，与 `conditions-wrapper` 叶子信息重复，按资源节省原则省略第二图。
