# 主键生成器（id-generator）

> 本文是 `core-foundation` 域下的叶子子系统文档。域级总览见 `../core-foundation.md`。
> 本文展开主键 ID 生成体系：`IdentifierGenerator` SPI、雪花 `Sequence`、`IdWorker` 门面与 `SystemClock`。
> 数据库序列键生成（OracleKeyGenerator 等）在 extension 域，不在本叶子。
>
> 源码基准：mybatis-plus 分支 3.0，commit bf67d907。模块 `mybatis-plus-core`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| ID 生成 SPI | `IdentifierGenerator` 接口：`nextId(entity)` 抽象、`assignId` 默认判空、`nextUUID` 默认 32 位 UUID | `incrementer/IdentifierGenerator.java:30` |
| 默认雪花生成器 | `DefaultIdentifierGenerator`：持有 `Sequence`，`nextId` 委托雪花；`getInstance()` 共享单例 | `incrementer/DefaultIdentifierGenerator.java:30,80` |
| imadcn 生成器 | `ImadcnIdentifierGenerator`：另一种 ID 生成实现（可选） | `incrementer/ImadcnIdentifierGenerator.java` |
| 数据库序列键 SPI | `IKeyGenerator`：各库序列键生成扩展点（Oracle 等实现在 extension） | `incrementer/IKeyGenerator.java` |
| 雪花算法核心 | `Sequence`：64 位 ID = 时间戳(41)+datacenter(5)+worker(5)+序列(12)，`twepoch=1288834974657` | `toolkit/Sequence.java:35,48` |
| 机器位自动分配 | `getDatacenterId` 从 MAC 地址推导（`:133`），`getMaxWorkerId` 从 MAC+PID hash 推导（`:226`） | `toolkit/Sequence.java` |
| 时钟回拨处理 | `nextId` synchronized：回拨≤5ms 睡 2×偏移等待，>5ms 直接抛异常拒绝（`:173-188`） | `toolkit/Sequence.java:170` |
| ID 门面 | `IdWorker`：静态 `IDENTIFIER_GENERATOR`，`getId/getIdStr/getTimeId/get32UUID`，可 `setIdentifierGenerator` 替换 | `toolkit/IdWorker.java:33` |
| 高性能时钟 | `SystemClock.now()`：缓存毫秒避免 `System.currentTimeMillis` 系统调用开销 | `toolkit/SystemClock.java` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `IdentifierGenerator` | `IdentifierGenerator.java:30` | SPI 扩展点：业务可自定义主键生成（如接入 Redis/号段模式） |
| `DefaultIdentifierGenerator` | `DefaultIdentifierGenerator.java:30` | 默认实现，包雪花 `Sequence`；静态单例 `DefaultInstance.INSTANCE`（`:84`） |
| `Sequence` | `Sequence.java:35` | 雪花算法本体，`nextId()` synchronized，位段拼装 `:205-208` |
| `IdWorker` | `IdWorker.java:33` | 静态门面，持有 `IDENTIFIER_GENERATOR`（`:38`），默认委托 DefaultIdentifierGenerator 单例 |
| `SystemClock` | `SystemClock.java` | 系统时钟封装，`now()` 供 Sequence 取毫秒 |

## 3. 关键调用链

**插入时分配主键（`IdType.ASSIGN_ID`）**：

1. SQL 注入器在 insert 方法填充主键前，调 `IdWorker.getId(entity)`（`IdWorker.java:59`）→ `IDENTIFIER_GENERATOR.nextId(entity)`（`:60`）。
2. 默认委托 `DefaultIdentifierGenerator.getInstance().nextId(entity)`（`IdWorker.java:38` → `DefaultIdentifierGenerator.java:76`）→ `sequence.nextId()`（`:77`）。
3. `Sequence.nextId()`（`Sequence.java:170`，synchronized）：`timeGen()` 取当前毫秒（`:171`，经 SystemClock）→ 若 `timestamp < lastTimestamp` 处理时钟回拨（`:173-188`）→ 同毫秒则 `sequence=(sequence+1)&sequenceMask`（`:192`），溢出则 `tilNextMillis` 自旋等下一毫秒（`:195,211`）；跨毫秒则序列重置为 1~2 随机数（`:199`）→ 位段拼装返回（`:205-208`）。
4. 构造期 `Sequence(inetAddress)`（`:87`）从 MAC 推导 datacenterId（`:90,133`）与 workerId（`:91,226`）；容器环境 pid<10 时用随机数兜底 workerId（`:235-237`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `IdType.ASSIGN_ID` | 默认走 `DefaultIdentifierGenerator` 雪花（注解层面见 annotations 叶子） | `IdType.java:50` |
| workerId/dataCenterId | 未指定时从 MAC 自动推导；多实例须保证组合唯一 | `Sequence.java:113` |
| 固定兜底生成器 | 取不到 IP 时用固定 (1,1) 并 warn | `DefaultIdentifierGenerator.java:70-72` |
| twepoch | `1288834974657`（2010-11-04），确定后不可变 | `Sequence.java:48` |
| 自定义生成器 | `IdWorker.setIdentifierGenerator` 或 GlobalConfig 注入 | `IdWorker.java:114` |

## 5. 错误与重试语义

- **时钟回拨**：偏移 ≤5ms 时 `Thread.sleep(offset<<1)` 等待并重读时钟（`Sequence.java:176-178`）；回退后仍回拨或偏移 >5ms 直接抛 `RuntimeException`（`:180,186`），**拒绝生成 ID**，避免重复。
- workerId/datacenterId 越界构造时 `Assert.isFalse` 立即失败（`:114-117`）。
- 取不到网卡时降级 datacenterId=1（`:151`）并 warn，不中断。
- 无业务重试；时钟回拨是硬失败，由调用方（应用）处理 NTP 同步。

## 6. 并发细节

- `Sequence.nextId()` 用 `synchronized` 方法（`Sequence.java:170`）串行化同毫秒序列自增，保证单机内 ID 不重复。
- `lastTimestamp`/`sequence` 为实例字段（`:77,81`），由 synchronized 保护。
- `ThreadLocalRandom.current()`（`:199`）用于跨毫秒序列初始值，避免全局 Random 竞争。
- `DefaultIdentifierGenerator` 默认单例（`:84`），全应用共享一个 Sequence；`IdWorker.IDENTIFIER_GENERATOR` 为静态可变字段（`:38`），启动期 set 后运行期不再变。
- 无线程池；高并发下 synchronized 是单机瓶颈（雪花设计如此，靠位段容纳每毫秒 4096 个 ID）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `core/incrementer/`（IdentifierGenerator/Default/Imadcn/IKeyGenerator）、`toolkit/IdWorker`/`Sequence`/`SystemClock`。

**Out-of-Scope（不在本仓库源码内）**
- 数据库序列键实现（OracleKeyGenerator/PostgreKeyGenerator 等，extension/incrementer，分片 B）。
- 雪花位段标准参考（twitter-commons snowflake，第三方思想）。
- 真正调用 nextId 的 insert 填充逻辑（sql-injector 叶子的 Insert 方法）。

## 8. 与相邻子系统交互

- 上游：sql-injector 叶子的 `Insert`/`InsertBatchSomeColumn` 等方法在 insert SQL 参数填充时调 `IdWorker.getId`。
- 本叶子 → 下游：`IdWorker.nextUUID` 默认走 `IdWorker.get32UUID`（`IdentifierGenerator.java:57`）；`SystemClock` 封装 JVM 时钟。
- 数据流：**实体插入请求 → IdWorker 门面 → Sequence.synchronized 取号 → 位段拼装 long → 回填实体主键**。

## 9. 语言专项适配口径（Java/JVM）

- **并发原语**：`synchronized` 方法（Sequence.nextId）+ `ThreadLocalRandom`；无 Lock/CAS。
- **位运算**：雪花用 `<<`/`|`/`&` 掩码拼装 64 位 long（`:205-208`），是 JVM 经典位段设计。
- **网络/管理 MXBean**：`NetworkInterface.getHardwareAddress` 读 MAC、`ManagementFactory.getRuntimeMXBean().getName()` 读 PID（`:145,229`），容器环境降级随机（`:235`）。
- **库型无 main**：嵌入业务应用。
- **时钟**：`SystemClock` 用后台线程定期刷新毫秒缓存（避免每次 `currentTimeMillis` 系统调用），是 JVM 性能优化常见手法。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 主键生成架构图 | `id-generator-architecture.html` | architecture | showcase |
| 雪花生成器状态机 | `id-generator-lifecycle.html` | lifecycle | standard（降档） |

JSON IR 源文件位于 `json/` 目录。
本叶子补 lifecycle 图：`Sequence.nextId` 内部"同毫秒递增/跨毫秒重置/序列耗尽自旋/时钟回拨拒绝"是围绕单个生成器实例的状态迁移，符合 lifecycle 单实体状态机语义。
**降档披露**：lifecycle 图经多轮 showcase 校验未过，失败检查名为 `composition/label-route-clearance`（spinning↔gen 双向回边标签与 gen→reject 下行边标签间距不足 4px）；
已采取的修复：调整 labelDy/labelDx 将双向边标签左右错开，仍因三条下行边共享通道无法满足 showcase 4px 间距，按兜底策略降为 standard 渲染（render 成功，799KB）。
未生成 sequence/dataflow/workflow：调用链已在第 3 节文字描述，位段拼装为纯计算无外部管道，按资源节省原则省略。
