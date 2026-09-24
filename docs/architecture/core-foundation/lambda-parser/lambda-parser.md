# Lambda 解析器（lambda-parser）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开 `LambdaUtils` + `toolkit/support/` + `toolkit/reflect/` 如何把用户写的
> `User::getName` 这种 SFunction lambda 反解为"属性名/列名"，是 LambdaWrapper 强类型 API 的地基。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 可序列化函数接口 | `SFunction<T,R> extends Function, Serializable`，靠 Serializable 才能拿到 SerializedLambda | `support/SFunction.java:28` |
| Lambda 信息抽象 | `LambdaMeta` 接口：`getImplMethodName()`/`getInstantiatedClass()` 两方法 | `support/LambdaMeta.java:25` |
| 五套 LambdaMeta 实现 | Reflect(标准序列化)/IdeaProxy(IDE 调试代理)/Groovy(脚本代理)/Kotlin(KProperty)/Shadow(兜底序列化) | `support/ReflectLambdaMeta.java:29` 等 5 文件 |
| Lambda 解析主入口 | `extract(SFunction)`：按 Proxy/反射/Kotlin 多分支选择实现 | `toolkit/LambdaUtils.java:72` |
| writeReplace 缓存 | `ClassValue<WriteReplace>` 缓存每个 lambda 类的 writeReplace 方法查找结果 | `toolkit/LambdaUtils.java:50` |
| Kotlin 兼容 | 识别 KProperty/KFunction 捕获参数、合成方法名 `enclosing$getName` 还原 getter | `toolkit/LambdaUtils.java:125-215` |
| 列名缓存 | `COLUMN_CACHE_MAP` 按实体类缓存属性→ColumnCache（列名/sqlSelect） | `toolkit/LambdaUtils.java:43,279` |
| 泛型参数解析 | `TypeParameterResolver.resolveClassIndexedParameter` 解析父类泛型实参 | `reflect/TypeParameterResolver.java:32,49` |
| 泛型工具 | `GenericTypeUtils`/`IGenericTypeResolver`/`SpringReflectionHelper` 辅助泛型上溯 | `reflect/` 目录 |
| 序列化 Lambda 提取 | `SerializedLambda.extract(Serializable)` 兜底反序列化取元信息 | `support/SerializedLambda.java:36,50` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `SFunction<T,R>` | `SFunction.java:28` | 函数式接口，标记 lambda 可序列化，是 LambdaWrapper 入参类型 |
| `LambdaMeta` | `LambdaMeta.java:25` | 解析结果抽象：实现方法名 + 实例化类 |
| `ReflectLambdaMeta` | `ReflectLambdaMeta.java:29` | 标准 JVM 路径：从 SerializedLambda 的 instantiatedMethodType 反推 Class（`:46-48`） |
| `IdeaProxyLambdaMeta`/`GroovyLambdaMeta` | `support/` | 调试期/脚本期 lambda 是动态代理，单独解析 |
| `KotlinLambdaMeta` | `support/` | Kotlin 属性引用（KProperty）与 getter 适配方法名解析 |
| `ShadowLambdaMeta` | `ShadowLambdaMeta.java:28` | 反射失败时的兜底，纯靠序列化 |
| `LambdaUtils` | `LambdaUtils.java:38` | 解析编排 + 列缓存门面 |
| `ColumnCache` | `support/ColumnCache.java` | 单字段列缓存（列名 + sqlSelect + mapping） |
| `TypeParameterResolver` | `TypeParameterResolver.java:32` | 泛型变量→实际类型解析 |

## 3. 关键调用链

**`LambdaUtils.extract(SFunction)` 主链（`LambdaUtils.java:72`）**：

1. 入口先判 `func instanceof Proxy`（`:74`）：`MethodHandleProxies.isWrapperInstance` 为真 → `IdeaProxyLambdaMeta`（`:76`）；否则视为 Groovy 代理 → `GroovyLambdaMeta`（`:79`）。
2. 非代理走反射：`WRITE_REPLACE_CACHE.get(clazz)`（`:84`）取缓存的 writeReplace 方法；若 `method==null`，先 `findKotlinCallableInFields(func)`（`:88`）扫字段找 Kotlin 可调用引用，命中 → `KotlinLambdaMeta`（`:90`），否则 `ShadowLambdaMeta`（`:93`）。
3. 有 writeReplace 则 `writeReplace.method.invoke(func)`（`:95`）得到 `SerializedLambda`；若 `getCapturedArgCount()>0` 且捕获了 KProperty → `KotlinLambdaMeta.ofCapturedProperty`（`:100-102`）；若 implMethodName 含 `$` 且末段是 getter 名 → `KotlinLambdaMeta.ofGetterMethodName`（`:110-112`）；默认 → `ReflectLambdaMeta`（`:115`）。
4. 任一步抛异常 → catch 兜底 `ShadowLambdaMeta(SerializedLambda.extract(func))`（`:118`）。
5. 上层 LambdaWrapper 拿到 `LambdaMeta.getImplMethodName()`（如 `getName`）→ 转属性名 → `LambdaUtils.getColumnMap(clazz)`（`:311`）查 `COLUMN_CACHE_MAP` 得列名。

## 4. 配置项

本叶子无外部配置。缓存行为：

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `COLUMN_CACHE_MAP` | ConcurrentHashMap，实体类名→列缓存；启动时由 `installCache`（`:279`）随 TableInfo 构建灌入 | `LambdaUtils.java:43` |
| `WRITE_REPLACE_CACHE` | `ClassValue`，随类卸载回收，避免泄漏 lambda ClassLoader | `LambdaUtils.java:50` |
| 列名 key 归一化 | `formatKey` 转大写（`toUpperCase(ENGLISH)`，`:270`），支持首字母大写字段 | `LambdaUtils.java:270` |

## 5. 错误与重试语义

- `extract` 整体 `try/catch(Throwable)`（`:116`）：任何反射异常不向外抛，统一降级到 `ShadowLambdaMeta` 序列化路径，保证 LambdaWrapper 不崩。
- `writeReplace` 查找 `NoSuchMethodException` 缓存为 `WriteReplace.NONE`（`:58`），其他 Throwable 缓存错误对象（`:60`），避免重复反射。
- 无重试/退避；失败即静默降级。

## 6. 并发细节

- `COLUMN_CACHE_MAP`（`:43`）为 `ConcurrentHashMap`，`getColumnMap` 用 `computeIfAbsent`（`:312`）并发安全填充。
- `WRITE_REPLACE_CACHE`（`:50`）与 `KOTLIN_REFERENCE_CACHE`（`:133`）为 JDK `ClassValue`：JVM 保证同一 Class 只计算一次且线程安全，随 ClassLoader 卸载自动回收。
- 无线程池、无共享可变实例；解析方法本身无状态、线程安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `toolkit/LambdaUtils.java`、`toolkit/support/`（SFunction/LambdaMeta/五实现/SerializedLambda/ColumnCache 等）、`toolkit/reflect/`（TypeParameterResolver/GenericTypeUtils/SpringReflectionHelper）。

**Out-of-Scope（不在本仓库源码内）**
- `java.lang.invoke.SerializedLambda`/`MethodHandleProxies`（JDK 原生）。
- `kotlin.reflect.KProperty`/`KFunction`（Kotlin 标准库，core 仅按类名字符串匹配，不编译期依赖）。
- 消费 LambdaMeta 的 LambdaWrapper（conditions-wrapper 叶子）。

## 8. 与相邻子系统交互

- 上游：用户写 `LambdaQueryWrapper::select(User::getName)` → conditions-wrapper 叶子调 `LambdaUtils.extract`。
- 本叶子 → 下游：`getColumnMap` 依赖 table-metadata 叶子的 `TableInfoHelper.getTableInfo`（`:313`）拿列元数据；`ClassUtils.toClassConfident`（toolkit 叶子）加载类。
- 数据流：**SFunction lambda → extract 分支判定 → LambdaMeta → implMethodName→属性名→ColumnCache→列名**。

## 9. 语言专项适配口径（Java/JVM）

- **JVM 字节码机制**：依赖 `invokedynamic` 生成的 lambda 类的 `writeReplace`（Serializable 契约）产出 `SerializedLambda`，这是 JDK 原生 lambda 序列化机制，非自研字节码增强。
- **多语言兼容矩阵**：同一入口 `extract` 按运行时形态（Proxy/反射/Kotlin）分派 5 种 LambdaMeta，是"按类名字符串识别 Kotlin 而不硬依赖 Kotlin 编译器"的典型兼容手法（`KOTLIN_KPROPERTY` 常量 `:125`）。
- **并发原语**：`ClassValue`（JVM 级线程安全缓存）+ `ConcurrentHashMap`；无显式锁。
- **库型无 main**：纯工具库。
- **lombok**：`@Slf4j` 用于 ReflectLambdaMeta 日志。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Lambda 解析架构图 | `lambda-parser-architecture.html` | architecture | showcase |
| Lambda 解析调用时序 | `lambda-parser-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。
本叶子补 sequence 图：`extract` 内 Proxy→writeReplace→Kotlin 分支→兜底是清晰的多步判定时序，与第 3 节调用链互相印证。
未生成 dataflow/lifecycle/workflow：列缓存虽有"实体→列"流向但已被架构图覆盖；无单实体状态机、无多角色审批流程，按资源节省原则省略。
