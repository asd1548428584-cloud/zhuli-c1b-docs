# zhuli · 批次 C1-B 实施方案 v3 ＋ v2→v3 修订对照表（合订全文）

> **合订本：两份文档按顺序接在一起，文字未改。** 第一部分原文为 CRLF 行尾，合订时统一为 LF，
> 除此之外没有任何删改（可对照 `docs/` 下的原文件逐行比对）。
>
> | 部分 | 原文件 | 行数 | 讲什么 |
> |---|---|---|---|
> | 第一部分 | `批次C1-B_记忆系统基础设施实施方案.md` | 3638 | 实施方案全文：7 张表 / 40 列 / v16→v17 迁移 / 三档 FTS / Store 设计 / 测试矩阵 / 退出条件 |
> | 第二部分 | `批次C1-B_方案v2到v3修订对照表.md` | 410 | v2 → v3 的 32 项逐项对照（原章节 / 改了什么 / 为什么 / 验收证据）+ 仍需人工核对清单 |
>
> **背景**（给第一次读的人）：目标是把 Android 应用 `zhuli` 的长期记忆从旧表 `zhuli_fact`（16 列）
> 迁到新结构 `zhuli_memory_item`（40 列）＋ source / evidence / relation / revision 等表，数据库 16 → 17。
> 架构依据是 `批次C0.2_记忆系统架构设计最终版`（文中简称 **C0.2**）与 `批次C1-A_FTS4兼容性验证报告`
> （文中简称 **C1-A**）。
>
> **引用约定**：文中 `§x.y` 指第一部分（实施方案）的章节；`C0.2 §x.y` 指架构依据文档。
>
> **三套测试口径不能混用**：逻辑用例 **75** 条（编号数）／执行 **126** 次（双跑展开）／
> Gradle XML `tests` **457**（331 基线 + 126）。
>
> **状态**：C1-B **尚未开工**。生产代码零改动，`DB_VERSION` 仍为 16。这是方案，不是实施结果。

---

**—— 第一部分：实施方案 v3（3638 行）——**

# 批次 C1-B：记忆系统基础设施实施方案

> 依据：`docs/批次C0.2_记忆系统架构设计最终版.md`（下称 **C0.2**）与
> `docs/批次C1-A_FTS4兼容性验证报告.md`（下称 **C1-A**）。
> 本文**只制定实施方案**，不含任何生产代码改动。所有"将要做"都写成计划，不是既成事实。
>
> **修订 v2（2026-11-25）**：项目负责人已对 §18 的 D-6（`FactStatus` 状态投影）与 D-7
> （`createCoreTables` 拆分）作出正式决定，本文据此修订。决策原文、落实映射与差异清单见
> `docs/批次C1-B_方案决策修订记录.md`（下称 **修订记录 v2**）。
>
> **修订 v3（2026-11-25）**：定点修复 **7 项 P0 + 8 项 P1 + 5 项验收细节**，外加 **12 项自查追加**
> （回读全文时发现的旧口径残留与算式错误），合计 **32 项**。不改变 C0.2 架构、不撤销 D-6/D-7、
> 不扩大生产代码范围。逐项对照表（原章节 / 修改内容 / 原因 / 验收证据）与"仍需人工核对清单"见
> `docs/批次C1-B_方案v2到v3修订对照表.md`（下称 **修订对照表 v3**）。
>
> **v2 → v3 的数字变化（必须区分三种口径，见 §10.11）**：
>
> | 口径 | v2 | v3 | 变化原因 |
> |---|---|---|---|
> | **逻辑用例数**（编号数） | 65 | **75** | +10：`T-M46`（`sqlite_sequence`）、`T-S21`–`T-S24`（FTS 写入职责）、`T-N01`–`T-N05`（并发） |
> | **执行次数**（Gradle XML `tests`） | 396（**错误**：把逻辑数当执行数） | **457** | 331 基线 + 126 次执行；126 由"哪些用例需要双跑"逐条算出 |
> | 修改生产文件 | 4 | **4**（不变） | v3 不新增生产文件改动 |
> | 测试代码行数 | ≤2,400 | **≤2,800** | 10 条新用例 + 白名单修正 |
>
> v3 **撤销** v2 的两个结论：① §5.12 的"WAL 下 checkpoint 后继续"（在 helper 事务内不可行，改为断言拒绝）；
> ② §14.1 的"`onUpgrade` 内分批提交"（Android 嵌套事务不会真的 COMMIT，改为"分批处理、单事务提交"）。

---

## 0. 结论摘要与阅读指引

### 0.1 一句话

C1-B 用 **7 个新文件 + 4 个改动文件**（生产），把 C0.2 的 7 张表、v16→v17 迁移、三档 FTS 与唯一写入口落地，并用 **75 条新用例（逻辑）/ 126 次执行 + 10 项设备验收**兜住；`MemoryRepo` 的 27 个公开签名一个不改。

> **为什么是两个数**：`75` 是编号数（一条用例一个编号，便于追溯 C0.2 / D-6 / D-7 / v3）；`126` 是 Gradle 实际执行数（其中 51 条要在 `TRIGGER` 与 `APP_SYNC` 两档各跑一遍）。v2 把这两个数混成了"65"，又用 331+65=396 当目标，这是 v3 要修的 P1-6（§10.11 有逐组明细）。

> v1 的"7 + 2"是 D-6/D-7 决定前的口径。D-6 要求扩展 `FactStatus`（`MemoryTypes.kt`）并让 `MemoryRanker.conflicts` 支持 CANDIDATE，故生产改动文件由 2 个增至 4 个：`InboxDatabase.kt`（D-7 拆分）、`MemoryRepo.kt`（投影重写）、`MemoryTypes.kt`（状态扩展）、`MemoryRanker.kt`（冲突过滤）。65 = 原 47 + D-6 的 12 + D-7 的 6。


### 0.2 五个子阶段与依赖

```
C1-B1 结构  ──┬─→ C1-B2 迁移 ──┐
              │                 ├─→ C1-B5 回归
              └─→ C1-B3 FTS ────┤
                      │         │
                      └─→ C1-B4 Store ──┘
```

| 阶段 | 内容 | 可否与前序并行 | 关键产物 |
|---|---|---|---|
| C1-B1 | 7 张表 DDL + `MemorySchema` + `MemoryText` + `FactStatus` 扩展 + 建表拆分 | — | 建表可跑、纯函数全绿 |
| C1-B2 | `MemoryMigrator` + 事务接入 | 与 B3 并行（互不依赖） | 真实副本迁移 10/10 不变量 |
| C1-B3 | `detectFtsMode` 功能性探测 + 两条同步路径 | 与 B2 并行 | 三档结论可复现 |
| C1-B4 | `MemoryStore` + `MemoryRepo` 投影重写 + `conflicts` 新语义 | 依赖 B1/B3 | 27 个签名不变、测试全绿 |
| C1-B5 | 回归、设备验收、报告 | 最后 | D1–D10 + 退出条件核对表 |

**B2 与 B3 可以并行**，因为迁移走"先灌数据、后建触发器"的一次性路径（C0.2 §2.6），不依赖运行时 `ftsMode`；B3 只在探测时用临时对象。B4 必须等 B3，因为它要按 `ftsMode` 决定写路径分支。

**D-6 的状态扩展必须在 B1 就位**：`MemoryItem.kt` 的 `MemoryStatus`（六态）与 `MemoryTypes.kt` 的 `FactStatus`（六态 + `UNKNOWN`）是两套模型，投影函数只存在于 `MemoryRepo`，但两者都要在 B1 编译通过——B2 的迁移映射与 B4 的投影都依赖它们。D-7 的建表拆分同属 B1，因为它改的是 `InboxDatabase.onCreate/onOpen/onUpgrade`，B2 的迁移入口挂在这条链上。

### 0.3 本方案与 C0.2 的关系

- **不改架构**：7 张表、40 列、六态、R0–R9、半开区间、`LB`/`UB`、恢复状态机、四阶段 SECRET，全部照抄。
- **只补实现细节**：C0.2 是"设计级"文档，有些地方到不了代码（例如 `MemorySchema` 怎么拿到 `SharedPreferences`）。凡是补细节的地方，本文用 **`[实现补充]`** 明确标注。
- **冲突已定**：C0.2 与现有生产代码有 10 处对不上（§3.4），其中 X-1 / X-2 / X-3 已由项目负责人决定（D-7 / D-6 / D-6 第 7 条），落实方案分别在 §4.8、§7.12、§7.12.5。其余 7 处仍按本文建议执行，它们不影响架构。


### 0.4 本文的四种陈述类型（沿用 C0.2 §0.4，务必区分）

| 标记 | 含义 | 例子 |
|---|---|---|
| **已核查** | 本轮在仓库/设备上直接读出来或跑出来的 | `DB_VERSION = 16`（`InboxDatabase.kt:50`） |
| **沿用** | 来自 C0.2/C1-A 已验证的结论，本轮未复跑 | 半开区间在 X 点不双命中（C0.2 探针） |
| **计划** | 尚未执行，C1-B 要做的 | `MemoryStore.softDelete()` 的签名 |
| **未验证** | 明确没有证据 | Android 16 真机上的 `trusted_schema` 默认值 |

**本文不把"计划"写成"已通过"。** §16 的退出条件表里，每一项的证据列要么指到一个具体命令，要么写"未执行"。

---

## 1. 当前仓库真实基线（已核查）

### 1.1 Git（P0-1，v3 统一基线）

v2 把 HEAD 写成 `6a66c1e`（C1-A 交付），那是 v1/v2 写作时的 HEAD。**v3 写作时 HEAD 已经是 `ba2c2f0`**（v2 的提交）。"当前 HEAD"是会动的，所以不能只写一个哈希——要写清"哪个哈希是什么用途"。

**六个基线，各有用途，不可混用**：

| 名字 | 提交 | 是什么 | 谁在用 |
|---|---|---|---|
| `BASE_B1` | **`3597904`** | B1 交付（`fix(agent): batch B1 debug test harness and resume budget fix`） | C0.2 §11.4 的 313 例回归基线来源 |
| `BASE_C1A` | **`6a66c1e`** | C1-A 交付（`test(c1a): verify FTS4 + trigger sync`），+18 例 | 331 例基线的另一半 |
| `BASE_C1B_V1` | **`90088f8`** | C1-B 方案 v1（47 条新增、378 目标） | 历史对照 |
| `BASE_C1B_V2` | **`ba2c2f0`** | C1-B 方案 v2（D-6/D-7 落实，65 条、396 目标） | 被 v3 修订的对象 |
| **`BASE_IMPL`** | **v3 提交的哈希**（提交后产生） | **C1-B 开工基线** | `git diff BASE_IMPL..HEAD` 的起点 |
| `HEAD_NOW` | `ba2c2f0`（v3 提交前） | 写作时的 HEAD | 只用于"v3 相对 v2 改了什么" |

**tag 的两个形态必须分清**（v2 在交付报告里把两者并列成"`202f2c5` / `394023c`"，容易误读为同一层级）：

| tag | 类型 | `git rev-parse` 给什么 | 实际指向的 commit | 能否移动 |
|---|---|---|---|---|
| `batch-a` | **annotated tag（附注标签）** | tag 对象 **`202f2c5`** | **`9555bab`**（`git cat-file -p batch-a` 的 `object` 行） | **不能** |
| `v16-baseline-20261124` | **轻量标签（lightweight）** | commit **`394023c`**（就是它自己） | `394023c` | **不能** |

一句话：`batch-a` 的 `202f2c5` 是**标签对象**的哈希，不是提交；`v16-baseline-20261124` 的 `394023c` 就是提交。用 `git rev-list -n1 <tag>` 永远拿到提交，用 `git rev-parse <tag>` 在附注标签上拿到的是标签对象——这就是 v2 那张表容易读错的原因。

**工作区与分支**：

| 项 | 值 | 核查方式 |
|---|---|---|
| 分支 | `master` | `git rev-parse --abbrev-ref HEAD` |
| 工作区 | 干净，只有未跟踪的 `Temp/` | `git status --porcelain` → `?? Temp/` |
| 线性历史 | C0 → C0.1 → C0.2 → C1-A → C1-B v1 → C1-B v2 → C1-B v3，无 merge、无 amend、无 rebase | `git log --graph --oneline` |
| 已推送 | **否**（本地仓库） | `git status -sb` 无 upstream |

提交链（近 7 个，v3 写作时）：

```
ba2c2f0 docs(c1b): apply D-6/D-7 decisions to the implementation plan
90088f8 docs(c1b): implementation plan for memory infrastructure, 5 sub-stages + 10 code conflicts
6a66c1e test(c1a): verify FTS4 + trigger sync on Android SQLiteDatabase
a0e1d52 docs(c0.2): fix four data-lying defects in memory design; add three engineering gates
ace7cbc docs(c0.1): add memory architecture design revision with rehearsal-verified migration
3106be9 docs(c0): fix dedupe_key bucketing for STATE memory type
b18e254 docs: add batch C0 memory architecture design report
```

> `batch-a`（`9555bab`）与 `v16-baseline-20261124`（`394023c`）都**未被移动**。C0/C0.1/C0.2/C1-A/C1-B 全部是 `batch-a` 之后的**新增提交**，没有改写历史。
>
> **`BASE_IMPL` 为什么不能写死**：v3 的提交哈希要提交后才知道。§16 退出条件的第 4 条因此改为"`git diff BASE_IMPL..HEAD`"，其中 `BASE_IMPL` 由 C1-B5 报告的第一节填写（开工时 `git rev-parse HEAD` 的输出）。写成 `3597904..HEAD` 会把 C0/C0.1/C0.2/C1-A 四个提交也算进 diff 范围，导致"白名单"永远对不上。

### 1.2 数据库

| 项 | 值 | 位置 |
|---|---|---|
| `DB_VERSION` | **16**（未改） | `InboxDatabase.kt:50` |
| 数据库名 | `zhuli_inbox.db` | `InboxDatabase.kt` |
| `journal_mode` | `delete`（应用库，C0.2 实测）；C1-A 的新建临时库默认 `truncate`。两者都是回滚日志模式，**都不是 WAL** | C0.2 §8.10 / C1-A §2.1 |
| `secure_delete` | 1 | C0.2 §6.3（V12） |
| SQLite（模拟器） | 3.42.0（进程内 `sqlite_version()` 与设备 CLI 一致） | C1-A §2.1 |
| minSdk | 28 | `app/build.gradle.kts:80` |
| compileSdk / targetSdk | 36 / 36 | 同上 |

现有与记忆相关的表（v16）：

| 表 | 列数 | 说明 |
|---|---|---|
| `zhuli_fact` | 16 | 旧记忆主体，v17 要迁走并改名为 `zhuli_fact_v1` |
| `zhuli_entity` | 7 | 实体索引（`UNIQUE(kind,norm)`） |
| `zhuli_fact_vec` | 4 | 向量，`fact_id` 主键 |
| `zhuli_vec_queue` | 4 | 向量索引队列 |

> `zhuli_fact_vec` 与 `zhuli_vec_queue` **不改结构**，只改 `fact_id` 的指代（旧 `fact.id` → 新 `item.id`）。ID 保真使这一步是零成本的（C0.2 §8.6）。

真实库副本（`Temp/c02/zhuli_inbox_v16_real.db`，274,432 字节）：

```
user_version = 16
zhuli_fact   = 2 行（ACTIVE 1 / SUPERSEDED 1）
zhuli_fact_vec = 1 行
```

预演库规模（供 C1-B2 复跑对照）：

| 库 | 行数 | 体积 | 状态分布 |
|---|---|---|---|
| `v16_edge.db` | 11 | 0.26 MB | ACTIVE 7 / DELETED 1 / SUPERSEDED 2 / UNCONFIRMED 1 |
| `v16_5k.db` | 5,002 | 1.12 MB | ACTIVE 3,682 / DELETED 453 / SUPERSEDED 391 / UNCONFIRMED 476 |
| `v16_20k.db` | 20,002 | 3.72 MB | ACTIVE 14,670 / DELETED 1,763 / SUPERSEDED 1,759 / UNCONFIRMED 1,810 |

### 1.3 生产代码现状

`com/zhuli/memory/` **8 个文件**、1,556 行（v1 误写"7 个文件"，逐行相加 485+154+241+231+148+124+115+58 = 1,556，故为 8 个）：

| 文件 | 行数 | 在 C1-B 的角色 |
|---|---|---|
| `MemoryRepo.kt` | **485** | **重写为投影层**（唯一的大 diff） |
| `MemoryTypes.kt` | 154 | **修改**（D-6）：`FactStatus` 加 `EXPIRED` / `REJECTED` / `UNKNOWN`，`of()` 不再 fallback 到 `ACTIVE` |
| `MemoryExtractor.kt` | 241 | 不动（纯函数） |
| `MemoryIndexer.kt` | 231 | 不动（只用 `MemoryRepo` 的向量方法） |
| `MemoryBackfill.kt` | 148 | 不动（只用 `ingest`；KDoc 里"低置信度进 UNCONFIRMED"仍成立） |
| `MemoryRanker.kt` | 124 | **修改一处**（D-6 第 7 条）：`conflicts` 的过滤集合从 `ACTIVE` 扩到 `ACTIVE` + `UNCONFIRMED` |
| `MemoryEmbedding.kt` | 115 | 不动 |
| `GemmaEmbedding.kt` | 58 | 不动 |

> `MemoryRanker.kt` 的改动是**一个过滤条件 + KDoc**（约 +8 / −3 行），不新增函数、不改签名。它是否算"修改生产文件"：本文按**改动落地的文件**计数，故生产文件从 2 个变 3 个（`InboxDatabase.kt` / `MemoryRepo.kt` / `MemoryTypes.kt`），`MemoryRanker.kt` 是**第 4 个落地文件**。§8.2 的表格按"必须改的文件"列 4 行，`git diff --name-only` 的期望值同步改为 4。


`MemoryRepo` 的公开成员共 **27 个**（1 个 `@Volatile var` + 26 个方法），是 C1-B 必须原样保留的契约面：

```
onIndexRequested / ingest / correct / deleteFact / byId / unconfirmed / confirmFact /
activeOf / active / activeCount / history / search / conflicts / resolveConflict /
putVector / markVectorIndexed / markVectorFailed / dropVector / pendingVectorFacts /
pendingVectorCount / waitingVectorCount / failedVectorCount / indexedSpaces /
vectorCount / enqueueAllActive / vectorText / hybridSearch(suspend)
```

调用方（已核查，`grep "MemoryRepo("`）：

| 调用点 | 用途 |
|---|---|
| `agent/ZhuliAgentRuntime.kt:53,58` | 构造实例、注入 `MemoryIndexer` |
| `agent/ZhuliAgentTools.kt:78,166,186,203` | `hybridSearch` / `ingest` / `correct` |
| `core/ExtractPipeline.kt:105` | `ingest`（通知流水线） |
| `memory/MemoryBackfill.kt:106` | `ingest(activateThreshold=…)` |
| `memory/MemoryIndexer.kt` | 向量 7 个方法 + `activeCount` |
| `test/java/com/zhuli/memory/MemorySqliteTest.kt:43,383` | 单测 |

`bridge/ZhuliMemoryPage.kt` 只用 `Fact` / `FactHit` / `MemoryBackfill` / `MemoryIndexer`，不直接构造 `MemoryRepo`。

### 1.4 测试基建现状

| 项 | 值 |
|---|---|
| `app/src/test` 文件数 | 40 |
| `app/src/test` 中 `@Test` 注解数 | 322 |
| **实际运行用例数（本轮实跑）** | **331**，0 失败 / 0 错误 / 0 跳过，40 个套件 |
| `app/src/androidTest` 文件数 | 12（含 C1-A 新增的 `MemoryFtsInstrumentedTest.kt`） |
| 记忆相关单测 | `MemorySqliteTest`（15 例）、`MemoryEmbeddingTest`、`MemoryExtractorTest`、`MemoryRankerTest`、`MemoryFtsRobolectricTest` |
| Robolectric 版本 | 4.15.1 |
| Robolectric 测试应用 | `com.zhuli.testutil.PlainTestApp`（`@Config(application=…)`） |
| 真实 SQLite | Robolectric `NATIVE` 模式 = SQLite **3.32.2**；`LEGACY` = 3.8.7（C1-A §2.2） |
| 真实库副本测试能力 | `MemorySqliteTest` 用 `ctx.deleteDatabase` + `InboxDatabase(ctx)` 建**新库**；**没有**加载 `.db` 副本的现成机制（C1-B2 要新建） |
| androidTest runner | `androidx.test.runner.AndroidJUnitRunner`，包名 `lastchat.rikkafork.cocolal.dev.test` |

**结论**：L2（真实 SQLite）走 Robolectric 的 `NATIVE` 模式是可行的，但 C1-A 已证明它**不复现** `trusted_schema=OFF` 的失败（C1-A §5）。所以：
- 迁移、状态机、唯一索引、scope 这类**纯 SQL 语义** → Robolectric 可信；
- FTS 触发器这类**依赖 ROM 配置**的行为 → 只认 `androidTest`；
- 迁移耗时 → 只认设备（C0.2 §9.6，**禁止**把桌面 Python 的耗时当设备耗时）。

### 1.5 基线数字汇总（后面反复引用）

**三种口径，v3 起必须分开写**（P1-6）：

| 口径 | 含义 | 谁在用 |
|---|---|---|
| **逻辑用例数** | 一个编号 = 一条用例（`T-M01` 是 1 条，不管它跑几遍） | C0.2 §11.2 的 57、本方案的 75 |
| **执行次数** | 同一编号在两种 `ftsMode` 下各跑一次 = 2 次 | `MemoryStore(db, ftsModeOverride=…)` 的双跑设计 |
| **XML `tests` 计数** | Gradle 报告 `<testsuite tests="N">` 的 N，**等于执行次数** | §16 退出条件、CI 门禁 |

| 名字 | 值 | 口径 |
|---|---|---|
| `BASE_TESTS` | **331** | XML（`testDevDebugUnitTest` 实跑：331 例 / 0 失败 / 0 跳过） |
| `BASE_TESTS_B1` | 313 | XML（C0.2 §11.4 记录） |
| `C1A_EXECUTED` | 18 = 9 × 2 个 `SQLiteMode` | XML（`MemoryFtsRobolectricTest` 两个类） |
| `BASE_B1_COMMIT` | `3597904` | — |
| `C0_CASES_TOTAL` | 57 | 逻辑 |
| `C1_CASES_IN_C1` | 54（扣 `T-D32` / `T-A34` / `T-A35`） | 逻辑 |
| `C1_CASES_DONE_IN_C1A` | 7（`T-F50`–`T-F56`） | 逻辑 |
| `C1B_ARCH_CASES` | **47** = 54 − 7 | 逻辑 |
| `C1B_D6_CASES` | **12**（`T-X01`–`T-X12`） | 逻辑 |
| `C1B_D7_CASES` | **6**（`T-M40`–`T-M45`） | 逻辑 |
| `C1B_V3_CASES` | **10**（`T-M46` / `T-S21`–`T-S24` / `T-N01`–`T-N05`） | 逻辑 |
| **`C1B_LOGICAL_CASES`** | **75** = 47 + 12 + 6 + 10 | **逻辑** |
| **`C1B_EXECUTED`** | **126**（逐组见 §10.11） | **执行** |
| **`C1B_TARGET_TESTS`** | ≥ 331 + 126 = **457** | **XML** |

> **为什么目标从 396 变成 457**：v2 的 396 = 331 + 65，把"逻辑用例数 65"直接加到了"XML 计数 331"上。但 65 条里有 51 条要双跑（`TRIGGER` + `APP_SYNC`），实际新增执行是 126 次。这不是范围扩张，是**口径修正**——如果只要求"XML 增加 65"，那么双跑就变成了"同一批用例跑两遍但只算一遍"，CI 会显示 396 而实际执行 457，门禁失去意义。
>
> **为什么不用 C0.2 的 `313 + 54 = 367`**：313 是 C1-A **之前**的数。C1-A 已交付 18 次执行（G 组 7 条 × 2 个 `SQLiteMode` + 环境诊断 + 负向对照按类计），重复计入会虚增要求。


---

## 2. C0.2 架构要求摘录与实现映射

这一节的作用：**每一条已冻结的规则，指到具体文件与函数**。C0.2 有的条款本文不复述全文，只摘"实现时必须照着做的那一句"。

### 2.1 七张表 → 落在哪个文件

| 表 | DDL 定义 | 建表入口 | 首次创建时机 |
|---|---|---|---|
| `zhuli_memory_item` | `MemorySchema.kt` 的 `DDL_ITEM` | `MemorySchema.createAll(db, withUnique=false)` | `onCreate` / `onUpgrade(v17)` |
| `zhuli_memory_source` | `DDL_SOURCE` | 同上 | 同上 |
| `zhuli_memory_evidence` | `DDL_EVIDENCE` | 同上 | 同上 |
| `zhuli_memory_relation` | `DDL_RELATION` | 同上 | 同上 |
| `zhuli_memory_revision` | `DDL_REVISION` | 同上 | 同上 |
| `zhuli_memory_entity` | `DDL_ENTITY` | 同上 | 同上 |
| `zhuli_memory_fts` | `DDL_FTS` + 3 个 `DDL_TRIGGER_*` | `MemorySchema.createFts(db)` | **探测通过后**（`ftsMode != NONE`） |

`zhuli_memory_tombstone` / `usage` / `ingest` **不在 C1 建**（C0.2 §2.1）。

### 2.2 40 列 item 的六组语义（实现时按组处理，不要逐列散着写）

| 组 | 列 | 写入方 | 不变量 |
|---|---|---|---|
| 身份 | `id` | `AUTOINCREMENT` | 迁移保真（§8.6） |
| 分类 | `memory_type` / `subject` / `subject_key` / `predicate` / `qualifier_key` / `cardinality` | `MemoryStore` | `subject_key = norm(subject)`；`qualifier_key` 默认 `''` |
| 内容 | `value` / `content` / `normalized_content` / `dedupe_key` / `slot_hash` / `search_text` / `legacy_kind` / `payload_json` | `MemoryText` + `MemoryStore` | `dedupe_key` 32 hex；`slot_hash` 与 value 无关（C0.2 §2.2） |
| 状态 | `status` / `confidence` / `confidence_basis` / `superseded_by` / `revision` / `extractor` | `MemoryPolicy` + `MemoryStore` | 六态；`revision` 每次写 +1 |
| **时间** | `event_start` `event_end` `valid_from` `valid_until` `observed_at` `recorded_at` `updated_at` `last_confirmed_at` `invalidated_at` `deleted_at` `time_confidence` | `MemoryStore`（唯一） | 见 §2.3 |
| 权限与治理 | `scope_type` / `scope_id` / `sensitivity` / `source_id` / `times_*` / `last_used_at` / `pre_delete_status` | `MemoryGuard` + `MemoryStore` | scope 合法性（§2.6）；`source_id` = PRIMARY 冗余（§2.7） |

### 2.3 时间规则的三处硬约束（任何一处写错都会让数据说谎）

C0.2 §4.9 给了 `LB` / `UB` 两个**唯一**的派生表达式。C1-B 把它们写成 `MemoryText` 里的常量，**全仓库只此一份**：

```kotlin
// MemoryText.kt —— 唯一来源，任何地方都不许就地另写一个
const val LB_SQL = "CASE WHEN valid_from > 0 AND time_confidence <> 'NONE' " +
                   "THEN valid_from ELSE observed_at END"
const val UB_SQL = "valid_until"          // 不设 time_confidence 门槛
```

| # | 约束 | 落地 | 违反后果 |
|---|---|---|---|
| 1 | `valid_from` 绝不因"不知道"写 `now` | `MemoryStore.insert` 的 `now` 参数**不参与** `valid_from`；只有显式 `worldFrom` 才写 | 伪造事实起点 |
| 2 | 纠正**继承** `valid_from` 与 `time_confidence` | `MemoryStore.correct()` 从 `old` 复制，**不**用 `now` | 一次纠正把系统时间洗成用户时间（C0.2 §4.2 注） |
| 3 | 替代/过期**必须**同时写 `invalidated_at`，且此后不再改 | `MemoryStore.supersede()` / `expire()`；`softDelete` 只在 `=0` 时写 | 信念线与世界线混用 |
| 4 | 迁移行 `time_confidence` **一律 `NONE`**、`valid_until` **一律 0** | `MemoryMigrator` 的常量，不按条件分支 | 伪造时间精度（C0.2 缺陷 #1） |

`CURRENT` / `WORLD_ASOF` / `BELIEF_ASOF` 三个谓词（C0.2 §4.3，逐字实现）：

```sql
-- CURRENT
status='ACTIVE' AND (LB) <= :now AND ((UB)=0 OR :now < (UB))

-- WORLD_ASOF(:at)
status IN ('ACTIVE','SUPERSEDED','EXPIRED')
  AND (LB) <= :at AND ((UB)=0 OR :at < (UB))
  AND NOT EXISTS (SELECT 1 FROM zhuli_memory_relation r
                  WHERE r.to_id = i.id AND r.relation='CORRECTS')

-- BELIEF_ASOF(:at)
status IN ('ACTIVE','SUPERSEDED','EXPIRED')
  AND observed_at <= :at AND (invalidated_at=0 OR :at < invalidated_at)
```

`WORLD_ASOF` **不得**加 `confidence_basis` 或 `sensitivity` 之外的状态过滤——`DELETED`/`REJECTED` 不是历史值，是"用户不要了"。

### 2.4 冲突决策树 R0–R9 的 C1 落点

| 规则 | C1 是否实现 | 落在 | 说明 |
|---|---|---|---|
| R0 合法性 | ✅ | `MemoryGuard.check()` → `MemoryPolicy` | scope 合法、非 SECRET、非空、长度、**时间自洽（清零 + 强制 CANDIDATE）** |
| R1 墓碑 | ❌ | — | 依赖 C3 的 `tombstone` 表；C1 无硬删入口 |
| R2 `dedupe_key` 命中 ACTIVE/CANDIDATE | ✅ | `MemoryStore.insert()` | 追加 `SUPPORTING` 证据 + 强化，不新增行 |
| R3 命中 DELETED/REJECTED | ✅ | 同上 | 用户显式陈述 → 新建 ACTIVE + `RESTORES`；否则抑制 |
| R4 / R4a / R4b / R4c | ✅ | `MemoryPolicy.decide()` | R4a 用 `UB > 0 AND UB <= now`（**不是** `valid_until>0 AND valid_until<now` 的自写版） |
| R5 `cardinality=MULTI` | ✅ | 同上 | 短路，直接 ACTIVE |
| R6 `intent=CORRECT` | ✅ | `MemoryStore.correct()` 直接走 §4.4 | C1 只有 `correct_fact` 工具这一条入口；句式识别在 C2 |
| R7 / R8 / R9 | ✅ | `MemoryPolicy.decide()` | 见下表 |
| 事务性约束 | ✅ | `MemoryStore` 的每个写方法单事务 | C0.2 §6.5 |

`R7`/`R8`/`R9` 的实现必须逐字照抄 C0.2 §4.5 的分支顺序（rank 相等时先看"双方 USER_STATED + 双方 `time_confidence≠NONE` + Δt>24h"）：

```kotlin
// MemoryPolicy.decide() —— 伪码，实际实现见 C1-B4
when {
    newRank > oldRank && old.basis == USER_CONFIRMED -> Candidate(conflictWith = old)
    newRank > oldRank                               -> Supersede(old)
    newRank == oldRank &&
        new.basis == USER_STATED && old.basis == USER_STATED &&
        new.timeConfidence != NONE && old.timeConfidence != NONE &&
        new.eventStart - old.eventStart > DAY_MS      -> Supersede(old)
    newRank == oldRank                              -> Candidate(conflictWith = old)
    else                                            -> Candidate(conflictWith = old)  // R9 永不替代
}
```

> `Δt` 用的是**世界时间**（`valid_from`），不是 `observed_at`。C0.2 §4.5 的"Δt>24h"写在"双方 `time_confidence≠NONE`"后面，语义是"两个可信的世界时间点相隔超过一天"。实现时若拿 `observed_at` 算，取件码那个反例（两条相隔 3 天的包裹）会被判成"变更"——因为 `observed_at` 也是相隔 3 天。**这一处必须写测试盯住**（复用 `T-S15`）。

### 2.5 删除与恢复

| 动作 | 入口 | 事务 | 关键点 |
|---|---|---|---|
| 软删 | `MemoryStore.softDelete(id, now): DeleteResult` | 单 | `pre_delete_status` **仅首次**记录；`invalidated_at` 仅在 `=0` 时写；清 FTS/向量/队列；写 `DELETE` revision |
| 恢复 | `MemoryStore.restore(id, now): RestoreResult` | 单 | 回 `pre_delete_status`；`''` → `CANDIDATE`；重查 `dedupe_key`（含 `CANDIDATE`）与槽位；写 `RESTORE` revision |
| 彻底删除 | — | — | **C3**，不在 C1-B |

`RestoreResult` 五态（C0.2 §6.9）：`Restored(status)` / `Demoted(status, conflictId)` / `AlreadyPresent(occupantId)` / `NotFound` / `NotDeleted(status)`。

`DeleteResult` **`[实现补充]`**：C0.2 只写了"返回 NotDeleted / NotFound / 成功"三种语义但没给类型。C1-B 定为 `Deleted(preStatus)` / `NotDeleted(status)` / `NotFound`，与 `RestoreResult` 同构，避免调用方用 `Int` 猜。

### 2.6 Scope

| 规则 | 实现位置 |
|---|---|
| 只写 `GLOBAL` / `ASSISTANT("zhuli")` / `TASK(taskId)` | `MemoryStore.validateScope()`，其余抛 `IllegalArgumentException` |
| `GLOBAL` 的 `scope_id` 必须 `''`；其它非空、≤64、匹配 `[A-Za-z0-9._:-]+` | 同上 |
| 检索 `scopes` 为空 → 抛异常（不是返回空） | `MemoryQuery` 的 `init {}` + `MemoryStore.query()` 双检 |
| SQL 用显式 OR 列表 | `MemoryStore.scopeClause()`，**永不**写 `scope_type != 'X'` |
| 记忆页例外（可看全部 scope） | C1 不改 UI，仅保留 `MemoryQuery.forUi()` 的能力 |

### 2.7 SECRET 与来源

| 规则 | 实现 |
|---|---|
| SECRET 拒绝写入（唯一一处拒绝） | `MemoryGuard.classify(content) == SECRET` → `MemoryStore.insert` 返回 `Rejected(SECRET)`，不写任何表 |
| 分级只看 `content`（**运行时**） | `MemoryGuard.classify()` 内部调 `SensitiveGuard.reason(content)`，不额外扩正则（避免出现第二套敏感词表） |
| 分级要查四字段（**迁移**） | 旧表有 `source_text` 这个运行时没有的字段，必须 `subject`+`predicate`+`value`+`source_text` 全查；理由与顺序见 §5.3.2 |
| 来源摘录二次脱敏 | `MemoryGuard.excerpt(text)` = `SensitiveGuard.redact(text).take(120)` |
| `item.source_id` = PRIMARY 冗余 | 写 `evidence` 的同一事务里回填；`selfCheck` 有对应 SQL |
| 来源 GC 只在硬删路径 | C1 不做（硬删在 C3） |

### 2.8 FTS 三档（C1-A 已定 `TRIGGER`，但**不许硬编码**）

| `ftsMode` | 判定 | 写入路径 | 检索 |
|---|---|---|---|
| `TRIGGER` | 七步功能性探测全过 | `MemoryStore` **不碰** FTS | `MATCH` 子查询 |
| `APP_SYNC` | 触发器建不出，或建了但索引不动 | 每次 insert/update/status 变更后显式 `syncFts(id)` | `MATCH` 子查询 |
| `NONE` | FTS4 虚拟表都建不出 | 不管 FTS | `search_text LIKE` 拼接通配（`"%" + kw + "%"`） |

C1-A 的结论（`TRIGGER`）只说明**当前测试机**是这档。C0.2 §14.1 与 A-6 要求运行时探测，C1-A §7.2 把它写成三条硬约束。C1-B3 的实现见 §6。

### 2.9 迁移映射

全部照 C0.2 §8.1–§8.9，逐字段表在本文 §5.3 复述并加"代码位置"列。三条最容易被写错的：

1. `time_confidence` **一律 `NONE`**（不是 `if (valid_from>0) DAY`）；
2. `valid_until` **一律 `0`**，旧 `valid_to` 进 `invalidated_at`；
3. 唯一索引 `idx_mi_dedupe` **最后建**（先建会在第 2 步遇到历史重复就整批失败）。

### 2.10 不变量与测试编号

| 类别 | 数量 | 出处 |
|---|---|---|
| 残留检查 | 10 组 / 11 条 SQL | C0.2 §6.6 |
| 迁移时间不变量 | 5 条 | C0.2 §8.1 |
| 架构用例 | 57 条（54 在 C1，7 已在 C1-A） | C0.2 §11.2 |
| **决策落实用例** | **18 条**（D-6 的 12 + D-7 的 6） | 本文 §10.2 / §10.9 |
| 设备验收 | D1–D10 | C0.2 §11.3 |

> 18 条**不属于** C0.2 的 57 条编号体系：57 条是架构冻结时定义的"系统行为"用例，18 条是 D-6/D-7 两项决策带来的"兼容层与建表时机"用例。两套编号不交叉（`T-X*` 是新前缀，`T-M40`–`T-M45` 接在 A 组已有编号之后），避免把架构用例的数量口径搞乱。


C1-B 把前两类做成 `MemoryStore.selfCheck(): List<String>`（运行时）+ `MemoryMigrator.verify(): MigrationReport`（迁移末尾），**两套 SQL 不复用**——C0.2 §8.1 的注已经说明为什么（`extractor='import:legacy'` 的范围收窄只在迁移期成立）。

### 2.11 文件范围与退出条件

C0.2 §10.1/§10.2 给了 7 新增 + 2 修改；§11.4 给了 9 条退出标准。本文 §8 把它们展开成"路径 + 行数估算 + 归属子阶段"，§16 逐条给"怎么证明"。

---

## 3. 现有代码审查结果

### 3.1 旧记忆链路全图（已核查）

```
通知/短信 ─→ ExtractPipeline:105 ─┐
Agent 工具 ─→ ZhuliAgentTools:166,186 ─┤
回填     ─→ MemoryBackfill:106 ────────┴─→ MemoryRepo.ingest(FactDraft, SourceType, sourceId, now)
                                                     │
                                                     ├─→ MemoryExtractor.norm / FactDraft
                                                     ├─→ activeOf(subject, predicate)   ← 读 zhuli_fact
                                                     ├─→ insert(Fact)                   ← 写 zhuli_fact
                                                     ├─→ supersede(old, new, now)       ← valid_to = now
                                                     └─→ enqueueVector(id, now)         ← zhuli_vec_queue

用户纠错 ─→ ZhuliAgentTools:203 ─→ MemoryRepo.correct(id, value, now)
                                      ├─ copy(old) 且 validFrom = now        ← 缺陷 #1
                                      └─ supersede(old, new, now)

检索     ─→ ZhuliAgentTools:78 ─→ MemoryRepo.hybridSearch(text, provider, now, 8)
                                      ├─ search() → MemoryRanker.search(active(300))   ← 只吃 ACTIVE
                                      └─ 向量：zhuli_fact_vec WHERE space=?

索引     ─→ MemoryIndexer.drainOnce()
                                      ├─ repo.indexedSpaces() / activeCount()
                                      ├─ repo.enqueueAllActive(now)
                                      ├─ repo.pendingVectorFacts(BATCH) → vectorText(id)
                                      └─ repo.putVector / markVectorIndexed / markVectorFailed

UI       ─→ ZhuliMemoryPage ─→ MemoryRepo.active/byId/unconfirmed/history/conflicts
```

### 3.2 `MemoryRepo` 的 27 个公开成员 → 新实现

这一列是 C1-B4 的施工图。**"投影"= 只读新表并映射成 `Fact`；"转 Store"= 调 `MemoryStore` 并映射返回值。**

| # | 成员 | 新实现 | 备注 |
|---|---|---|---|
| 1 | `onIndexRequested` | 保留原样 | 写入完成后触发索引器 |
| 2 | `ingest(drafts, source, sourceId, now, activateThreshold)` | 转 `MemoryStore.insert(draft, …)` 逐条 | 返回"新建行数" |
| 3 | `correct(factId, newValue, now)` | 转 `MemoryStore.correct()` | 返回新 id 或 null |
| 4 | `deleteFact(id, now)` | 转 `MemoryStore.softDelete()` | `Deleted`→1，其余→0 |
| 5 | `byId(id)` | 投影 | — |
| 6 | `unconfirmed(limit)` | 投影 `status='CANDIDATE'` | — |
| 7 | `confirmFact(id, now)` | 转 `MemoryStore.confirm()` | CANDIDATE→ACTIVE + 入队 |
| 8 | `activeOf(subject, predicate)` | 投影 `CURRENT` + slot 子句 | 语义收紧（X-8） |
| 9 | `active(limit)` | 投影 `CURRENT` 按 `observed_at` 倒序 | 同上 |
| 10 | `activeCount()` | 投影 `COUNT(CURRENT)` | 同上 |
| 11 | `history(subject, predicate)` | 投影全部状态（含 DELETED / EXPIRED / REJECTED） | 六态一一投影，`UNKNOWN` 兜底（D-6） |
| 12 | `search(text, now, limit)` | `MemoryStore.search()` → `FactHit` | 内部走 FTS/LIKE |
| 13 | `conflicts()` | **重写**：`MemoryStore.conflictFacts()` 取 SINGLE 槽的 ACTIVE + CANDIDATE → `MemoryRanker.conflicts()` 配对 | **改 `MemoryRanker` 的一处过滤**（D-6 第 7 条，X-3 已决） |
| 14 | `resolveConflict(keep, drop, now)` | 转 `MemoryStore.resolveConflict()` | 写 `SUPERSEDED` + 关系 |
| 15 | `putVector` | 保持直写 `zhuli_fact_vec` | 向量不是记忆语义，不值得进 Store |
| 16 | `markVectorIndexed` | 同上 | — |
| 17 | `markVectorFailed` | 同上 | — |
| 18 | `dropVector` | 同上 | — |
| 19 | `pendingVectorFacts` | 同上，JOIN 改新表 | 见下 |
| 20 | `pendingVectorCount` | 同上 | — |
| 21 | `waitingVectorCount` | 同上 | — |
| 22 | `failedVectorCount` | 同上 | — |
| 23 | `indexedSpaces` | 同上 | — |
| 24 | `vectorCount` | 同上 | — |
| 25 | `enqueueAllActive` | 同上，`SELECT id FROM 新表 WHERE CURRENT` | — |
| 26 | `vectorText(id)` | 投影 + `display` | 改 `CURRENT`（与 #10/#11 一致；v2 这里写"只对 ACTIVE 返回"，与 §7.12.3 第 12 行矛盾，v3 统一为 `CURRENT`） |
| 27 | `hybridSearch(text, provider, now, limit)` | `search()` + 向量混合，逻辑不变 | `suspend` 保持 |

**`[实现补充]` 向量方法与 `MemoryStore` 的边界**：C0.2 §10.6 说 `MemoryStore` 是"所有运行时写入、所有查询"的唯一入口，但向量表 `zhuli_fact_vec` / `zhuli_vec_queue` 在 C0.2 里被归为"索引"而不是"记忆"。C1-B 决定：

- **向量读写留在 `MemoryRepo`**，因为它已经是"索引器 ↔ 存储"的适配层，且 13.2 明确要求只从 `MemoryRepo` 读；
- `MemoryStore.softDelete()` 仍**负责**删向量与队列（C0.2 §6.2 第 3、4 行），因为那是删除语义的一部分，不能漏；
- 这条边界写进 KDoc，避免以后有人把 `putVector` 搬进 `MemoryStore` 又搬回来。

### 3.3 三个必须保真的约束

| # | 约束 | 为什么 | 怎么验证 |
|---|---|---|---|
| 1 | **27 个公开签名一个不改** | 5 个生产调用方 + 现有 15 个单测都按签名编译 | `git diff` 里 `MemoryRepo.kt` 的 `fun ` 行逐条对照（§8.5 review checklist） |
| 2 | **`ingest` 返回"新建行数"** | 调用方拿它判断"这条通知有没有产生新记忆"（`ExtractPipeline` 日志、工具返回文案） | `T-S11`：同值两来源 → 返回 0 |
| 3 | **`hybridSearch` 保持 `suspend`** | `ZhuliAgentTools:78` 在协程里调 | 编译即验证 |

### 3.4 与 C0.2 的冲突清单（**10 处，X-1 / X-2 / X-3 已决定**）

按影响排序。X-1 / X-2 / X-3 已由项目负责人决定（D-7 / D-6 / D-6 第 7 条），本节保留**原始证据**与**决定后的施工口径**；完整落实方案在 §4.8（X-1）、§7.12（X-2、X-3）。其余 7 处仍按本文建议执行。

#### X-1（高·**已决定：方案 A**）`createCoreTables` 会在 v17 库里重建 `zhuli_fact`

**证据**：`InboxDatabase.kt`

```
269  override fun onCreate(...)      { …; createCoreTables(db); createV16Tables(db) }
335  if (!db.isReadOnly) { ensureColumns(db); createSourceTable(db); createCoreTables(db); … }
510  private fun createCoreTables(db) { … CREATE TABLE IF NOT EXISTS zhuli_fact … }
```

**冲突**：C0.2 §8.9 第 12 步把 `zhuli_fact` 改名为 `zhuli_fact_v1`，而 `onOpen` **每次开库**都会 `CREATE TABLE IF NOT EXISTS zhuli_fact`。于是：

1. 迁移后重启一次，`zhuli_fact` 以空表形式重新出现 → `T-M09`（"`zhuli_fact` 不存在"）**必然失败**；
2. C0.2 §13.7 的 fail-loud 设计失效——残留的写旧表代码不会报 `no such table`，而是**悄悄写进空表**；
3. 新装（`onCreate`）路径也会建一张永远为空的 `zhuli_fact`，与"v17 只有新表"矛盾。

**决定**：选方案 A——拆分 `createCoreTables`，v17 起不再创建 `zhuli_fact`。三条硬要求（来自 D-7，逐条对应见 §4.8）：

| D-7 要求 | 落实 |
|---|---|
| ① 新库 `onCreate` 创建完整的当前版本初始结构 | `createCoreTablesV15` + `createV16Tables` + `MemorySchema.createAll(withUnique=true)`，**不建** `zhuli_fact` |
| ② 旧库升级必须通过明确的版本迁移 | `onUpgrade`：`if (oldVersion<15) { createCoreTablesV15; createFactTableV15 }`，`if (oldVersion<17) MemoryMigrator.migrate` |
| ③ `onOpen` 不得无条件重建已迁移/废弃的旧记忆表 | `onOpen`：`createCoreTablesV15` + `MemorySchema.ensureAll`，**不建** `zhuli_fact` |
| ④ 不影响其他核心表 | `createCoreTablesV15` 保留 entity / task / step / fact_vec 与 3 个索引，一个不删 |
| ⑤ v17 及以后不得自动重建 `zhuli_fact` | 三个 CREATE 路径全部移除；`T-M40`/`T-M41`/`T-M43` 盯住 |
| ⑥ 检查所有 `createCoreTables` 调用点 | 3 处（`onCreate` 内 `:317` / `onOpen` 内 `:339` / `onUpgrade` 的 `oldVersion<15` 分支 `:787`），逐处替换见 §4.8.2 |
| ⑦ 明确长期核心结构 vs 旧记忆兼容结构 | §4.8.1 的 20 张表分类 |
| ⑧ 迁移后旧 Repository 不得静默写入空表 | 表不存在 → `no such table`；`MemoryRepo` 重写后零写旧表；`T-M42` 负向验证 |
| ⑨ 不得删除其他核心表的初始化逻辑 | `createCoreTablesV15` 原样保留 4 张表 + 3 索引，只把 `zhuli_fact` 与它的 3 个索引移出 |

代价：`InboxDatabase.kt` 的 diff 从 C0.2 估的 `+15 行` 变成约 `+34 / −10 行`（比 v1 的 +32/−8 多 2 行，因为要加 `createFactTableV15` 的调用与注释）。仍在允许范围内。

#### X-2（高·**已决定：方案 B + `UNKNOWN`**）`EXPIRED` 在 `FactStatus` 里没有对应值

**证据**：`MemoryTypes.kt:56-71`

```kotlin
enum class FactStatus { ACTIVE, UNCONFIRMED, SUPERSEDED, DELETED;
    companion object { fun of(raw: String) = entries.firstOrNull { it.name == raw } ?: ACTIVE }
}
```

**冲突**：C0.2 六态含 `EXPIRED` / `REJECTED`；`FactStatus` 只有四态。而 `FactStatus.of()` **未知值 fallback 到 `ACTIVE`**。于是：

- `byId()` 读到一条 `EXPIRED`（任务到期、STATE 关闭）→ 投影成 `ACTIVE` → **记忆页把过期值显示成当前值**，正是 C0.2 缺陷 #2 在投影层的翻版；
- `history()` 同理。

**决定**：扩展 `FactStatus`，加 `EXPIRED` / `REJECTED` / `UNKNOWN`，`of()` 未知值返回 `UNKNOWN`（不 fallback）。D-6 的 9 条要求逐条落实：

| D-6 要求 | 落实位置 | 验证 |
|---|---|---|
| ① 明确支持 `EXPIRED` / `REJECTED` | `MemoryTypes.kt` 枚举加 2 个值 | `T-X01` / `T-X06` |
| ② 必要时增加 `UNKNOWN` | 加 `UNKNOWN`，作为反序列化兜底 | `T-X02` |
| ③ `of()` 未知值不得 fallback 到 `ACTIVE` | `?: UNKNOWN` | `T-X02` |
| ④ 未知状态按非当前、不可自动激活 | `isCurrent` / `isActionable` 均为 false；`confirm` 的 `WHERE status='CANDIDATE'` 挡住 | `T-X04` |
| ⑤ 四态语义保持不同 | 数据库字符串、投影、查询可见性三层都不合并 | `T-X07` |
| ⑥ 检查所有使用方 | §7.12.3 的 15 行使用方清单 | 编译 + `T-X05` |
| ⑦ `conflicts` 支持新模型 | `MemoryRanker.conflicts` 过滤集合扩到 `ACTIVE` + `UNCONFIRMED` | `T-X09`–`T-X12` |
| ⑧ 旧 `Fact` 只做兼容投影 | `MemoryRepo` 零写旧表；`zhuli_fact` 不存在 | `T-M42` + `grep` |
| ⑨ 序列化/反序列化/DB 兼容入测 | 新增 `MemoryFactStatusTest` | 12 条 |

#### X-3（中·**已决定：改 `MemoryRanker.conflicts`**）`conflicts()` 只看 ACTIVE，C0.2 之后冲突方是 CANDIDATE

**证据**：`MemoryRanker.kt:111-123`

```kotlin
fun conflicts(facts: List<Fact>) = facts.filter { it.status == FactStatus.ACTIVE }
    .groupBy { it.slot } … // 同槽两条以上 ACTIVE 才算冲突
```

**冲突**：C0.2 §4.5 的 R8/R9 把"打不平的新值"放进 `CANDIDATE`，槽位里只留一条 ACTIVE。于是 `MemoryRanker.conflicts(active(300))` **永远返回空**，记忆页的"待裁决冲突"区域会静默消失。

**决定**：按 D-6 第 7 条，**修改 `MemoryRanker.conflicts` 本身**（而不是像 v1 建议的那样在 `MemoryRepo` 里另算一套）：

1. 过滤集合改为 `ACTIVE` + `UNCONFIRMED`（`FactStatus.isActionable`）；
2. 配对规则改为"ACTIVE × 每个不同值的 CANDIDATE"优先，全候选槽位退化为相邻配对；
3. `MULTI` 槽位的排除放在 `MemoryStore.conflictFacts()` 的 SQL 里（`cardinality='SINGLE'`），因为 `MemoryRanker` 拿不到 `cardinality`，纯函数不该猜存储语义；
4. 签名 `List<Pair<Fact, Fact>>` 不变，UI 零改动。

细节与伪码见 §7.12.5；验收 `T-X09`–`T-X12`。


#### X-4（中）`ingest(activateThreshold)` 与 R4b 缺少衔接规则

**证据**：`MemoryRepo.kt:33-38` 的 `activateThreshold` 参数；`MemoryBackfill.kt:124` 传非 0 值。现有实现：`confidence < threshold` → `UNCONFIRMED`，并且因为 `if (status == ACTIVE)` 守卫，**不参与冲突判定**。

**冲突**：C0.2 §4.5 的 R4b 说"新行按自身 basis 决定 ACTIVE/CANDIDATE（C2 的自动写入矩阵）"，没提这个阈值参数。两个"是否候选"的判据并存，谁优先没写。

**建议**（保持现有行为，不改架构）：`activateThreshold > 0 && confidence < threshold` → **强制 `CANDIDATE` 并跳过 R4–R9**，但仍参与 R2（同值则强化）。这与现有代码完全一致（`UNCONFIRMED` 也走 `activeOf` + `same` 判定）。写进 `MemoryStore.insert` 的 KDoc 与 `T-S11` 的变体。

#### X-5（低）`MemorySchema` 拿不到 `SharedPreferences`

C0.2 §14.1 要求"探测结果存 `SharedPreferences`，带探测版本"，但 §10.1 把 `MemorySchema` 设计成无状态 `object`（无 `Context`）。

**`[实现补充]`**：`MemorySchema.attach(prefs: SharedPreferences)` 在 `InboxDatabase` 的 `init` 里调一次（`InboxDatabase` 已有 `appCtx`）；未 attach 时退化为"每次开库都探测"，并在 debug 下 `Log.w` 提醒。**不改** `MemorySchema` 的纯函数性质。

#### X-6（低）`PRAGMA user_version = 17` 与 `SQLiteOpenHelper` 重复

C0.2 §8.9 第 13 步要求迁移末尾写 `PRAGMA user_version = 17`。但 `SQLiteOpenHelper` 在 `onUpgrade` 返回后会自己 `db.setVersion(mNewVersion)`（`InboxDatabase.kt` 的 `onUpgrade` 只是被它调用）。

**建议**：**不写**这条 PRAGMA，依赖 helper。理由：在 `onUpgrade` 内改 `user_version` 若与 helper 的顺序不符，会出现"版本号到了 17 但迁移没提交"的窗口。预演脚本（纯 Python、无 helper）**要**写，那是它自己的事。这条差异写进 `MemoryMigrator` 的 KDoc。

#### X-7（低）`deleteFact` 返回 `Int`，恢复没有对应 API

现有 `deleteFact(id, now): Int`（影响行数）；C0.2 的 `RestoreResult` 五态无处安放。且 `MemoryRepo` 没有 `restore` 方法——C1 也没有 UI 调它。

**建议**：`MemoryRepo.deleteFact` 保持 `Int`（`Deleted`→1，其余→0）；`restore()` **只**暴露在 `MemoryStore`（C0.2 §6.9 的"任务边界"已这么定），C1 的 `MemoryRepo` 不转发。测试直接测 `MemoryStore`。

#### X-8（低）`active(limit)` 收紧为 `CURRENT`

旧语义 `status='ACTIVE'`；新语义 `CURRENT`（多一个 `LB <= now`）。一条"明年起生效"的 ACTIVE 行会从记忆页消失。

**评估**：C1 不产生这种行（R0 清零 + 强制 CANDIDATE；T-S39/T-T40 盯住），迁移也不产生（`valid_from` 都是过去的 `now` 副本）。所以**行为差异为 0**，但要在 `active()` 的 KDoc 里写明，防止 C2 引入"未来生效"后有人以为是 bug。

#### X-9（低）`zhuli_fact_vec` 表名保留但语义变了

`fact_id` 从"`zhuli_fact.id`"变成"`zhuli_memory_item.id`"。改表名会波及 `MemoryEmbedding.kt` 以外的常量（`Db.COL_FV_*`）与 4 处现有 SQL。

**建议**：**不改名**，只改注释与 KDoc。ID 保真（§8.6）保证旧向量仍然指向正确的行。

#### X-10（低）`backupBeforeMigration` 在 `onUpgrade` 事务内复制主库

`SQLiteOpenHelper` 在调 `onUpgrade` **之前**就开了事务，所以 `backupBeforeMigration` 复制的是"事务已开、尚未写入"的文件。当前 `journal_mode=delete` 下可用（C0.2 §8.10 已记录），但若哪天改 WAL 就会缺内容。

**建议**：C1-B2 在迁移入口加 `PRAGMA journal_mode` 断言（`wal` → 先 `wal_checkpoint(FULL)`），不改 `backupBeforeMigration` 本身（那是既有机制，动它超出 C1-B 范围）。

### 3.5 旧表/旧字段的下游引用（已核查，无隐藏依赖）

| 检查 | 结果 |
|---|---|
| grep 查 `.validFrom` 与 `.validTo` | 只命中 `MemoryRepo.kt:405-406`（写入），**无读路径**（C0.2 A-7 复核通过） |
| `grep "TABLE_FACT" app/src/main` | 只在 `InboxDatabase.kt`（DDL）与 `MemoryRepo.kt`（将被重写）→ 重写后只剩 `InboxDatabase` |
| `grep "com.zhuli.memory" app/src/main` | 无反射、无 `Class.forName`、无 Gson 序列化 `Fact` |
| `Fact` 的序列化 | 只在进程内传递；`AssistantExportImport` 用的是宿主 `MemoryEntity`，与 `Fact` 无关 |
| 宿主 `MemoryRepository` | 与 `com.zhuli.memory.MemoryRepo` **同名不同物**，两套并存，C1 不合并（C0 已定） |

结论：C0.2 的 A-7（"`Fact` 字段无下游反射依赖"）**成立**，`MemoryRepo` 重写不会引起 `NoSuchField`。

---

## 4. 新旧数据结构对照

### 4.1 相关表总览（v16 / v17 对照）

| 表 | v16 | v17 | C1-B 动作 |
|---|---|---|---|
| `inbox` | 有 | 有 | 不动 |
| `inbox_fts` | 有（可能建不出） | 有 | 不动（与记忆 FTS 是两套） |
| `chat` / `reminder` / `ledger` / `cloud_log` / `skill` / `embedding` / `source_rule` / `tool_exec` | 有 | 有 | 不动 |
| `zhuli_fact` | **16 列** | **不存在** | 迁移后改名 `zhuli_fact_v1` |
| `zhuli_fact_v1` | — | 16 列（只读） | 迁移产出 |
| `zhuli_entity` | 7 列 | 7 列 | **不动**（新表 `zhuli_memory_entity` 是另一张） |
| `zhuli_fact_vec` | 4 列 | 4 列 | 结构不动，`fact_id` 语义改为 `item.id` |
| `zhuli_vec_queue` | 4 列 | 4 列 | 不动 |
| `zhuli_memory_item` | — | **40 列** | 新增 |
| `zhuli_memory_source` | — | 12 列 | 新增 |
| `zhuli_memory_evidence` | — | 6 列 | 新增 |
| `zhuli_memory_relation` | — | 6 列 | 新增 |
| `zhuli_memory_revision` | — | 10 列 | 新增 |
| `zhuli_memory_entity` | — | 3 列 | 新增 |
| `zhuli_memory_fts` | — | FTS4 虚拟表（1 列 + 5 影子表） | 新增，`ftsMode != NONE` 时 |

> **`zhuli_entity` 与新表并存**：C0.2 没有把旧 `zhuli_entity` 迁进 `zhuli_memory_entity`（前者是 `kind+norm` 去重表，后者是 `memory↔entity` 关联表，语义不同）。C1 保留旧表不动，`zhuli_memory_entity` 在 C1 **只建不写**（写入方在 C2 的模型抽取）。这是 C0.2 §2.5 的既定设计，不是遗漏。
>
> **每张表的"结构归属"（长期核心 / 旧记忆兼容）见 §4.8.1**。这个分类是 D-7 的直接产物：它决定 `onCreate` / `onOpen` / `onUpgrade` 各自能建哪些表，不能凭"表名里有 zhuli 就都建"。


### 4.2 `zhuli_fact`（16 列）→ `zhuli_memory_item`（40 列）逐列对照

| # | 旧列（v16） | 新列（v17） | 变换 | 丢失信息？ |
|---|---|---|---|---|
| 1 | `id` | `id` | 原值 | 否 |
| 2 | `subject` | `subject` | `trim()`，空则跳过 | 空主体行丢弃（C0.2 #2） |
| 3 | `predicate` | `predicate` | `trim()` | 否 |
| 4 | `value` | `value` | `trim()`，空则跳过 | 空值行丢弃（C0.2 #5） |
| 5 | `kind` | `legacy_kind` | `FactKind.name` 原样 | 否 |
| 6 | `kind` | `memory_type` | `PREFERENCE`→`PREFERENCE`，其余→`FACT` | 类型变粗（C0.2 §8.2 有意为之） |
| 7 | `confidence` | `confidence` | 按 basis 上限 clamp（只降不升） | 高估的置信度被降（C0.2 §8.3） |
| 8 | `source_type` | `confidence_basis` | 5 条映射（§5.3） | 否 |
| 9 | `source_type` + `source_id` + `source_text` | `zhuli_memory_source` 行 + `evidence(PRIMARY)` | 见 §5.5 | 否（`source_text` 截 120 并脱敏，原文还在 `inbox`） |
| 10 | `observed_at` | `observed_at` | 原值；0→`created_at`；仍 0→`now` | 否 |
| 11 | `valid_from` | `valid_from` | **原值保留**（0 保留 0） | 否，但**不可信**（`time_confidence=NONE`） |
| 12 | `valid_to` | `valid_until` | **一律 0** | 旧值转记 `invalidated_at`（#23），不丢 |
| 13 | `status` | `status` | 4→6（§5.3） | 否 |
| 14 | `superseded_by` | `superseded_by` | 原值；目标不存在则清 0 | 断链只清指针，保 `invalidated_at` |
| 15 | `created_at` | `recorded_at` | 原值 | 否 |
| 16 | `updated_at` | `updated_at` | 原值 | 否 |

新增的 **29 列**（旧表 16 列中 11 列同名对应，其余 5 列拆分或丢弃——见上表）分五组：

| 组 | 列（共 29） | 值 | 依据 |
|---|---|---|---|
| 内容与检索（7） | `subject_key` `content` `normalized_content` `dedupe_key` `slot_hash` `search_text` `qualifier_key` | `qualifier_key=''`，其余见 §5.3 | C0.2 §2.2 / §2.7 |
| 分类与来源（4） | `memory_type` `cardinality` `confidence_basis` `legacy_kind` | 见 §5.3 的映射表 | C0.2 §8.2 / §8.3 |
| 时间（9） | `event_start` `event_end` `valid_until` `recorded_at` `last_confirmed_at` `invalidated_at` `time_confidence` `deleted_at` `pre_delete_status` | `event_start/end=0`、`valid_until=0`、`time_confidence='NONE'`、`pre_delete_status=''` | C0.2 §4.1 / §8.1 |
| 权限与治理（4） | `scope_type` `scope_id` `sensitivity` `extractor` | `'GLOBAL'` / `''` / 分级结果 / `'import:legacy'` | C0.2 §7.1 / §6.7 / §8.1 |
| 版本与统计（5） | `revision` `payload_json` `times_reinforced` `times_retrieved` `last_used_at` | `1` / `{"migrated_from":N}` / `0`（前者第 5 步重算） | C0.2 §8.1 |

> 7 + 4 + 9 + 4 + 5 = **29**，与 40 − 11 = 29 相符。

### 4.3 DDL 落地物与一致性校验

C0.2 §2.2–§2.6 已给出 7 段 DDL，本文**不复制粘贴**（复制会产生两份真相）。落地方式是：

| 项 | 做法 |
|---|---|
| 唯一真相 | `MemorySchema.kt` 的 `DDL_*` 常量 |
| 机器可验的一致性 | `MemorySchemaTest.schema_matches_expected()`：建库后读 `PRAGMA table_info(zhuli_memory_item)` 与 `sqlite_master`，与测试里的 `EXPECTED_COLUMNS`（40 个列名，有序）/ `EXPECTED_INDEXES`（15 个索引名 + 唯一性）逐项比对。**这能抓住 DDL 拼写错、漏列、索引少建** |
| 人工对照 | C1-B1 完成时，把 `DDL_*` 与 C0.2 §2.2–§2.6 逐段对照一遍，结论写进 C1-B 报告（"逐段一致 / 差异 N 处及理由"） |
| 不做的检查 | 不在单测里解析 markdown（路径脆弱、且 C0.2 是文档不是契约源）；不做 SQL 语法树 fuzz |

> **为什么值得写这个测试**：C0.2 是"冻结的架构"，实现漂移是最难发现的缺陷。一个 20 行的 `table_info` 比对能永久防住"漏一列 / 拼错名 / 少建索引"，成本远低于每次人工 review 40 列。

### 4.4 建表顺序

顺序不是随意的，每一步都有理由：

```
S1  zhuli_memory_item            （无外键，可先建）
S2  zhuli_memory_source          （被 evidence 引用）
S3  zhuli_memory_evidence        （依赖 S1/S2 语义，但**不建外键**）
S4  zhuli_memory_relation        （自引用，无外键）
S5  zhuli_memory_revision        （无外键）
S6  zhuli_memory_entity          （无外键）
S7  普通索引 × 12                （见 §4.5.1 的 #1–#5、#7、#9、#11–#15）
S8  zhuli_memory_fts（虚拟表）    ← 仅 ftsMode != NONE
S9  触发器 × 3                    ← 仅 ftsMode == TRIGGER
S10 UNIQUE INDEX idx_mi_dedupe    ← **迁移时最后建**；运行时建表可随 S7
S11 UNIQUE INDEX idx_mev_primary  ← 随 S7
S12 UNIQUE INDEX idx_ms_dedupe    ← 随 S7
---
合计：6 张表 + 12 普通 + 3 唯一 = 15 个显式索引
另有 4 个自动索引由 S1/S3/S4/S5/S6 的表内 PK/UNIQUE 自动产生（§4.5.2）
```

**为什么不用外键**：C0.2 的 DDL 全部没有 `FOREIGN KEY`。这不是省事：`ON DELETE CASCADE` 会在硬删时跳过 revision 记录（审计丢失），而 `PRAGMA foreign_keys` 在 Android 上默认关闭、需要每连接开。C1 用**应用层顺序 + `selfCheck`** 保证一致性（C0.2 §6.3 的子→父顺序）。

**为什么 `idx_mi_dedupe` 在迁移时最后建**：C0.2 §8.9 第 10 步。先建会在逐行插入时撞上历史重复，整批失败。运行时（`createAll` 的 `withUnique=true` 分支）可以一次建好，因为那时表是空的。

### 4.5 索引与约束清单（15 个显式 + 4 个自动，P0-6 重写）

v2 的这张表有 15 行，但只写了名字和列，**没写"创建时机"、没区分"显式/自动"、也没核对唯一约束的 DDL 是否真的存在**。v3 补齐，并给出计数口径——否则 `MemorySchemaTest` 的 `EXPECTED_INDEXES` 到底该断言 15 个还是 19 个，实现者会猜。

**口径**：

```
19 个索引 = 15 个显式索引（CREATE INDEX / CREATE UNIQUE INDEX）
          +  4 个自动索引（SQLite 为复合 PRIMARY KEY / UNIQUE 自动创建，名为 sqlite_autoindex_*）
不计入索引：5 个 INTEGER PRIMARY KEY AUTOINCREMENT（rowid 别名，不建独立索引，只维护 sqlite_sequence）
```

#### 4.5.1 15 个显式索引（12 普通 + 3 唯一）

| # | 名称 | 表 | 类型 | 列 / 条件 | 服务于 | 创建时机 |
|---|---|---|---|---|---|---|
| 1 | `idx_mi_slot` | item | 普通 | `(subject_key, predicate, qualifier_key, status)` | R4 取同槽 ACTIVE | 随建表（S7） |
| 2 | `idx_mi_scope` | item | 普通 | `(scope_type, scope_id, status, memory_type)` | scope 过滤 + CURRENT | 随建表（S7） |
| 3 | `idx_mi_type_time` | item | 普通 | `(memory_type, status, valid_until, event_start)` | as-of / 过期扫描 | 随建表（S7） |
| 4 | `idx_mi_observed` | item | 普通 | `(status, observed_at)` | `active()` 倒序、BELIEF_ASOF | 随建表（S7） |
| 5 | `idx_mi_source` | item | 普通 | `(source_id)` | 主来源反查 | 随建表（S7） |
| 6 | **`idx_mi_dedupe`** | item | **唯一（部分）** | `(dedupe_key) WHERE status IN ('ACTIVE','CANDIDATE')` | 防重复；**恢复时的主要冲突源** | **运行时随建表；迁移时最后建（S10）** |
| 7 | `idx_ms_ref` | source | 普通 | `(source_type, ref_table, ref_id)` | 回溯源 | 随建表（S7） |
| 8 | **`idx_ms_dedupe`** | source | **唯一** | `(source_type, ref_table, ref_id, ref_sub_id, content_hash)` | 同一证据幂等 | 随建表（S11） |
| 9 | `idx_mev_source` | evidence | 普通 | `(source_id)` | 来源引用计数 / GC | 随建表（S7） |
| 10 | **`idx_mev_primary`** | evidence | **唯一（部分）** | `(memory_id) WHERE role='PRIMARY'` | 一条记忆最多一个主来源 | 随建表（S11） |
| 11 | `idx_mr_from` | relation | 普通 | `(from_id, relation)` | 查"我替代了谁" | 随建表（S7） |
| 12 | `idx_mr_to` | relation | 普通 | `(to_id, relation)` | WORLD_ASOF 的 `CORRECTS` 排除 | 随建表（S7） |
| 13 | `idx_mrev_mem` | revision | 普通 | `(memory_id, revision_no)` | 历史 | 随建表（S7） |
| 14 | `idx_mrev_at` | revision | 普通 | `(created_at)` | 按时间清理 | 随建表（S7） |
| 15 | `idx_me_entity` | entity | 普通 | `(entity_id)` | 反查 | 随建表（S7） |

**三个唯一约束的 DDL 已逐一核对存在**（这是 P0-6 明确要求的核验，不能只写"有"）：

| 约束 | C0.2 行号 | 原始 DDL |
|---|---|---|
| `idx_mi_dedupe` | **201** | `CREATE UNIQUE INDEX IF NOT EXISTS idx_mi_dedupe ON zhuli_memory_item(dedupe_key) WHERE status IN ('ACTIVE','CANDIDATE');` |
| `idx_ms_dedupe` | **260** | `CREATE UNIQUE INDEX IF NOT EXISTS idx_ms_dedupe ON zhuli_memory_source(source_type, ref_table, ref_id, ref_sub_id, content_hash);` |
| `idx_mev_primary` | **286** | `CREATE UNIQUE INDEX IF NOT EXISTS idx_mev_primary ON zhuli_memory_evidence(memory_id) WHERE role='PRIMARY';` |

> 三个都是 `CREATE UNIQUE INDEX`，**不是**表内 `UNIQUE(...)` 子句。这个区别有实际后果：部分唯一索引（`WHERE`）只能用 `CREATE UNIQUE INDEX` 表达，表内约束做不到。所以拆分 `withUnique` 参数时，这三个必须跟着索引走，不能塞进 `CREATE TABLE`。

#### 4.5.2 4 个自动索引（不可删、不出现在 `EXPECTED_INDEXES` 的 CREATE 清单里）

| # | 名称（SQLite 自动生成） | 表 | 来源 | 能否删 |
|---|---|---|---|---|
| 16 | `sqlite_autoindex_zhuli_memory_evidence_1` | evidence | `PRIMARY KEY(memory_id, source_id)` | **不能**（删表才消失） |
| 17 | `sqlite_autoindex_zhuli_memory_relation_1` | relation | `UNIQUE(from_id, to_id, relation)` | 不能 |
| 18 | `sqlite_autoindex_zhuli_memory_revision_1` | revision | `UNIQUE(memory_id, revision_no)` | 不能 |
| 19 | `sqlite_autoindex_zhuli_memory_entity_1` | entity | `PRIMARY KEY(memory_id, entity_id, role)` | 不能 |

**`MemorySchemaTest.schema_matches_expected()` 的断言方式**（v3 明确，避免实现者纠结）：

```sql
SELECT name, "unique" FROM pragma_index_list('zhuli_memory_item');   -- 6 行（5 普通 + 1 唯一）
SELECT name FROM sqlite_master WHERE type='index' AND tbl_name='zhuli_memory_item';  -- 同样 6 行
-- 自动索引只在该表的 pragma_index_list 里出现，name 以 sqlite_autoindex_ 开头
```

- `EXPECTED_INDEXES`（显式，15 项）：断言 **存在性 + 唯一性 + 列顺序**；
- `EXPECTED_AUTO_INDEXES`（自动，4 项）：只断言 **存在性**（名字由 SQLite 决定，不硬编码后缀数字）；
- **不做**"索引总数 == 19"的断言——因为 `sqlite_sequence` 这类内部表也会出现在 `sqlite_master`，硬数总数会在不同 SQLite 版本上假失败。

#### 4.5.3 5 个 rowid 别名（不是索引，但影响 `sqlite_sequence`）

`item.id` / `source.id` / `relation.id` / `revision.id` 四张表都是 `id INTEGER PRIMARY KEY AUTOINCREMENT`。`INTEGER PRIMARY KEY` 是 rowid 的别名，SQLite **不**为它建独立索引（B-tree 本身就是 rowid 表）。`AUTOINCREMENT` 的额外代价是维护 `sqlite_sequence`（§5.7）。

> `evidence` 与 `entity` **没有** `id` 列，它们的 PK 是复合的 → 走自动索引（§4.5.2）。这是 `T-M46`（后续 ID 不冲突）只针对 `zhuli_memory_item` 的原因。

**索引数量的取舍（口径不变）**：C0.2 删掉了 `idx_ms_hash` / `idx_ms_indep`（2 万行省 2.3MB），因为 5 元组唯一索引的前缀已覆盖。C1-B **不加**任何"看起来有用"的索引——每加一个都要在 20k 预演库里量体积（§11.4）；要删也必须走 §14.3 的可评估清单，不能删约束。

### 4.6 表关系图

```
                    ┌──────────────────────────┐
                    │    zhuli_memory_item     │ 40 列
                    │  id (PK, AUTOINCREMENT)  │
                    │  status / memory_type    │
                    │  scope_type + scope_id   │
                    │  dedupe_key (uq, partial)│
                    │  slot_hash / search_text │
                    │  pre_delete_status       │
                    └───┬───────┬───────┬──────┘
                        │       │       │
        source_id (冗余) │       │       │ id
                        │       │       └──────────────┐
                        ▼       ▼                      ▼
   ┌────────────────────────┐  ┌──────────────────┐  ┌─────────────────────┐
   │ zhuli_memory_evidence  │  │  relation        │  │  revision           │
   │ PK(memory_id,source_id)│  │ from_id → item   │  │ memory_id → item    │
   │ role: PRIMARY/SUP/DRV  │  │ to_id   → item   │  │ UNIQUE(mem, rev_no) │
   │ uq(memory_id) WHERE    │  │ UNIQUE(f,t,rel)  │  │ change_kind 9 种    │
   │    role='PRIMARY'      │  └──────────────────┘  └─────────────────────┘
   └───────────┬────────────┘
               │ source_id
               ▼
   ┌────────────────────────┐        ┌──────────────────────────┐
   │ zhuli_memory_source    │        │ zhuli_memory_entity      │
   │ id (PK)                │        │ PK(memory_id,entity_id,  │
   │ independent_key        │        │    role)                 │
   │ uq(5 元组)              │        │ C1 只建不写              │
   └────────────────────────┘        └──────────────────────────┘

   ┌────────────────────────┐        ┌──────────────────────────┐
   │ zhuli_memory_fts (FTS4)│        │ zhuli_memory_item         │
   │ rowid = item.id        │◀── 3 个触发器同步 ──│ search_text / status │
   └────────────────────────┘        └──────────────────────────┘

   外部共享（结构不改，语义改）：
   zhuli_fact_vec.fact_id ──→ item.id
   zhuli_vec_queue.fact_id ──→ item.id
```

**没有画进图的**：`zhuli_memory_tombstone` / `usage` / `ingest`（C3/C4），`zhuli_fact_v1`（只读遗留表，无引用关系）。

### 4.7 数据库版本升级策略

| 起点 | 走哪条路 | 关键动作 |
|---|---|---|
| 新装（version=0） | `onCreate` | 建 v16 全部长期核心表（**不含** `zhuli_fact`，见 X-1 / §4.8）+ `MemorySchema.createAll(db, withUnique=true)` + `detectFtsMode` |
| v16 | `onUpgrade(16,17)` → `upgradeSteps` 末尾 | `MemorySchema.createAll(withUnique=false)` → `MemoryMigrator.migrate()` → 建 FTS → 建触发器 → 建唯一索引 → 改名 |
| v15 及更早 | `upgradeSteps` 逐版补齐 → 再走 v17 | `if (oldVersion<15)` 先 `createCoreTablesV15` + `createFactTableV15`（老库确实有这张表），其余同 v16 |
| v15/v16 但 `zhuli_fact` 缺失（损坏库） | `onUpgrade` | **不建空表**。迁移前置断言抛 → 回滚 → 报错（`T-M45`）。这不是"v17 重造旧表"，而是"拒绝在缺源表的库上假装迁移成功" |
| v17 再次打开 | `onOpen` | `createCoreTablesV15` + `MemorySchema.ensureAll`（缺表补表、缺触发器补触发器，**不重建** `zhuli_fact`） |
| v17 → 任意低版本 | `onDowngrade` | **未实现**。SQLiteOpenHelper 默认抛 `SQLiteException`。C1-B 不实现降级（§12.3 说明为什么） |
| 迁移中途失败 | 事务回滚 | 版本仍 16；`lastMigrationError` 有原因；下次打开重试 |

**版本号只 +1**（16→17），`DB_VERSION` 只在 `InboxDatabase.kt` 改一处。**不写 `PRAGMA user_version`**（X-6）。

**三个建表入口的职责边界（D-7 的核心，任何一处写反都会让旧表复活或让核心表缺失）**：

| 入口 | 允许建 | 禁止建 | 理由 |
|---|---|---|---|
| `onCreate`（新装） | 长期核心表（§4.8.1 的 A 组 + C 组） | `zhuli_fact` / `zhuli_fact_v1` / 任何 `zhuli_memory_*` 之外的记忆表 | 新装库没有旧数据，建旧表只会造出"永远为空的假表" |
| `onOpen`（每次开库） | 长期核心表 + `MemorySchema.ensureAll`（7 张新表的 `IF NOT EXISTS`） | `zhuli_fact` / `zhuli_fact_v1` | 自愈只能修"结构缺失"，不能把已迁移/已废弃的表造回来 |
| `onUpgrade`（v17 迁移） | `createFactTableV15`（**仅 `oldVersion<15`**）+ `MemorySchema.createAll` + 迁移产出 | 其它 | 老库要先有源表才能迁；v15/v16 库的源表本应存在，缺失即损坏 |

---

### 4.8 表分类与建表职责（D-7 落实）

D-7 的第 7 条要求"明确哪些表属于长期核心结构，哪些属于旧记忆兼容结构"。这个分类不是文档洁癖：它直接决定 `onCreate` / `onOpen` / `onUpgrade` 三个入口的建表清单，也是 `T-M40`–`T-M45` 的断言依据。

> 计数口径：**24 张表** = A 组 9 + B 组 5 + C 组 7 + D 组 3。不含 `inbox_fts` / `zhuli_memory_fts` 的 5 张影子表（由 SQLite 自动创建）与 2 组共 6 个触发器（不是表）。

#### 4.8.1 全库表的归属（24 张表，按"v17 起还要不要创建"分类）

**A 组｜长期核心·采集与提醒（v17 前后完全一致，D-7 不动它们）**

| 表 | 列/形态 | 创建函数 | 调用点 |
|---|---|---|---|
| `inbox` | 21 列 + 4 索引 | `onCreate` 内联 | `onCreate` |
| `inbox_fts` | FTS5 外部内容表（可能建不出） | `onCreate` 内联 `try/catch` | `onCreate` |
| `inbox_fts` 的 3 个触发器 | — | `ensureFtsSync` | `onOpen`（一次性） |
| `chat` | 7 列 | `createChatTable` | `onCreate` / `onUpgrade(<3)` |
| `reminders` | 13 列 | `createReminderTable` | `onCreate` / `onUpgrade(<4)` |
| `ledger` | 6 列 | `createLedgerTable` | `onCreate` / `onUpgrade(<6)` |
| `cloud_log` | 10 列 + 1 索引 | `createCloudLogTable` | `onCreate` / `onUpgrade(<8)` |
| `skill` | 7 列 | `createSkillTable` | `onCreate` / `onUpgrade(<10)` |
| `embedding` | 6 列 | `createEmbeddingTable` | `onCreate` / `onUpgrade(<11)` |
| `source_rule` | 3 列 | `createSourceTable` | `onCreate` / `onOpen` / `onUpgrade(<13)` |

**B 组｜长期核心·Agent 与索引（原 `createCoreTables` 的一部分，v17 起继续创建）**

| 表 | 列/形态 | 归属函数 | 说明 |
|---|---|---|---|
| `zhuli_agent_task` | 13 列 + 2 索引 | `createCoreTablesV15` | Agent 状态机，与记忆无关，**绝不能动** |
| `zhuli_agent_step` | 13 列 + 1 索引 | `createCoreTablesV15` | 同上 |
| `zhuli_fact_vec` | 4 列 | `createCoreTablesV15` | 向量索引；v17 后 `fact_id` 指 `zhuli_memory_item.id`，**表结构不改**（X-9） |
| `zhuli_vec_queue` | 4 列 | `createV16Tables` | 索引队列，同上 |
| `zhuli_tool_exec` | 6 列 | `createV16Tables` | 工具执行留档 |

**C 组｜长期核心·新记忆（v17 新增，`MemorySchema` 负责）**

| 表 | 形态 | 归属函数 |
|---|---|---|
| `zhuli_memory_item` | 40 列 + 12 普通索引 + 3 唯一索引 | `MemorySchema.createAll` / `ensureAll` |
| `zhuli_memory_source` | 12 列 + 2 索引 | 同上 |
| `zhuli_memory_evidence` | 6 列 + 1 索引 + 1 唯一索引 | 同上 |
| `zhuli_memory_relation` | 6 列 + 2 索引 + 1 表内唯一 | 同上 |
| `zhuli_memory_revision` | 10 列 + 2 索引 + 1 表内唯一 | 同上 |
| `zhuli_memory_entity` | 3 列 + 1 索引 + 主键 | 同上 |
| `zhuli_memory_fts` | FTS4 虚拟表 + 5 影子表 + 3 触发器 | `MemorySchema.createFts*`（`ftsMode != NONE`） |

**D 组｜旧记忆兼容（v17 起不再创建；只读或只做迁移源）**

| 表 | v17 状态 | 谁创建 | 谁还能写 | 处置 |
|---|---|---|---|---|
| `zhuli_fact` | **不存在** | 仅 `createFactTableV15`（**只在 `oldVersion<15` 的升级路径**） | 无人 | 迁移第 12 步改名 |
| `zhuli_fact_v1` | 存在（迁移后） | **无人**（只由 `ALTER TABLE RENAME` 产生） | 无人（只读） | C3 清理（§17 第 8 项） |
| `zhuli_entity` | 存在（7 列，只读） | `createCoreTablesV15`（保留，见 §4.8.5） | **C1-B 后无写入方** | C2 评估是否并入 `zhuli_memory_entity` |

> **`zhuli_entity` 为什么留在 `createCoreTablesV15` 而不是也移出去**：D-7 第 9 条明确"不得为了修复 X-1 而直接删除其他核心表的初始化逻辑"。`zhuli_entity` 不是废弃表（C0.2 §2.5 说"保留不动"），它只是**暂时没有写入方**。移出创建逻辑会让"老库有、新库没有"，制造新的结构漂移；而保留创建的代价是 7 列的空表 + 1 个唯一索引，可以忽略。

#### 4.8.2 三个调用点（已核查，逐处替换）

`grep -n "createCoreTables" InboxDatabase.kt` 的完整结果只有 3 处：

| # | 行 | 现状 | v17 改成 | 为什么 |
|---|---|---|---|---|
| 1 | `onCreate` 内调用点 `:317` | `createCoreTables(db)` | `createCoreTablesV15(db)`<br>`MemorySchema.attach(prefs)`<br>`MemorySchema.createAll(db, withUnique = true)` | 新装库要完整当前结构，不要旧表 |
| 2 | `onOpen` 内调用点 `:339` | `createCoreTables(db)` | `createCoreTablesV15(db)`<br>`MemorySchema.ensureAll(db)` | 自愈不重建旧表 |
| 3 | `onUpgrade` 的 `oldVersion<15` 分支内调用点 `:787` | `createCoreTables(db)` | `createCoreTablesV15(db)`<br>`createFactTableV15(db)` | 老库要补齐**全部** v15 表（含源表），然后才能迁 |

`onCreate` 的 `createV16Tables(db)`（318 行）与 `onOpen` 的（340 行）**保持原样**——D-7 第 4 条。

#### 4.8.3 拆分后的两个函数（签名与内容）

```kotlin
/**
 * v15 核心表（**不含** zhuli_fact）。
 * onCreate / onOpen / onUpgrade(<15) 三处共用；全部 IF NOT EXISTS。
 */
private fun createCoreTablesV15(db: SQLiteDatabase) {
    // 原样保留：zhuli_entity（7 列 + UNIQUE(kind,norm)）
    // 原样保留：zhuli_agent_task（13 列）+ idx_task_status + idx_task_dedupe
    // 原样保留：zhuli_agent_step（13 列 + UNIQUE(task,seq)）+ idx_step_task
    // 原样保留：zhuli_fact_vec（4 列，fact_id 主键）
}

/**
 * v15 的旧记忆主体表 + 3 个索引。
 * **只允许**在 onUpgrade 且 oldVersion < 15 时调用 —— 那是"老库还没有这张表"的唯一合法场景。
 * v17 及以后任何路径都不得调用（D-7 第 5 条）。
 */
private fun createFactTableV15(db: SQLiteDatabase) {
    // CREATE TABLE IF NOT EXISTS zhuli_fact（16 列，与 v16 逐字一致）
    // CREATE INDEX IF NOT EXISTS idx_fact_slot / idx_fact_source / idx_fact_status
    // KDoc 必须写明："这是迁移源表的兜底创建，不是 v17 的运行时表"
}
```

**拆分的技术要求**：

1. `createFactTableV15` 的 DDL 必须与 v16 现状**逐字一致**（16 列、3 个索引名、`AUTOINCREMENT`），否则 `MemoryMigrator` 的 `SELECT *` 列序假设会失效；
2. `createCoreTablesV15` 的内容必须是原 `createCoreTables` 的**真子集**（删掉 fact 表与 3 个 fact 索引），不许顺手改别的表；
3. 两个函数都保持 `private`，不新增 public API；
4. 函数名带 `V15` 是为了让后来者一眼看出"这是历史版本的结构"，而不是"当前版本的核心表"。

#### 4.8.4 "旧 Repository 不得静默写入空表"的两层保证

D-7 第 8 条要求的不是一个断言，而是**两条独立的防线**：

| 层 | 机制 | 违反时的表现 | 验证 |
|---|---|---|---|
| 结构层 | v17 库里 `zhuli_fact` **不存在** | 任何 `INSERT INTO zhuli_fact` 抛 `SQLiteException: no such table: zhuli_fact`（fail-loud） | `T-M42` |
| 代码层 | `MemoryRepo` 重写后不含 `TABLE_FACT` 的写语句；`Db.TABLE_FACT` 常量只被 `InboxDatabase` 的 `createFactTableV15` 与 `MemoryMigrator` 引用 | `grep -rn "TABLE_FACT" app/src/main` 只剩 2 个文件；多一个就是回归 | §8.5 自检第 5 条 |

为什么两层都要：
- 只有结构层：`onOpen` 一旦被改回"补建旧表"，fail-loud 就静默失效（这正是 X-1 的成因）；
- 只有代码层：`grep` 防不住字符串拼接（`"zhuli_" + "fact"`）或反射，表不存在才是最终的运行时保证。

**`MemoryRepo` 重写后必须删掉的方法**：`insert` / `reinforce` / `supersede`（三者**写** `zhuli_fact`）、`upsertEntity`（写 `zhuli_entity`，见 §4.8.5）、以及 `activeOf` / `active` / `activeCount` / `history` 里对 `TABLE_FACT` 的查询（改写新表）。**保留**的：向量 7 个方法（`putVector` / `markVectorIndexed` / `markVectorFailed` / `dropVector` / `pendingVectorFacts` / `enqueueAllActive` / `vectorText`）与私有的 `enqueueVector`——它们读写的是 `zhuli_fact_vec` / `zhuli_vec_queue`，属于 B 组长表，只是 JOIN 的源表从 `zhuli_fact` 换成 `zhuli_memory_item`。

#### 4.8.5 `zhuli_entity` 的写入停止（显式记录，不静默）

`MemoryRepo.ingest()` 第 50 行有 `d.entityKind?.let { upsertEntity(it, subject, now) }`，写的是 `zhuli_entity`。重写后 `MemoryRepo` 不再写旧表，这行随之删除。影响评估：

| 项 | 结论 |
|---|---|
| 有读方吗 | **没有**。`grep -rn "TABLE_ENTITY" app/src/main` 只有 `InboxDatabase` 的 DDL 与 `MemoryRepo` 的写入 |
| 有 UI 依赖吗 | 没有。`ZhuliMemoryPage` 不读实体表 |
| 会丢数据吗 | 存量行保留（表不删、行不动）；新数据不再进旧实体表 |
| 新实体去哪 | C1 不写 `zhuli_memory_entity`（C0.2 §2.5：只建不写，写入方在 C2）。`FactDraft.entityKind` 字段保留，C2 接手 |
| 怎么防止"以为还在写" | 本表 + §4.8.4 的 `grep` 自检 + `T-M40` 的结构断言（表存在但 0 行不是失败，**有行也不失败**，因为它可能有历史数据） |

> 这是 C1-B 唯一一处"功能静默减弱"：实体索引停止更新。它必须写进 C1-B5 报告，不能等用户发现"实体表怎么不长了"。C0.2 已把实体写入排到 C2，所以这不是范围外改动，而是按设计执行。

#### 4.8.6 D-7 的 6 条回归测试（编号与归属）

| # | 用例 | 归属文件 | 断言要点 |
|---|---|---|---|
| `T-M40` | 新装库的初始结构 | `MemorySchemaTest` | `zhuli_fact` 与 `zhuli_fact_v1` **都不存在**；7 张新表 + 15 索引齐全；`user_version=17` |
| `T-M41` | v16→v17 后关闭并重开 | `MemoryMigratorTest` | 连开 3 次；`zhuli_fact` 始终不存在；`zhuli_fact_v1` 行数不变；`user_version` 始终 17 |
| `T-M42` | 旧写路径 fail-loud | `MemoryMigratorTest` | 迁移后直写 `INSERT INTO zhuli_fact` 抛 `SQLiteException`（消息含 `no such table`），不静默成功 |
| `T-M43` | `onOpen` 幂等 | `MemoryMigratorTest` | 连开 3 次；`sqlite_master` 的对象数与名字集合不变；`COUNT(*)` 不变；无异常 |
| `T-M44` | 重开后新记忆数据完整 | `MemoryMigratorTest` | 重开后逐列抽检 12 个关键列（id/subject/value/status/valid_from/valid_until/time_confidence/invalidated_at/dedupe_key/scope_type/extractor/revision）+ 向量行 + evidence 行数 |
| `T-M45` | v16 库缺 `zhuli_fact` 时升级失败且不建空表 | `MemoryMigratorTest` | 抛异常 → 回滚 → `user_version=16` → `zhuli_fact` 仍不存在（**没有**被补建成空表） |

> `T-M40` 放在 `MemorySchemaTest` 而不是 `MemoryMigratorTest`：它测的是 `onCreate`（新装）路径，与迁移无关；`T-M41`–`T-M45` 都要先造 v16 库，归 `MemoryMigratorTest`。



---

## 5. v16 → v17 迁移详细方案

### 5.1 触发点与事务边界（v3 重写）

**先纠正 v2 的一处技术错误**：v2 写"嵌套事务 T1（Android 用 savepoint 实现）"——**这是错的**。AOSP `SQLiteSession.beginTransactionUnchecked()` 只在 `mTransactionStack == null` 时真正执行 `BEGIN EXCLUSIVE`；嵌套调用只做链表记账，**不产生 SAVEPOINT**。嵌套层 `setTransactionSuccessful()` 也不会提交，只有最外层 `endTransaction()` 才 COMMIT；任一子层未标记成功会把 `mChildFailed` 传给外层，导致整体 ROLLBACK。

**实际时序（已核对 AOSP 源码语义）**：

| # | 动作 | 是否在 helper 的 T0 内 | 失败后的状态 |
|---|---|---|---|
| 1 | 调用方 `db.writableDatabase` → `getDatabaseLocked()` 读 `PRAGMA user_version`，发现 `16 ≠ 17` | — | 读版本失败 → 异常，库不动 |
| 2 | helper `db.beginTransaction()`（`BEGIN EXCLUSIVE`，delete 模式） | **T0 开始** | `BEGIN` 失败 → 异常，库不动 |
| 3 | `onUpgrade` 第一行：`backupBeforeMigration(16)` 文件拷贝 | **是**（T0 内，**尚无任何写语句**） | 备份失败 → 记 `lastBackupError`，**继续迁移**（备份不是前置条件，现有语义） |
| 4 | `PRAGMA journal_mode` 断言（§5.12） | 是 | 抛 → T0 回滚 → 库仍 v16 |
| 5 | `if (oldVersion < 15) { createCoreTablesV15; createFactTableV15 }` | 是 | 抛 → 回滚 |
| 6 | `if (oldVersion < 17) { MemorySchema.createAll(withUnique=false); MemoryMigrator.migrate() }` | 是 | 抛 → 回滚 |
| 7 | helper `db.setVersion(17)` | 是（**同一事务**） | 抛 → 回滚（版本号也回滚） |
| 8 | helper `setTransactionSuccessful()` → `endTransaction()` → COMMIT | **T0 结束** | 进程崩溃 → SQLite rollback journal 回滚，库仍 v16 |
| 9 | helper `onOpen(db)` → `MemorySchema.ensureAll` | **否**（已提交，新事务外） | 失败 → 库已是 17，结构可能缺；下次打开自愈（`T-M43`） |

三点必须照此理解，否则会写出错的测试：

1. **第 3 步是唯一合法的备份点**。`onUpgrade` 的第一条语句、T0 已开但未写——此时主库文件内容仍等于"上次提交后的完整状态"。一旦执行过任何 DDL/DML，rollback journal 模式下主库页已被就地改写，再拷就是"半新半旧"的混合文件。
2. **第 7 步的版本号在事务内**。所以迁移失败时 `user_version` 不会留在 17，这是 `T-M08` / `T-M45` 断言的依据。
3. **`onUpgrade` 里的 `db.beginTransaction()` 是纯嵌套**，不产生新的 BEGIN。现有 `began` 变量（失败则降级为"无事务"）在 T0 内**永远为 true**，保留它无害但也不再是保障；真正的事务保障来自 helper。

`MemoryMigrator.migrate()` **不自己开事务**，它假定被 `onUpgrade` 的 T0 包住。理由：C0.2 §8.9 的第 12 步（改名）与第 2 步（插数据）必须在同一事务，否则会出现"数据插了但表没改名"的半迁移。它也不得调用 `setTransactionSuccessful()`——那只属于最外层。

> **为什么不把备份前移到 helper 之前**：那需要 `override getWritableDatabase()`，而 `InboxDatabase` 是 Kotlin `final class`（`InboxDatabase.kt:19`），必须先把类改成 `open`。改类声明会影响 30+ 处 `InboxDatabase(ctx)` 构造点的可见性语义，属于"为了修一个小问题扩大生产代码范围"（§8.5 明令禁止）。**当前方案用断言把 WAL 挡在门外，而不是用重构去兼容 WAL**——见 §5.12。

### 5.2 迁移步骤（13 个动作 + 1 条已取消的动作）

**步骤数口径（v3 修正）**：C0.2 §8.9 是 **13 步**（第 13 步为 `PRAGMA user_version = 17`）。本方案：
- **新增**第 0 步（前置断言）→ 动作步 1 个；
- **保留** C0.2 的 1–12 步 → 动作步 12 个；
- **取消** C0.2 的第 13 步（`X-6`：交给 helper）→ 不是动作，是"明确不做"。

所以下表共 **14 行**，其中**动作 13 个**（第 0 步 + 第 1–12 步），第 13 行是撤销说明。v2 标题写"十三个步骤"但表里有 14 行，两者都不算错、但口径没写清——v3 明确为"13 个动作 + 1 行撤销说明"。

| 步 | 动作 | 实现 | 失败行为 |
|---|---|---|---|
| 0 | 前置断言 | ① `journal_mode` 属于回滚日志族（`delete`/`truncate`/`persist`），**`wal` 直接拒绝**（§5.12）；② `newVersion == 17`；③ **`oldVersion` 在 `1..16`**（`0` 不该进这里，`>17` 是降级）；④ 源表存在性**按 `oldVersion` 分支判定**（见下） | 抛 → 回滚 |
| 1 | 建 7 张表 + 12 个普通索引（**不含** 3 个唯一索引） | `MemorySchema.createAll(db, withUnique=false)` | 抛 → 回滚 |
| 2 | 逐行迁移 item / source / evidence | `migrateRows()`，见 §5.3 | 单行失败：记 `failed++` 并**继续**；末尾 `failed>0` 即抛（§5.9） |
| 3 | 替代链 → relation；断链清理 | `migrateSupersedeChain()` | 抛 → 回滚 |
| 4 | 重复组合并 | `mergeDuplicates()`，见 §5.6 | 抛 → 回滚 |
| 5 | 重算 `times_reinforced` | 一条 UPDATE（见下） | 抛 → 回滚 |
| 6 | 向量校验与清理 | `verifyVectors()`，见 §5.8 | 逐条删无效项，不抛 |
| 7 | 建 FTS4 虚拟表 | `MemorySchema.createFtsTable(db)` | 失败 → `ftsMode=NONE`，**不中止迁移** |
| 8 | 一次性灌 FTS | `INSERT INTO … SELECT … WHERE status<>'DELETED'` | 失败 → `ftsMode=NONE` |
| 9 | 建 3 个触发器 | `MemorySchema.createFtsTriggers(db)` | 失败 → `ftsMode=APP_SYNC`（探测决定） |
| 10 | 建 `idx_mi_dedupe` | 建失败时打印前 5 个重复键（**只打印键的哈希前 8 位，不打印原文**） | 抛 → 回滚 |
| 11 | 对齐 `sqlite_sequence` | `INSERT OR REPLACE INTO sqlite_sequence(name,seq) …`（见 §5.7） | 抛 → 回滚 |
| 12 | `ALTER TABLE zhuli_fact RENAME TO zhuli_fact_v1` | — | 抛 → 回滚 |
| 13 | **不做** `PRAGMA user_version = 17` | 由 helper 的 `db.setVersion(17)` 在同一事务内完成（X-6） | — |

**第 0 步的源表断言（P0-7，v3 重写）**：

v2 写"`user_version=16`"，这句**必须删掉**。原因：`upgradeSteps()` 支持 `oldVersion` 从 1 到 16 的全部路径，v14/v15 库升级到 v17 时 `user_version` 是 **14 或 15**，不是 16。若照 v2 断言，所有 v14/v15 用户升级必失败。

正确做法是**按 `oldVersion` 分支**，而不是查 `PRAGMA`：

| `oldVersion` | 源表 `zhuli_fact` 是否应存在 | 断言 | 依据 |
|---|---|---|---|
| `1..13` | **进来时不存在**，由 helper 的 `upgradeSteps` 在 `oldVersion<15` 分支用 `createFactTableV15` 补建（§5.1 时序表第 5 步，**不是本节第 5 步**） | 在 `createFactTableV15` 返回**之后**断言"存在且 `PRAGMA table_info` 为 16 列" | v15 才引入这张表 |
| `14` | **应该不存在**（v14 没有 fact 表），同上补建 | 同上 | v15 引入 |
| `15` / `16` | **进来时就应存在** | 在第 0 步**立即**断言"存在"；缺失即损坏 → 抛，**不补建**（`T-M45`） | v15/v16 一定有 |
| `0` | 不该进 `onUpgrade`（走 `onCreate`） | 抛 `IllegalStateException` | helper 语义 |
| `> 17` | 降级，不该进 `onUpgrade` | 抛 `IllegalStateException` | `onDowngrade` 负责 |

一句话：**"缺表要不要补"由 `oldVersion < 15` 决定，"缺表算不算损坏"由 `oldVersion >= 15` 决定**。两者不是同一条断言，v2 把它们揉成了一句"`zhuli_fact` 存在"，于是 v14/v15 必然误判。

> `PRAGMA user_version` 在整个迁移过程中**只读不写**；它由 helper 在 `onUpgrade` 返回后写（§5.1 第 7 步）。迁移代码不依赖它，只依赖 `oldVersion` 参数——这是 Android 的正式契约。

第 5 步的 SQL（C0.2 §5.2 的派生定义）：

```sql
UPDATE zhuli_memory_item SET times_reinforced = (
  SELECT MAX(0, COUNT(DISTINCT s.independent_key) - 1)
  FROM zhuli_memory_evidence e JOIN zhuli_memory_source s ON s.id = e.source_id
  WHERE e.memory_id = zhuli_memory_item.id
);
```

### 5.3 逐字段迁移（代码级）

`MemoryMigrator.migrateRow(c: Cursor): Row?` 的核心，一列不漏：

| 目标列 | 取值 | 失败/边界 |
|---|---|---|
| `id` | `c.getLong(id)` | 显式写入，保真 |
| `memory_type` | `if (legacyKind == "PREFERENCE") PREFERENCE else FACT` | — |
| `subject` | `c.str(subject).trim()` | 空 → `skipped++`，返回 null |
| `subject_key` | `MemoryText.norm(subject)` | 空（全标点）→ 用 `subject.lowercase()` 兜底，仍空则跳过 |
| `predicate` | `trim()` | 空 → 保留空串（C0.2 不要求跳过；空谓词仍是"属性名缺失"的事实，但**必须**参与 dedupe，不能丢） |
| `qualifier_key` | `""` | 常量 |
| `value` | `trim()` | 空 → `skipped++` |
| `content` | `"$subject｜$predicate｜$value"` | 与 `Fact.display` 逐字一致 |
| `normalized_content` | `norm(content)` | — |
| `dedupe_key` | `MemoryText.dedupeKey(GLOBAL, "", memoryType, subjectKey, predicate, "", norm(value), bucket)` | `bucket` 见 §5.4 |
| `slot_hash` | `MemoryText.slotHash(GLOBAL, "", memoryType, subjectKey, predicate, "")` | 与 value 无关 |
| `search_text` | `MemoryText.searchText(content)` | 2 字滑窗 |
| `status` | 4→6 映射（见下方"状态映射的三种输入"） | **未知值（含 `NULL` / 空串）→ `CANDIDATE` + `unknown_status++`**（D-6 第 3/4 条：不得 fallback `ACTIVE`） |
| `cardinality` | `legacyKind in (EXPRESS,ORDER,MONEY,SCHEDULE)` → `MULTI` else `SINGLE` | 另：`payload_json.cardinality` 存在则覆盖 |
| `confidence` | `min(旧值, basis 上限)` | 被降则 `clamped++` |
| `confidence_basis` | 5 条映射（见下表） | — |
| `source_id` | `evidence` 的 PRIMARY 的 source id | — |
| `scope_type` / `scope_id` | `'GLOBAL'` / `''` | 常量 |
| `sensitivity` | `MemoryGuard.classify(ALL_TEXT)`：SECRET → 整行隔离；SENSITIVE → 1；否则 0。**`ALL_TEXT` = `subject`+`predicate`+`value`+`source_text` 四字段拼接**（§2.7 / P1-3） | SECRET → `quarantined++`，**不插 item、不插 source、不插 evidence** |
| `event_start` / `event_end` | `0` | 旧表没有 |
| `valid_from` | 原值（0 保留 0） | **不设 `time_confidence`** |
| `valid_until` | **`0`** | 常量，不读 `valid_to` |
| `observed_at` | 原值；0→`created_at`；仍 0→`now` | 三个来源按序 |
| `recorded_at` | `created_at` | — |
| `updated_at` | 原值；0→`created_at` | — |
| `last_confirmed_at` | `basis==USER_CONFIRMED ? updated_at : 0` | — |
| `invalidated_at` | `SUPERSEDED → (valid_to ?: updated_at)`；`DELETED → updated_at`；其它 0 | 旧 `valid_to` 的**唯一**去处 |
| `time_confidence` | **`'NONE'`** | 常量，不按条件分支 |
| `superseded_by` | 原值（目标不存在则清 0，§5.7） | — |
| `revision` | `1` | 常量 |
| `extractor` | `'import:legacy'` | 常量，自检全靠它定位迁移行 |
| `legacy_kind` | `FactKind.name` | 保真 |
| `payload_json` | `{"migrated_from":<id>}`；**仅未知状态行**追加 `"legacy_status":"<redact 后截 32 字>"` | 便于回溯；**不放任何原文** |
| `deleted_at` | `status=DELETED ? updated_at : 0` | — |
| `pre_delete_status` | `''` | 迁移不产生（§5.4 禁止 4） |
| `times_reinforced` / `times_retrieved` / `last_used_at` | `0`（前者第 5 步重算） | — |

**basis 映射（逐字照 C0.2 §8.3）**

| `source_type` | 条件 | `confidence_basis` | 上限 |
|---|---|---|---|
| `USER` | `source_text` 以 `"用户纠正："` 开头 | `USER_CONFIRMED` | 1.00 |
| `USER` | 其它 | `USER_STATED` | 0.95 |
| `INBOX` | — | `RULE_EXTRACTED` | 0.85 |
| `INFERRED` | — | `MODEL_INFERRED` | **0.65** |
| `SYSTEM` | — | `EXTERNAL_VERIFIED` | 0.90 |

#### 5.3.1 状态映射的三种输入（P1-1：必须覆盖 `NULL`）

旧表 `zhuli_fact.status` 是 `TEXT NOT NULL DEFAULT 'ACTIVE'`（`InboxDatabase.kt:520` 附近），理论上不会是 `NULL`。但**迁移代码不能依赖"理论上"**：手工改库、历史 beta 包、`ALTER TABLE` 补列都可能留下 `NULL`。更关键的是**统计口径**——SQL 的三值逻辑会让 `NULL` 从"未知值计数"里漏掉：

```sql
-- 错：NULL 不满足任何比较，COUNT 不出来
SELECT COUNT(*) FROM zhuli_fact
WHERE status NOT IN ('ACTIVE','UNCONFIRMED','SUPERSEDED','DELETED');
-- 对：显式判 NULL
SELECT COUNT(*) FROM zhuli_fact
WHERE status IS NULL
   OR TRIM(status) NOT IN ('ACTIVE','UNCONFIRMED','SUPERSEDED','DELETED')
   OR TRIM(status) = '';
```

映射函数必须显式覆盖三种输入，**不允许 `when (raw)` 只写四个分支**：

| 输入 | 映射为 | `unknown_status` | `payload_json` |
|---|---|---|---|
| `"ACTIVE"` / `"UNCONFIRMED"` / `"SUPERSEDED"` / `"DELETED"` | 各自对应的六态值 | 不计数 | 只写 `migrated_from` |
| `"EXPIRED"` / `"REJECTED"`（v16 不该有，但可能被手工写入） | `EXPIRED` / `REJECTED` 原样 | 不计数（这是**合法**状态，不是未知） | 只写 `migrated_from` |
| `NULL` | `CANDIDATE` | **+1** | 追加 `"legacy_status":"<NULL>"` |
| `""` / 纯空白 | `CANDIDATE` | **+1** | 追加 `"legacy_status":"<EMPTY>"` |
| 其它字符串（含大小写不符、尾空格、未来值） | `CANDIDATE` | **+1** | 追加 `"legacy_status":"<redact 后截 32 字>"` |

三条硬约束：

1. **`unknown_status` 的计数必须等于 §5.9 第 6b 条那条 SQL 的结果**，两边都用"`IS NULL` + `TRIM` + 空串"三条件。只改一边就会在 `T-X08` 上出现"计数 1、SQL 查 2"的假失败。
2. **`legacy_status` 只对未知值写**，合法值不写——否则 `T-M36` 的第 6 条断言（"含 `legacy_status` 的行必须 `CANDIDATE`"）会把正常行也扫进来。
3. **`legacy_status` 的值必须过 `SensitiveGuard.redact` 再截 32 字**。它进了 `payload_json`，而 `payload_json` 会被 `selfCheck` 和调试日志读取；一个被手工写成手机号的状态值不能原样落库。

#### 5.3.2 SECRET 检查的位置与顺序（P1-3：全字段 + 先判后写）

v2 写"分级只看 `content`"（§2.7），但一条 `zhuli_fact` 有**四个**文本字段：`subject` / `predicate` / `value` / `source_text`。SECRET 完全可能只出现在 `source_text` 里（短信原文含卡号），而 `content` 是 `subject｜predicate｜value` 的拼接——只查 `content` 会**漏掉整条短信里的卡号**。

**检查对象（v3 修正）**：

```kotlin
// MemoryGuard.classifyForMigration(...) —— 迁移专用，与运行时 classify 分开
fun classifyForMigration(
    subject: String?, predicate: String?, value: String?, sourceText: String?
): Sensitivity {
    // 任一字段命中即 SECRET；四段各自判，不做拼接（拼接会让跨字段的假阳性变多，
    // 但更重要的是：拼接后无法定位是哪一段命中，日志里就没法只记"哪个字段"）
    val fields = listOf(subject, predicate, value, sourceText)
    if (fields.any { SensitiveGuard.reason(it ?: "") == Reason.SECRET }) return Sensitivity.SECRET
    if (fields.any { SensitiveGuard.reason(it ?: "") != null }) return Sensitivity.SENSITIVE
    return Sensitivity.NORMAL
}
```

**顺序（这是 P1-3 的第二个要求）**：

```
migrateRow(cursor) 的第 1 步就是 classifyForMigration —— 在任何 INSERT 之前
  ├─ SECRET → quarantined++，return null（不写 item / source / evidence / relation / revision）
  └─ 其它   → 继续算 content / dedupe_key / excerpt
                └─ 第 2 步才写 source
                └─ 第 3 步写 item（带 source_id 回填）
                └─ 第 4 步写 evidence
```

**为什么顺序重要**：`source` 表的 `excerpt` 字段是脱敏后的文本，但它**仍然是原文的摘要**。如果先插 `source` 再发现是 SECRET，事务里虽然能删，但：
- `source` 行会占用 `idx_ms_dedupe` 的唯一键，导致同一条来源在后续行里被"复用"，SEcret 摘要留在库里；
- 一旦 `migrateRow` 里某处抛异常被吞（`failed++` 继续），这行 source 就成了孤儿。

所以**拒绝必须发生在任何关联表写入之前**，不是一个"事后清理"的问题。

> **`excerpt` 的残留风险不在 C1 解决**：`SensitiveGuard.redact` 是正则脱敏，不保证 100%。C1 的承诺是"SECRET 整行不迁入"；SENSITIVE 行的 `excerpt` 仍可能含部分信息，这条写进 §19.3 的 U-12（与 C0.2 §14.2 阶段 3 的备份清理同一批处理）。

### 5.4 四条禁止（写错任何一条就等于让数据说谎）

| # | 禁止 | 为什么 | 谁盯 |
|---|---|---|---|
| 1 | **不得**把 `valid_from>0` 标成 `DAY`/`MONTH`/`YEAR` | 旧 `valid_from` 全部由 `ingest()`/`correct()` 写成 `now`（`MemoryRepo.kt:79/114`），标精度等于伪造 | `T-M36` |
| 2 | **不得**把 `valid_to` 写进 `valid_until` | 旧 `valid_to` 是"我们何时不再相信"（`MemoryRepo.kt:431` 写 `now`），是世界线的伪终点 | `T-M37` |
| 3 | **不得**因 `valid_from` 被清零就让旧事实变成"当前值" | `LB` 退回 `observed_at` 后，一条本已过期的值会重新生效。规则：清零只发生在 R0（运行时），迁移**不清零**（保留原值 + `NONE`），两者不冲突 | `T-M36` / `T-T40` |
| 4 | **不得**为迁移行写 `pre_delete_status` | 迁移行从没被运行时删过；写了会让"恢复回原状态"误以为知道原状态 | `T-M36` 的伴生断言 / 6.6 #10 |

第 3 条的澄清（容易被误解）：迁移**保留** `valid_from` 原值（不清零），因为清零后 `LB` 同样退回 `observed_at`，行为完全一致，但保留原值可让 UI 显示"系统记录于…"。真正危险的是"保留原值 + 标 `DAY`"，那会让 `LB` 直接取 `valid_from` 并宣称它是事实起点。所以 #1 是主约束，#3 是它的防护性说明。

### 5.5 来源与证据（伪码）

```kotlin
// 对每一行 fact —— 顺序不可调换：classify 必须在任何 INSERT 之前（§5.3.2）
val sensitivity = MemoryGuard.classifyForMigration(subject, predicate, value, sourceText)
if (sensitivity == Sensitivity.SECRET) { report.quarantined++; return null }   // 零写入

val excerpt = MemoryGuard.excerpt(sourceText.ifBlank { content })   // redact + ≤120
val hash    = MemoryText.sha256Hex(MemoryText.norm(sourceText.ifBlank { content }))
val refId   = c.getLong(sourceId)
val key     = SourceKey(sourceType, refTable = "zhuli_fact", refId = refId.toString(),
                        refSubId = "", contentHash = hash)
val sourceId = sourceCache.getOrPut(key) {
    // INSERT OR IGNORE 后 SELECT id（唯一索引 idx_ms_dedupe 保证幂等）
}
val independent = if (refId != 0L) "$sourceType|zhuli_fact|$refId" else "$sourceType|-|$hash"
INSERT evidence(memory_id = factId, source_id = sourceId, role = 'PRIMARY',
                first_seen_at = observedAt, last_seen_at = observedAt, seen_count = 1)
UPDATE item SET source_id = sourceId WHERE id = factId
```

三条约束：

1. **同一 `source_id` 抽出的多条事实共享 `independent_key`，但各自有 source 行**（`content_hash` 不同）——这是"一条来源产生多条记忆"的正确表达，C0.2 §8.4 已说明。
2. `source_text` 为空时用 `content` 兜底，保证 `excerpt` 不为空（UI 要显示"凭什么"）。
3. `trust`：`USER → 3`，其它 `1`。

### 5.6 重复合并（只合并真重复）

触发条件（建唯一索引**之前**执行）：

```sql
SELECT dedupe_key, COUNT(*) FROM zhuli_memory_item
WHERE status IN ('ACTIVE','CANDIDATE') GROUP BY dedupe_key HAVING COUNT(*)>1
```

每组按 C0.2 §8.8 处理。实现要点：

| 项 | 实现 | 容易写错的地方 |
|---|---|---|
| 保留行 | `rank(basis)` desc → `last_confirmed_at` desc → `observed_at` asc → `id` asc | 别用 `id desc`（会让通知盖过用户陈述） |
| `valid_from` | 组内**全为正**取 `min`；**任一为 0** 取 `0` | "用已知起点代表未知起点"是错的 |
| `valid_until` | `0` | 迁移行本来就全是 0，别"合并区间" |
| `time_confidence` | `NONE` | 合并不产生新时间知识 |
| `observed_at` | `min` | — |
| `last_confirmed_at` | `max` | — |
| 被合并行 | `status=SUPERSEDED`, `superseded_by=保留行`, `invalidated_at=now` | 别写成 `DELETED` |
| 关系 | `relation(保留, 被合并, 'SAME_AS')` | 方向：from 是保留行 |
| 证据 | 复制被合并行的 evidence 指向保留行，`role='SUPPORTING'`；**被合并行的 evidence 原样保留** | C0.2 §9.7：初稿"搬走再删"会让 6.6 #7 不成立 |
| revision | 保留行写一条 `MERGE` | 迁移里唯一需要 revision 的地方 |

### 5.7 ID 保真与断链

```kotlin
// 插入用显式 id，绝不让 AUTOINCREMENT 分配
db.execSQL("INSERT INTO zhuli_memory_item(id, …) VALUES(?, …)", args)
```

**`sqlite_sequence` 的处理（P1-8，v3 重写）**

v2 写"显式插大 id 时 SQLite 会自动跟进，但仍显式对齐一次"。结论对，**前提没写，写法也不稳**。三点必须补齐：

**① 前提：`sqlite_sequence` 在迁移时一定存在。** SQLite 文档：只要库里创建过任何带 `AUTOINCREMENT` 的普通表，`sqlite_sequence` 就会被自动创建（可能是空表）。本库有 9 张 `AUTOINCREMENT` 表（`inbox` / `reminders` / `ledger` / `chat` / `zhuli_entity` / `zhuli_fact` / `zhuli_agent_step` / `skill` / `cloud_log`，见 `InboxDatabase.kt` 的 9 处 `AUTOINCREMENT`），其中 `inbox` 在 v1 就存在。所以：
- **v1–v16 的任何真实库**：`sqlite_sequence` 必然存在（`inbox` 建表时就创建了）；
- **全新空库**：不该走到迁移（走 `onCreate`）。

因此"表不存在"不是需要处理的场景。但迁移代码仍要**先 `tableExists(db, "sqlite_sequence")`** 再操作，因为 `T-M45`（人造只含 `inbox` 的 v16 库）里它存在、而某些测试夹具可能手工建表绕过 `onCreate`。不存在就跳过这一步，不抛。

**② 写法：用 `INSERT OR REPLACE`，不用 `UPDATE`。** 原因：迁移**可能一行都没插入**（全被 `skipped` / `quarantined`）。此时 `sqlite_sequence` 里**没有** `zhuli_memory_item` 这一行，`UPDATE` 影响 0 行、静默不生效；后续第一条 INSERT 仍从 1 开始——这在"空迁移"下恰好正确，但在"插入过但 sequence 行被外部删除"的损坏库里会撞 id。

```sql
-- 正确：无行则插入，有行则更新
INSERT OR REPLACE INTO sqlite_sequence(name, seq)
SELECT 'zhuli_memory_item', COALESCE(MAX(id), 0) FROM zhuli_memory_item;
```

**③ 必须验证"后续自动 ID 不冲突"，而不只是对齐。** 对齐本身不能证明没冲突——要断言"下一条 INSERT 拿到的 id > `MAX(迁移 id)`"。这一步放在 `MemoryMigratorTest` 的 `T-M46`（v3 新增，见 §10.2）：

```
迁移后：INSERT INTO zhuli_memory_item(subject, …) VALUES('新行', …)   -- 不指定 id
断言：  新 id == MAX(迁移 id) + 1
断言：  SELECT seq FROM sqlite_sequence WHERE name='zhuli_memory_item' >= MAX(id)
```

> **为什么不在迁移里"预留一段 id 区间"**：那会让 ID 不连续、并给 C2 的 `MAX(id)` 类查询制造空洞。AUTOINCREMENT 的语义就是"单调递增、不复用"，`INSERT OR REPLACE` 对齐后它自然满足。

断链：

```kotlin
// superseded_by > 0 但目标行不存在（被 SECRET 隔离或空值跳过）
UPDATE zhuli_memory_item SET superseded_by = 0 WHERE id = ?   // 只清指针
// 不动 invalidated_at：清了会让一条已被替代的记忆在信念线里复活
```

### 5.8 向量校验

```
for each row in zhuli_fact_vec:
    space 能解析出 dim（"provider|dim"）
    且 LENGTH(vec) == dim*4
    且 zhuli_memory_item 存在该 fact_id
  → 保留
否则:
    DELETE FROM zhuli_fact_vec WHERE fact_id = ?
    INSERT OR IGNORE INTO zhuli_vec_queue(fact_id, attempts=0, at=now)
```

**不做重新 embed**：迁移不跑模型（C0.2 §8.7）。换 space 的重建由 `MemoryIndexer.drainOnce` 的 existing 逻辑处理（`MemoryIndexer.kt:169-174`）。

### 5.9 逐行处置账本与数量闭合（P1-2，v3 重写）

v2 的恒等式是 `src == inserted + quarantined + skipped`。**它是漏的**：它没有给"插入失败但仍继续"的行留位置（v2 §5.2 第 2 步明确允许 `failed`），也没有区分"被合并"的行去了哪里。结果是：一个 20k 库只要有 1 行插入失败，恒等式要么不成立（迁移失败），要么被人为调平（数据说谎）。

**每条源记录必须有且只有一个终态**。账本如下：

| 终态 | 报告字段 | 目标表里的结果 | 能否与其它终态重叠 |
|---|---|---|---|
| `INSERTED` | `inserted` | `zhuli_memory_item` 1 行，`extractor='import:legacy'` | 否 |
| `MERGED` | `merged`（`⊆ inserted`） | 行**在**表里，但 `status='SUPERSEDED'`、`superseded_by=保留行`、`invalidated_at=now` | 是——它**同时**计入 `inserted`，`merged` 只是子集标记 |
| `QUARANTINED` | `quarantined` | 无新行（原文留在 `zhuli_fact_v1`） | 否 |
| `SKIPPED_NO_SUBJECT` | `skippedNoSubject` | 无新行 | 否 |
| `SKIPPED_NO_VALUE` | `skippedNoValue` | 无新行 | 否 |
| `FAILED` | `failed` | 无新行 | 否 |

**主恒等式**（`merged` 不参与，它是 `inserted` 的子集）：

```
src == inserted + quarantined + skippedNoSubject + skippedNoValue + failed
且  merged <= inserted
且  failed == 0          ← 提交的附加条件，见下
```

**为什么 `failed` 要计入主恒等式、又要求为 0**：
- 计入，是为了让"失败时"的报错能说清"少了几条、少在哪一类"，而不是只报"不闭合"；
- 要求为 0，是因为**部分迁移比不迁移更危险**：用户看到 19,999 条，少的那 1 条无声消失。真正的处理是抛异常回滚，让下次重试（迁移是幂等的，因为 `zhuli_fact` 未改名、`zhuli_memory_item` 因回滚而不存在）。
- 所以顺序是：先 `check(闭合)`，再 `check(failed == 0)`，两个都过才继续第 10 步。

**为什么 `merged` 是子集而不是独立终态**：被合并的行**确实插进了新表**（`status='SUPERSEDED'`），它保留了 `id`、`superseded_by` 和原有 `evidence`（§5.6 的"被合并行 PRIMARY 仍在"）。如果把它算成"未插入"，`T-M06` 会断言"合并后行数减少"，与"ID 保真"矛盾。

**三类 `skipped` 必须分开计数**：v2 只有一个 `skipped`，导致 `T-M05`（空 subject / 空 value）无法分辨"是哪一类空"。分开后，`T-M05` 可以分别断言 `skippedNoSubject=1`、`skippedNoValue=1`。

**逐行账本的实现约束**：

```kotlin
// 每处理一行，恰好触发且只触发一个分支；用一个 enum 返回值，不用多个 boolean
sealed interface RowOutcome {
    data class Inserted(val id: Long) : RowOutcome
    data class Merged(val id: Long, val keeperId: Long) : RowOutcome
    data object Quarantined : RowOutcome
    data object SkippedNoSubject : RowOutcome
    data object SkippedNoValue : RowOutcome
    data class Failed(val error: String) : RowOutcome
}
// migrateRow 只返回 RowOutcome；计数与日志在调用方，避免"某条路径忘了 ++"
```

这条约束是给 review 用的：**只要 `migrateRow` 里出现第二个 `return null`，就说明有人绕过了账本**。用 sealed 类型 + `when` 穷举，编译器会挡住。

迁移末尾必须自证：

```kotlin
val src = count("SELECT COUNT(*) FROM zhuli_fact_v1")        // 改名后
check(src == report.inserted + report.quarantined +
             report.skippedNoSubject + report.skippedNoValue + report.failed) {
    "数量不闭合：$src != " + report.ledgerLine()
}
check(report.merged <= report.inserted) { "merged 不是 inserted 的子集" }
check(report.failed == 0) { "有 ${report.failed} 行插入失败，拒绝提交半迁移" }
```

5 条时间不变量（C0.2 §8.1，**必须带 `extractor`**）：

| # | 断言 SQL | 期望 |
|---|---|---|
| 1 | `COUNT(*) WHERE extractor='import:legacy' AND time_confidence<>'NONE'` | 0 |
| 2 | `COUNT(*) WHERE extractor='import:legacy' AND valid_until<>0` | 0 |
| 3 | `COUNT(*) WHERE extractor='import:legacy' AND valid_from<>0 AND valid_from<>observed_at` | **告警不中止** |
| 4 | `COUNT(*) WHERE status='SUPERSEDED' AND invalidated_at=0` | 0 |
| 5 | `COUNT(*) WHERE extractor='import:legacy' AND status='DELETED' AND pre_delete_status<>''` | 0 |

第 6 条（**D-6 新增**，状态不变量）：

| # | 断言 SQL | 期望 |
|---|---|---|
| 6 | `COUNT(*) WHERE extractor='import:legacy' AND payload_json LIKE '%"legacy_status"%' AND status<>'CANDIDATE'` | 0 |
| 6b | `report.unknownStatus` 与旧库的未知状态行数（**含 `NULL`**，见下方 SQL） | 相等 |

第 6b 条的 SQL 必须写全三条件，否则 `NULL` 会被三值逻辑漏掉（§5.3.1）：

```sql
SELECT COUNT(*) FROM zhuli_fact_v1
WHERE status IS NULL
   OR TRIM(status) NOT IN ('ACTIVE','UNCONFIRMED','SUPERSEDED','DELETED','EXPIRED','REJECTED')
   OR TRIM(status) = '';
```

> 注意第 6b 条查的是 `zhuli_fact_v1`（改名后的旧表），不是新表——因为"未知状态有多少"是旧数据的属性，新表里它们已经统一成 `CANDIDATE` 了。

第 6 条的含义：迁移遇到无法识别的状态值时，**只能落到 `CANDIDATE`**（非当前、不可自动激活），任何落到 `ACTIVE` / `SUPERSEDED` / `EXPIRED` / `REJECTED` / `DELETED` 的未知状态行都是缺陷。`legacy_status` 是唯一的定位标记，C3 可以据此人工核对这批行。

第 3 条为什么"告警不中止"：它检测的是"有没有未知写入方动过 `valid_from`"（C0.2 §8.1 的注）。中止会让一个**未来**的良性改动（有人加了用户时间入口）变成升级失败；告警能让它被发现，又不会拦住用户。


### 5.10 日志规格（不含敏感原文）

迁移日志是一行 JSON（`Log.i(TAG, ...)`），字段固定：

```json
{"phase":"v16->v17","src":2195,"inserted":2190,"merged":7,
 "quarantined":3,"skipped_no_subject":1,"skipped_no_value":1,"failed":0,
 "sources":2180,"evidence":2190,"vec_kept":1,"vec_dropped":0,
 "clamped":12,"vf_mismatch":0,"unknown_status":0,"fts_mode":"TRIGGER","ms":1840}
```

字段与 §5.9 账本一一对应：`inserted` / `merged` / `quarantined` / `skipped_no_subject` / `skipped_no_value` / `failed`。**六个字段全部必填**（为 0 也要写），这样"某类没出现"和"某类忘了统计"在日志里长得不一样。

**禁止出现的字段**：`subject` / `value` / `content` / `source_text` / 行 `id` / `dedupe_key`。
理由：`id` 可反查旧表；内容本身就是敏感数据。计数足够回答"有没有、有几条"（C0.2 §14.2）。

失败时额外一条：`{"phase":"row","error":"<异常类名>: <message 前 80 字>"}`，且 message 必须过 `SensitiveGuard.redact`——SQLite 的异常消息会带参数值。

### 5.11 失败与回滚

| 失败点 | 行为 | 用户可见 |
|---|---|---|
| 前置断言（`journal_mode=wal` / `oldVersion` 越界 / v15+ 缺源表） | 抛 → 回滚 | 升级失败，App 回退到"数据库打不开"的既有处理 |
| 建表失败（磁盘满/DDL 不支持） | 抛 → 回滚 | 同上 |
| 单行插入失败 | 记 `failed++`，继续处理剩余行（为了在一次运行里报出**全部**失败行） | 无 |
| 数量不闭合 **或** `failed>0` | 抛 → 回滚（**整个迁移一条都不留**） | 升级失败 + `lastMigrationError` |
| 唯一索引建不起来 | 打印前 5 个重复键的 `dedupe_key` 前 8 位（不打印原文） → 抛 → 回滚 | 同上（说明 §5.6 有遗漏） |
| FTS 建表/触发器失败 | **不中止** → `ftsMode` 降档 | 检索变慢但正确 |
| `sqlite_sequence` 表不存在 | **跳过**（不抛），记 `seqAligned=false` | 无（新库不该走迁移，见 §5.7） |
| 改名失败 | 抛 → 回滚（数据已在新表，但事务回滚会一起撤） | 升级失败 |

### 5.12 WAL 前置断言：**拒绝**而不是 checkpoint（P0-2，v3 重写）

v2 §5.12 写的是"检测到 `wal` 就 `PRAGMA wal_checkpoint(FULL)`，把 WAL 并回主库，保证后续文件复制完整"。**这个方案不成立**，两个原因：

1. **位置错**：那段代码在 `MemoryMigrator.migrate()` 里，而备份（`backupBeforeMigration`）早在 `onUpgrade` 第一行就执行完了。checkpoint 发生在备份**之后**，救不了备份。
2. **执行不了**：此时 helper 的 T0 已经 `BEGIN EXCLUSIVE`（WAL 下为 `BEGIN IMMEDIATE`），当前连接持有写锁。`wal_checkpoint` 需要写锁，会返回 `SQLITE_BUSY` 或直接被阻塞——**在事务内 checkpoint 自己的库是不可能的**。

**v3 的方案：断言 + 拒绝。**

```kotlin
// MemoryMigrator 的第 0 步（在 onUpgrade 内、backupBeforeMigration 之后）
val mode = db.rawQuery("PRAGMA journal_mode", null).use { if (it.moveToFirst()) it.getString(0) else "" }
require(mode.lowercase() in setOf("delete", "truncate", "persist")) {
    "迁移拒绝在 WAL 模式下进行：当前 journal_mode=$mode。" +
    "备份已在事务内完成，WAL 下该备份不完整；请先切回 delete 模式并重新打开库。"
}
```

**为什么是"拒绝"而不是"想办法兼容"**：

| 选项 | 代价 | 是否采纳 |
|---|---|---|
| A 拒绝（v3） | 若哪天切 WAL，升级会显式失败并给出可读原因 | ✅ |
| B 把备份前移到 helper 之前 | 需 `open class InboxDatabase` + `override getWritableDatabase/getReadableDatabase`，影响 30+ 构造点；且 `getReadableDatabase` 也会触发 `onUpgrade`，两个入口都要覆盖 | ❌ 超出 §8.5 的 diff 预算 |
| C 在事务内 checkpoint | 技术上不可行（持锁） | ❌ |
| D 备份时连 `-wal` / `-shm` 一起拷 | 拷到的是"hot WAL"，恢复时必须三个文件配套且顺序正确；任何一步失败就是坏库 | ❌ 复杂度远高于收益 |

**当前事实支持选 A**：
- C0.2 §8.10 实测应用库 `journal_mode=delete`；
- `InboxDatabase` 从未调用 `enableWriteAheadLogging()`，Android 默认就是 delete 模式；
- 全仓库 `grep "wal"` 在数据库相关代码里 0 命中（除注释）。

所以 A 是"用断言把不存在的情况变成显式错误"，成本 3 行，收益是**永远不会生成一个坏备份却以为它好的**。

> **如果将来要切 WAL**（例如为了读写并发），必须先做两件事，且都不在 C1-B：① 把 `backupBeforeMigration` 前移到 `getDatabaseLocked` 之前（含 `open class` 改造）；② 在 C1-B5 之外单开一批，用 `T-M47` 验证 WAL 下的备份可恢复。这条写进 §19.3 的 U-13。

**备份本身的语义（不因 WAL 而变）**：

| 项 | 现状 | C1-B 是否改 |
|---|---|---|
| 时机 | `onUpgrade` 第一行，T0 内、未写入 | **不改**（§5.1 第 3 步） |
| 内容 | 只拷主库 `.db` | 不改 |
| 失败处理 | 记 `Log.w`，继续迁移 | 不改（备份是"多一层保险"，不是前置条件） |
| 保留份数 | 最近 3 份 | 不改 |
| 目录 | `databases/pre_migration/` | 不改 |
| **新增** | 记录 `lastBackupError`（静态字段）供自检页读取 | 新增，+2 行 |

### 5.13 Android 设备耗时测试设计（**禁止用桌面 Python 的数**）

C0.2 的预演是 host Python（0.17–0.21 ms/行），C0.2 A-3 明确说"设备 CPU/IO 与 Kotlin 实现都不同"。所以 C1-B2 必须自己测：

| 项 | 设计 |
|---|---|
| 载体 | `app/src/androidTest/java/com/zhuli/memory/MemoryMigrationTimingTest.kt`（新增） |
| 数据 | 用 `MemoryTestFixture.seedLegacy(db, n)` 在设备上**现场生成** n 行 v16 库（n ∈ {1_000, 5_000, 20_000}），不依赖 PC 上的 `.db` 文件 |
| 计时 | `SystemClock.elapsedRealtime()` 包住整段 `onUpgrade`；另计时 `migrateRows` 单独一段 |
| 资源 | `Debug.getMemoryInfo` 取 `totalPss` 峰值差；`StatFs` 取库文件前后体积 |
| 记录 | 每条一个 `Log.i`，字段同 §5.10 + `device` / `sdk` / `sqlite` / `rows` / `ms` / `pss_kb` |
| 门槛 | 5k ≤ 3s（C0.2 A-3）；20k ≤ 12s（线性外推 4×）。超门槛的处置见 §5.13.1 |
| 不算数的 | Robolectric 的耗时（JVM + 不同 SQLite，只用于相对比较） |

#### 5.13.1 超门槛的处置：只能"分批处理、单事务提交"（P0-3，v3 重写）

v2 在 §5.13 与 §14.1 两处都写"超门槛 → 启用 C0.2 §9.6 的分批方案（`batchSize` 参数 + `onUpgrade` 内的分批提交）"。**"在 `onUpgrade` 内分批提交"这句话是错的**，必须撤回：

- `onUpgrade` 被 helper 的 T0 包住。在 T0 内每批 `setTransactionSuccessful()` 只会增加嵌套标记，**不会真的 COMMIT**（Android 的嵌套事务不发 `COMMIT`）；
- 即便绕开 helper 用裸 `execSQL("COMMIT")`，那会把 helper 的事务提前结束，随后 `db.setVersion(17)` 落在事务外——**版本号与数据不同步**，这是最坏的一种半迁移；
- 真正的"分批提交"需要：迁移进度表（哪一批已提交）+ 幂等重入（重启后从断点继续）+ 版本号策略（迁移未完时 `user_version` 不能是 17）。这是 C0.2 §9.6 的原意，但它是**一项独立工程**，不是 C1-B 的一个参数。

**v3 明确区分两种"分批"**：

| 类型 | 做法 | 事务 | 崩溃后 | C1-B 是否做 |
|---|---|---|---|---|
| **分批处理、单事务提交** | 用 `LIMIT/OFFSET` 分页读 `zhuli_fact`，每批处理完只清内存，不 commit | **1 个**（T0） | 整体回滚，下次从头重跑 | ✅ **只做这个** |
| 真正分批提交 | 每批独立事务 + 进度表 + 幂等重入 | N 个 | 从断点继续 | ❌ 不做（C2/C3 评估） |

**所以 `batchSize` 的语义只有一个**：控制单批从 `Cursor` 读入内存的行数，避免 20k 行的 `CursorWindow` 撑爆（Android 的 CursorWindow 默认 2MB，一行 40 列 + 长文本很容易超）。它**不改变事务边界**。

```kotlin
// 分批处理、单事务提交
var lastId = 0L
while (true) {
    val batch = query("SELECT * FROM zhuli_fact WHERE id > ? ORDER BY id LIMIT ?", lastId, BATCH)
    if (batch.isEmpty()) break
    for (row in batch) { outcomes += migrateRow(row) }   // 同一个 T0
    lastId = batch.last().id
    // 不 commit、不 setTransactionSuccessful
}
// 全部批处理完，才走到 §5.9 的闭合断言；然后由 helper 统一提交
```

**超门槛时的实际选项**（按优先级）：

| 优先级 | 措施 | 代价 | 验收 |
|---|---|---|---|
| 1 | **确保迁移在后台线程触发**（首次开库预热：`Executors.newSingleThreadExecutor().submit { db.writableDatabase }`，在 `Application.onCreate` 或首页初始化时） | 需要确认 30+ 处构造点里哪一处是"首次"；不能保证用户不抢先在主线程开库 | D2 记录 `thread=` 字段；StrictMode 无告警（D7） |
| 2 | 调大 `BATCH`（默认 500，可到 2000）减少 cursor 往返 | 内存峰值上升 | `pss_kb` ≤ 40MB |
| 3 | 接受更长时间：20k ≤ **30s**（门槛松绑，但仍是单事务） | 用户首启等待 | D2 实测值写进报告 |
| 4 | 真正的分批提交 | 见上表 | **不在 C1-B** |

> **不给"迁移中"的进度条**：C1 禁止改 UI（§7.11）。所以优先级 1 是唯一能让用户"感觉不到"的手段。这也是为什么 D2 的验收不只看毫秒数，还要看 `thread=` 是不是主线程。

---

## 6. FTS 功能性探测与两条同步路径

### 6.1 C1-A 的结论如何变成代码

C1-A 的结论是 `TRIGGER`，但附了三条硬约束（C1-A §7.2）：

| C1-A 的约束 | C1-B 的落地 |
|---|---|
| 必须功能性探测，不是 DDL 探测 | `detectFtsMode` 第 5 步插哨兵行并核对 FTS 行数 +1（§6.2） |
| 探测与使用**同一连接** | `detectFtsMode(db)` 接收的就是 `onCreate`/`onOpen` 的 `db`，不另开 |
| 结论落盘 + 探测版本 | `SharedPreferences` 键 `mem_fts_mode_v1` / `mem_fts_probe_at` / `mem_fts_probe_sqlite` |

C1-A 还实测了一条**必须写进代码**的事实：`trusted_schema=OFF` 时触发器**建得成、写才炸**（`SQLiteException: unsafe use of virtual table`）。所以第 2 步（建触发器）通过**不能**作为 `TRIGGER` 的依据。

### 6.2 `detectFtsMode` 的七步

```kotlin
// MemorySchema.kt
enum class FtsMode { TRIGGER, APP_SYNC, NONE }

fun detectFtsMode(db: SQLiteDatabase, cache: FtsModeCache?, dbVersion: Int): FtsMode {
    // 0) 缓存命中必须经过"三条有效性校验"，不能直接 return（P0-4，见 §6.4）
    cache?.get(dbVersion = dbVersion, sqliteVersion = sqliteVersion(db), db = db)?.let { return it }

    val tmp = "zhuli_fts_probe_${System.nanoTime()}"      // 独立表名，绝不碰用户表
    try {
        // 1) 建 FTS4 虚拟表（这一步失败 → NONE）
        try { db.execSQL("CREATE VIRTUAL TABLE $tmp USING fts4(search_text, tokenize=simple)") }
        catch (t: Throwable) { return finish(NONE, t) }

        // 2) 建"探测用 item 表 + 3 个触发器"（触发器体与真实 DDL 逐字一致，只换表名）
        db.execSQL("CREATE TABLE ${tmp}_item(id INTEGER PRIMARY KEY, search_text TEXT, status TEXT)")
        val triggers = listOf(
            "CREATE TRIGGER ${tmp}_ai AFTER INSERT ON ${tmp}_item BEGIN " +
                "INSERT INTO $tmp(rowid, search_text) VALUES (new.id, new.search_text); END",
            "CREATE TRIGGER ${tmp}_au AFTER UPDATE OF search_text, status ON ${tmp}_item BEGIN " +
                "DELETE FROM $tmp WHERE rowid = old.id; " +
                "INSERT INTO $tmp(rowid, search_text) SELECT new.id, new.search_text WHERE new.status <> 'DELETED'; END",
            "CREATE TRIGGER ${tmp}_ad AFTER DELETE ON ${tmp}_item BEGIN " +
                "DELETE FROM $tmp WHERE rowid = old.id; END",
        )
        var triggerOk = 0
        for (sql in triggers) {
            try { db.execSQL(sql); triggerOk++ } catch (t: Throwable) { /* 记录，不抛 */ }
        }
        // 2 失败 → APP_SYNC（虚拟表能用，只是触发器不行）
        if (triggerOk < 3) return finish(APP_SYNC, lastError)

        // 3–6) 功能性验证：插哨兵 → 索引 +1 → 改词 → 旧词消失 → 删 → 索引 -1
        val before = ftsCount(db, tmp)                    // 0
        db.execSQL("INSERT INTO ${tmp}_item(id, search_text, status) VALUES (1, '会议 议提 提醒', 'ACTIVE')")
        if (ftsCount(db, tmp) != before + 1) return finish(APP_SYNC, null)   // 建成功但没同步
        if (!matches(db, tmp, "\"会议\" \"提醒\"")) return finish(APP_SYNC, null)
        db.execSQL("UPDATE ${tmp}_item SET search_text='快递 递到 到了' WHERE id=1")
        if (matches(db, tmp, "\"会议\"")) return finish(APP_SYNC, null)      // 旧词还在
        if (!matches(db, tmp, "\"快递\"")) return finish(APP_SYNC, null)     // 新词没进
        db.execSQL("UPDATE ${tmp}_item SET status='DELETED' WHERE id=1")
        if (ftsCount(db, tmp) != before) return finish(APP_SYNC, null)
        db.execSQL("UPDATE ${tmp}_item SET status='ACTIVE' WHERE id=1")
        if (ftsCount(db, tmp) != before + 1) return finish(APP_SYNC, null)   // 恢复没重挂
        db.execSQL("DELETE FROM ${tmp}_item WHERE id=1")
        if (ftsCount(db, tmp) != before) return finish(APP_SYNC, null)

        return finish(TRIGGER, null)
    } catch (t: Throwable) {
        return finish(APP_SYNC, t)                        // 任何未预期异常 → 不赌 TRIGGER
    } finally {
        // 清理：触发器 → item 表 → 虚拟表（顺序不能反）
        runCatching { db.execSQL("DROP TRIGGER IF EXISTS ${tmp}_ai") }
        runCatching { db.execSQL("DROP TRIGGER IF EXISTS ${tmp}_au") }
        runCatching { db.execSQL("DROP TRIGGER IF EXISTS ${tmp}_ad") }
        runCatching { db.execSQL("DROP TABLE IF EXISTS ${tmp}_item") }
        runCatching { db.execSQL("DROP TABLE IF EXISTS $tmp") }
    }
}
```

四点必须照做：

1. **`finish()` 里写缓存**（模式 + `DB_VERSION` + `sqlite_version()` + 时间），只在 `finally` 之前调用一次；
2. **任何异常都落 `APP_SYNC`，绝不落 `TRIGGER`**——这正是 C1-A 的负向对照教的事；
3. 哨兵行用 `id=1`，探测表是新建的，不会与真实数据冲突；
4. 探测在 `onCreate`/`onOpen` 里只跑一次（缓存命中则跳过），且**不在主线程之外另起连接**。

> **`sqliteVersion(db)` 必须用探测的这条连接读**，不能查 `Build.VERSION` 或缓存的静态值——同一台设备上不同连接读到的是同一个 `sqlite_version()`，但如果哪天换了库（例如以后引入自带 SQLite），静态值会说谎。一次 `SELECT sqlite_version()` 成本可忽略。

### 6.2.1 探测与缓存的关系（P0-4，v3 新增）

v2 只有一句 `cache?.get()?.let { return it }`，把"缓存命中"当成了"结论可信"。**缓存只是一个加速器，不是一个真相源**。三件事必须写清：

**① 缓存的三个有效性条件（缺一即失效）**

| 条件 | 检查方式 | 不满足时的后果（若不检查） |
|---|---|---|
| 缓存里的 `db_version == 当前 DB_VERSION` | `prefs.getInt(KEY_DB_VERSION, -1)` | 升级后仍用旧结论；v17 新增了触发器但缓存还是 v16 的 `NONE` |
| 缓存里的 `sqlite_version == 当前 sqlite_version()` | `prefs.getString(KEY_SQLITE, "")` | 系统 OTA 换了 SQLite（比如从 3.42 到 3.45），`trusted_schema` 默认值可能变，结论失效 |
| **缓存的 `TRIGGER` 档必须额外校验 3 个触发器真实存在** | 查 `sqlite_master` 的 3 个名字 | 缓存说 TRIGGER、触发器被删（清库/部分恢复）→ `MemoryStore` 不调 `syncFts`，**索引永久不同步且无人发现** |

第三条是 v3 新增的、也是最容易漏的一条。前两条是"环境变了要重探"，第三条是"结论本身被破坏了要重探"。

**② 缓存的写入门槛**

| 探测结果 | 写缓存吗 | 理由 |
|---|---|---|
| `TRIGGER`（七步全过） | 写 | 最强的结论，但下次命中时仍要过第 ① 条第 3 项 |
| `APP_SYNC`（FTS 能建、触发器不行/不同步） | 写 | 功能正确、只是慢；不需要每次开库都花 80ms 重探 |
| `NONE`（FTS4 建不出） | 写 | 同上 |
| 探测过程中抛未预期异常 | **写 `APP_SYNC`**，不写 `TRIGGER` | 保守：宁可慢，不可错。异常原因进日志 |

**③ 连接级配置变化怎么处理**

`trusted_schema` 是**连接级** pragma（C1-A §5 已证：`PRAGMA trusted_schema=OFF` 后触发器"建得成、写才炸"）。C1-A §7.2 因此要求"探测与使用同一连接"。C1-B 的处理：

| 层 | 做法 |
|---|---|
| 同一 `SQLiteDatabase` 实例 | 探测用 `onOpen(db)` 的 `db`；`MemoryStore` 用 `db.writableDatabase`。两者是**同一个 `SQLiteDatabase` 对象**（helper 的 `mDatabase`），但不是同一条底层连接（连接池按线程分配） |
| 无法保证"同一条连接" | `SQLiteDatabase` 的连接池在并发时会给不同线程不同连接；`trusted_schema` 不在 `SQLiteDatabaseConfiguration` 里，无法通过 open 参数统一设置 |
| 因此的结论 | 探测结论是**库级假设**：假定"本进程内没有第三方在任何连接上改过 `trusted_schema`"。C1 的代码里没有任何地方改它 → 假设成立 |
| 若假设被破坏 | 只能靠 `selfCheck` 的第 1/2 条（FTS 与 item 不一致）在事后发现，不能事前阻止。**这是 U-12，写进 §19.3** |

**④ 降级条件（明确"什么时候允许降"）**

```
FTS4 虚拟表建不出            → NONE      （能力缺失，只能 LIKE）
FTS4 建出、触发器 < 3        → APP_SYNC  （能力部分缺失，Store 兜底）
FTS4 + 触发器都建出但不同步   → APP_SYNC  （C1-A 的"建成功、写才炸"就是这一类）
探测代码自身抛异常            → APP_SYNC  （不赌，见上表第 4 行）
```

**唯一不允许的降级**：从 `APP_SYNC` 降到 `NONE`。`NONE` 只由"FTS4 建不出"触发，不能因为"触发器不行"就给 `NONE`——那会白白丢掉 FTS 的检索能力。

### 6.3 三档下的写路径差异（P0-5，v3 重写）

**完整职责表**（v3 补齐：v2 只列了 4 个写路径，漏了 `confirm` / `resolveConflict` / `expire`——这三条都改 `status`，`APP_SYNC` 下漏调就是"确认了但还是搜不到"）：

| # | 路径 | 改动的列 | `TRIGGER` | `APP_SYNC` | `NONE` |
|---|---|---|---|---|---|
| 1 | `insert`（新建行） | 插入 `search_text` + `status` | 触发器 `_ai` 负责 | **`syncFts(id)`** | 只写 `search_text` |
| 2 | `correct`（新建行 + 旧行失效） | 新行 INSERT；旧行 `status`/`valid_until` | `_ai` + `_au` | **两次 `syncFts`**（新 id、旧 id） | 只写 `search_text` |
| 3 | `confirm` | `status` CANDIDATE→ACTIVE | `_au` | **`syncFts(id)`** | 不碰 FTS |
| 4 | `resolveConflict` | `status`→SUPERSEDED | `_au` | **`syncFts(id)`** | 不碰 |
| 5 | `expire`（TASK 到期/STATE 关闭） | `status`→EXPIRED + `valid_until` | `_au` | **`syncFts(id)`** | 不碰 |
| 6 | `softDelete` | `status`→DELETED | `_au`（把该 rowid 从 FTS 删掉） | **`syncFts(id)`**（幂等：先删后插，因 status=DELETED 不再插） | 不碰 |
| 7 | `restore` | `status`→target | `_au` | **`syncFts(id)`** | 不碰 |
| 8 | `touchUsed` | `times_retrieved` / `last_used_at` | **不触发**（`_au` 限定 `OF search_text, status`） | 不调 | 不碰 |
| 9 | 迁移灌数据（§5.2 第 7–9 步） | 批量 INSERT | **先 `INSERT…SELECT` 灌完，再建触发器** | 同左（灌完靠运行期 `APP_SYNC` 维护增量） | 跳过，只保证 `search_text` 已写 |
| 10 | 硬删（C3） | 物理删行 | `_ad` | `syncFts` 或直接 DELETE | 不碰 |
| — | 检索 | — | `MATCH` | `MATCH` | `LIKE` 兜底 |

**三条不变量（"不双写、不漏写"的可检验形式）**：

| # | 不变量 | 怎么验 |
|---|---|---|
| **W-1** | `TRIGGER` 档下，`MemoryStore` **一次都不调** `syncFts` | `grep -n "syncFts" MemoryStore.kt` 的每处都在 `if (ftsMode == APP_SYNC)` 分支内；`T-S21`（v3 新增）断言"TRIGGER 档下写一次只产生 1 条 FTS 行"（若双写会变成 2 条或抛约束） |
| **W-2** | `APP_SYNC` 档下，**凡改 `status` 或 `search_text` 的语句，同一事务内必有一次 `syncFts`** | 上表 1–7 共 7 条路径，每条一个用例；`T-S22`（v3 新增）用"逐条路径写完就查 FTS 行数"的方式断言 |
| **W-3** | 任何档下，`search_text` **都必须写** | `NONE` 档的 LIKE 兜底、`APP_SYNC` 档的 `syncFts`、`TRIGGER` 档的触发器，全部依赖这一列。`T-S23`（v3 新增）在 `NONE` 档下断言 `search_text` 非空且 LIKE 能命中 |

**唯一的写入职责位置**：`syncFts` 只出现在 `MemoryStore` 的**私有方法**里，调用点是上表 1–7 那 7 处。**不许**散落在 `MemoryRepo`、`MemoryIndexer` 或调用方——否则 W-1 / W-2 无法用 grep 验证。

**`deleteFts` 不需要单独存在**：`syncFts` 的实现是"先 `DELETE WHERE rowid=?`，再 `INSERT … SELECT … WHERE status<>'DELETED'`"。当行是 DELETED 时，第二条 INSERT 选不出行，效果等于删除。所以 v2 里的 `deleteFts` 合并进 `syncFts`，API 面从 2 个减到 1 个。

`syncFts` 的实现（幂等，可重复调）：

```sql
DELETE FROM zhuli_memory_fts WHERE rowid = ?;
INSERT INTO zhuli_memory_fts(rowid, search_text)
  SELECT id, search_text FROM zhuli_memory_item WHERE id = ? AND status <> 'DELETED';
```

**触发器 DDL 的列限定是 W-1 的前提**：`_au` 必须写 `AFTER UPDATE OF search_text, status ON zhuli_memory_item`。少了 `OF` 限定，`touchUsed` 的每次 `UPDATE` 都会触发一次全行 DELETE + INSERT——20k 行的库上，一次检索结果的 `touchUsed` 会变成 20 次 FTS 写。

### 6.3.1 `NONE` 档下的检索约束（v3 补充）

`NONE` 不是"跳过所有条件"。`search()` 在 `NONE` 档下走 LIKE，但**四个约束一个都不能少**（§7.9）：`status` / 生效区间 / `sensitivity <= max` / scope OR 列表。v2 §7.9 已经写了这句，这里再强调一次是因为实现时最容易"为了性能把 scope 拿掉"。

```sql
-- NONE 档的正确写法（示意）
SELECT * FROM zhuli_memory_item i
WHERE i.search_text LIKE ? ESCAPE '\'            -- ← 通配符要转义用户输入里的 % 和 _
  AND i.status = 'ACTIVE'
  AND (LB_SQL) <= :now AND (valid_until = 0 OR :now < valid_until)
  AND i.sensitivity <= :max
  AND (scope OR 列表)                             -- 与 FTS 档完全同一段 SQL
LIMIT :limit
```

`LIKE` 的转义是 v3 新增的注意点：用户搜 `100%` 时，`"%" + kw + "%"` 会变成 `%100%%`，把所有以 100 开头的行都捞出来。必须 `kw.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")` 后再拼。

### 6.4 `FtsModeCache` 的完整契约（P0-4，v3 重写）

v2 只列了 3 个键和"失效条件见下"，没有一个可实现的 `get()`。v3 把它写成一个有明确返回值的函数——**缓存只返回"可信的结论"或 `null`，绝不返回"可能过期的结论"**。

**四个键**（v2 是三个，v3 加 `probe_db_version`）：

| 键 | 类型 | 值 | 作用 |
|---|---|---|---|
| `mem_fts_mode_v1` | String | `TRIGGER` / `APP_SYNC` / `NONE` | 结论 |
| `mem_fts_probe_db_version` | Int | 探测时的 `DB_VERSION` | 升级后必失效 |
| `mem_fts_probe_sqlite` | String | 探测时的 `sqlite_version()` | OTA 换 SQLite 后必失效 |
| `mem_fts_probe_at` | Long | 探测时间（ms） | 只用于诊断，不参与失效判断 |

**`get()` 的六道门（任一不过就返回 `null`）**：

```kotlin
fun get(dbVersion: Int, sqliteVersion: String, db: SQLiteDatabase): FtsMode? {
    // 1) 没写过
    val raw = prefs.getString(KEY_MODE, null) ?: return null
    // 2) 值不是合法枚举（被手工改过 / 未来版本降级）
    val mode = FtsMode.entries.firstOrNull { it.name == raw } ?: return null
    // 3) DB_VERSION 变了
    if (prefs.getInt(KEY_DB_VERSION, -1) != dbVersion) return null
    // 4) SQLite 版本变了
    if (prefs.getString(KEY_SQLITE, null) != sqliteVersion) return null
    // 5) 结论是 TRIGGER，但 3 个触发器不在了 → 结论已失效
    if (mode == FtsMode.TRIGGER && countFtsTriggers(db) != 3) return null
    // 6) 结论不是 NONE，但 FTS 虚拟表不在了 → 结论已失效
    if (mode != FtsMode.NONE && !tableExists(db, "zhuli_memory_fts")) return null
    return mode
}
```

第 5、6 道门是 v3 新增的**结构一致性校验**，也是这套缓存唯一"防得住"的地方：`ensureAll` 会补建表和触发器，但如果补建失败（磁盘满、ROM 限制），缓存仍然说 `TRIGGER`，而实际没有触发器 → `MemoryStore` 永远不调 `syncFts` → **索引静默不同步**。第 5/6 道门把这种情况变成"重探一次"，代价 80ms，收益是"不会静默坏掉"。

**`put()` 的写法**：先写 `KEY_MODE` 最后写——这样任何写一半的崩溃都表现为"没有 mode 键"或"版本不匹配"，被 `get()` 判为 `null`，不会出现"有 mode 但没有版本"的半状态。

```kotlin
fun put(mode: FtsMode, dbVersion: Int, sqliteVersion: String, now: Long) {
    prefs.edit()
        .putInt(KEY_DB_VERSION, dbVersion)
        .putString(KEY_SQLITE, sqliteVersion)
        .putLong(KEY_AT, now)
        .putString(KEY_MODE, mode.name)   // ← 最后
        .commit()                          // 必须同步提交：下一次开库可能立刻读
}
```

**强制重探的四个时机**（v2 是三个，v3 加"结构修复后"）：

| # | 时机 | 触发方式 |
|---|---|---|
| 1 | 首次安装 / 清数据 | 无缓存 → `get()` 返回 null |
| 2 | `DB_VERSION` 变化（升级） | 第 3 道门 |
| 3 | `sqlite_version()` 变了（系统 OTA） | 第 4 道门 |
| 4 | **`ensureAll` 补建过触发器或 FTS 表** | 补建成功后主动 `prefs.remove(KEY_MODE)`，强制下次重探 |

第 4 条容易被忽略：`ensureAll` 发现触发器缺失并补建成功后，**不能**假定"现在一定好了"——它可能在补建时也失败了（只是没抛）。所以补建动作之后直接清缓存，让探测去实际验证。这是"结构自愈"与"能力探测"的分工：`ensureAll` 负责让对象存在，`detectFtsMode` 负责证明它可用。

**手动重探入口**：`MemorySchema.invalidateFtsCache()`，只被 `selfCheck` 的 debug 菜单调用，不暴露 UI。

### 6.5 索引不一致的兜底

`MemoryStore.selfCheck()` 里加两条（C0.2 §6.6 的第 1、2 条，已在清单里）：

```sql
-- FTS 有、item 没有
SELECT rowid FROM zhuli_memory_fts
WHERE rowid NOT IN (SELECT id FROM zhuli_memory_item WHERE status<>'DELETED');
-- item 活着、FTS 没有（ftsMode=NONE 时跳过，那是已知降级）
SELECT id FROM zhuli_memory_item i WHERE i.status<>'DELETED'
  AND NOT EXISTS (SELECT 1 FROM zhuli_memory_fts f WHERE f.rowid=i.id);
```

第 2 条在 `APP_SYNC` 下**就是漏同步的检测器**。C1 的 `selfCheck` 只报告不自动修（自动修在 C3 的维护任务里），但提供 `MemoryStore.rebuildFts()` 供 `MemoryIndexer` 之外的手动入口调用。

### 6.6 不允许做的事

| # | 禁止 | 理由 |
|---|---|---|
| 1 | 把 `ftsMode` 写成 `const val TRIGGER` | C1-A 只在**一台**机器上验证；A-6 未关闭 |
| 2 | 用 `CREATE TRIGGER` 是否抛错当唯一依据 | C1-A §5 实测"建成功、写才炸" |
| 3 | 探测开新连接 | 结果依赖连接级 pragma（C1-A §7.2） |
| 4 | 探测失败后静默继续 | 索引不一致会让检索"看起来正常但漏结果" |
| 5 | 用 Robolectric 的结果决定模式 | A-2 已证伪（Robolectric 复现不出失败，方向都相反） |
| 6 | `ftsMode=NONE` 时跳过 `search_text` 的写入 | 那是 LIKE 兜底的唯一依据 |

---

## 7. Store / Manager 设计

### 7.1 四个类的职责（C0.2 §10.6 的展开）

```
┌──────────────────────────────────────────────────────────────┐
│ MemoryRepo（适配层，485 → ~150 行）                            │
│   · 27 个公开签名不变                                          │
│   · 只做：MemoryDraft ↔ FactDraft 转换、MemoryItem → Fact 投影  │
│   · 向量 7 个方法保持直写（§3.2 的边界）                         │
│   · 禁止：写 SQL、开事务、判冲突、碰 pre_delete_status           │
└───────────────────────┬──────────────────────────────────────┘
                        │ 调用
┌───────────────────────▼──────────────────────────────────────┐
│ MemoryStore（唯一写入口 + 全部查询，~520 行）                    │
│   写：insert / correct / confirm / resolveConflict /           │
│       softDelete / restore / touchUsed                         │
│   查：byId / current / asOfWorld / asOfBelief / search /       │
│       candidates / history / counts                            │
│   维护：selfCheck / rebuildFts / syncFts                       │
│   · 单事务边界（§7.3）                                          │
│   · 禁止：读 zhuli_fact_v1、判 FactKind、构造 Fact              │
└──────┬──────────────────────────────────┬────────────────────┘
       │ 用                                │ 用
┌──────▼───────────────┐        ┌─────────▼────────────────────┐
│ MemoryPolicy（~130）  │        │ MemoryGuard（~90）            │
│  rank / cardinalityOf│        │  classify / excerpt /        │
│  decide(R0–R9)       │        │  validateScope / 长度上限     │
│  纯函数，无 IO        │        │  纯函数，无 IO                │
└──────────────────────┘        └──────────────────────────────┘

旁路（不在 Store 内）：
MemoryMigrator（~250）  只在 onUpgrade 内被调一次
MemorySchema（~150）    只在 onCreate / onOpen / Migrator 内被调
MemoryText（~90）       纯函数，被所有类依赖
```

`MemoryManager` **不新建**：C0.2 §4.8 提到的 `MemoryManager.correct()` 在 C1 由 `MemoryStore.correct()` 承担；"Manager"这层等到 C2 有"提取→判定→写入"的编排需求时再加，C1 提前建会变成空壳。

### 7.2 写入 API（签名与语义）

```kotlin
class MemoryStore(private val db: InboxDatabase) {

    /** 唯一写入口。返回结果携带"落到哪个状态"，调用方不需要猜。 */
    fun insert(draft: MemoryDraft, now: Long): InsertResult

    /** 用户纠错：继承 valid_from / time_confidence；旧行 valid_until=新起点 + CORRECTS。 */
    fun correct(id: Long, newValue: String, now: Long): Long?

    /** 用户确认候选（无新值）。 */
    fun confirm(id: Long, now: Long): Boolean

    /** 裁决冲突：keep 保持 ACTIVE，drop → SUPERSEDED + 关系。 */
    fun resolveConflict(keepId: Long, dropId: Long, now: Long)

    /** 软删：任何状态都可删；记录 pre_delete_status。 */
    fun softDelete(id: Long, now: Long): DeleteResult

    /** 恢复：回到 pre_delete_status；冲突时降级或 AlreadyPresent。 */
    fun restore(id: Long, now: Long): RestoreResult

    /** 检索一次就 +1（不触发 FTS，因为 UPDATE 不含 search_text/status）。 */
    fun touchUsed(ids: List<Long>, now: Long)
}
```

`MemoryDraft` 是 `[实现补充]`——C0.2 用 `draft` 这个词但没定义。字段：

```kotlin
data class MemoryDraft(
    val memoryType: MemoryType,
    val subject: String,
    val predicate: String,
    val value: String,
    val qualifierKey: String = "",
    val cardinality: Cardinality? = null,   // null → 由 MemoryPolicy 推
    val basis: ConfidenceBasis,
    val confidence: Double,
    val scope: MemoryScope,
    val worldFrom: Long = 0,                // 用户明确给的起点；0 = 不知道
    val timeConfidence: TimeConfidence = NONE,
    val worldUntil: Long = 0,
    val eventStart: Long = 0,
    val eventEnd: Long = 0,
    val source: SourceRef,                  // type + 可选 ref
    val excerpt: String,                    // 原文片段，MemoryGuard 会再脱敏
    val intent: WriteIntent = WriteIntent.NEW,   // NEW | CORRECT | CHANGE
    val forceCandidate: Boolean = false,    // 对应 activateThreshold 命中（X-4）
    val legacyKind: String = "",
)

sealed class InsertResult {
    data class Created(val id: Long, val status: MemoryStatus) : InsertResult()
    data class Reinforced(val id: Long, val independentSources: Int) : InsertResult()
    data class Suppressed(val reason: SuppressReason) : InsertResult()
    data class Rejected(val reason: RejectReason) : InsertResult()
    data object Duplicate : InsertResult()
}

enum class RejectReason { SECRET, EMPTY_TEXT, TOO_LONG }
enum class SuppressReason { TOMBSTONED, DELETED_WITHOUT_RESTORE, REJECTED_WITHOUT_RESTORE }
```

**`InsertResult` 是 5 个类型，不是 4 个（P1-5，v3 修正）**

v2 的代码块里有 5 个 `data class/object`，正文却写"四种结果直接对应 R0–R9 的四个出口"。代码与正文对不上，实现者会纠结 `Duplicate` 到底存不存在。v3 定死为 **5 个**，并把"触发条件 + 数据库副作用"写成可验收的表：

| 类型 | 触发条件 | 写 item | 写 source | 写 evidence | 写 relation | 写 revision | 返回给调用方的含义 |
|---|---|---|---|---|---|---|---|
| `Created(id, status)` | R4b / R5 / R7（新建行），或 R3 的"用户显式陈述复活" | **+1** | 0 或 1（有 `SourceRef` 时） | +1 | 0 或 1（`SUPERSEDES` / `RESTORES`） | +1（`CREATE`） | "产生了新记忆"，`ingest` 的计数 +1 |
| `Reinforced(id, n)` | R2：`dedupe_key` 命中 ACTIVE/CANDIDATE，且**来源独立**（`independent_key` 不在已有 evidence 里） | 0（只改 `times_reinforced` / `updated_at` / `revision`） | 0 或 1 | **+1**（`SUPPORTING`） | 0 | +1（`REINFORCE`） | "同一条事实又有一个独立来源"，计数 **不** +1 |
| `Duplicate` | R2 的退化分支：`dedupe_key` 命中，且**同一 source 重复到达**（`idx_ms_dedupe` 命中，`content_hash` 相同） | 0 | 0 | 0（只 `seen_count++` / `last_seen_at`） | 0 | **0** | "这条来源已经记过了"，零副作用 |
| `Suppressed(reason)` | R1（墓碑）/ R3（DELETED/REJECTED 且非用户显式陈述） | 0 | 0 | 0 | 0 | +1（`SUPPRESS`，**用于审计"为什么没记"**） | "被策略挡下"，计数不 +1 |
| `Rejected(reason)` | R0：SECRET / 空文本 / 超长 | 0 | 0 | 0 | 0 | **0** | "根本不该记"，零写入 |

三条必须写进 KDoc 的边界：

1. **`Reinforced` 与 `Duplicate` 的分界是"来源是否独立"**，不是"值是否相同"。同一短信被通知和短信两条通道各投递一次 → 两个 `source_id` 不同但 `independent_key` 可能相同（都指向同一个 `inbox.id`）→ 走 `Duplicate`。这条判据来自 C0.2 §5.2 的 `independent_key` 定义。
2. **`Suppressed` 写 revision，`Rejected` 不写**。两者的区别是"策略判断"vs"合法性判断"：前者是系统主动决定不记（需要审计），后者是输入不合法（没有可审计的对象）。
3. **`ingest` 的返回值只数 `Created`**。现有调用方（`ExtractPipeline` 日志、`ZhuliAgentTools` 文案）都按"新建了几条"理解，`Reinforced` / `Duplicate` 不算新建。

> `Duplicate` 在 v1/v2 里没有触发条件，是"代码里有、文档里没有"的孤儿。v3 把它绑定到"同一 source 重复到达"，并明确"零副作用"，这样 `T-S11` 的变体（同值两来源 → `Reinforced`）与 `T-S24`（同一来源重复 → `Duplicate`）才有区分度。

### 7.3 事务边界（每个写方法一个事务，不嵌套）

| 方法 | 事务内做的事 | 提交后做的事 |
|---|---|---|
| `insert` | R0 校验 → R2/R3 查重 → R4 取槽 → R5/R7/R8/R9 决策 → 写 item + source + evidence + relation + revision → 回填 `source_id` | `onIndexRequested`（由 `MemoryRepo` 触发） |
| `correct` | 读旧行 → 写新行（继承）→ 旧行 `SUPERSEDED` + `valid_until` → `CORRECTS` → 两条 revision | 入向量队列 |
| `confirm` | 读 → 校验 CANDIDATE → 改 ACTIVE + `last_confirmed_at` → revision(`CONFIRM`) | 入队 |
| `resolveConflict` | 读两行 → `drop` 改 SUPERSEDED + `invalidated_at` → `SUPERSEDES` → revision | 删 drop 的向量 |
| `softDelete` | 读 `pre` → 改 DELETED（`pre_delete_status` 仅首次、`invalidated_at` 仅 0）→ 删 FTS/向量/队列 → revision(`DELETE`) | `optimize`（C3 才做） |
| `restore` | 读 → 定 `target` → 查 `dedupe_key` → 查槽位 → 改状态 → 关系 → revision(`RESTORE`) | 入队（仅 ACTIVE） |
| `touchUsed` | 单条 UPDATE（不含 `search_text`/`status`，所以不触发 FTS） | — |

**禁止跨方法复用事务**：`insert` 内部**不调** `correct`，`restore` 内部**不调** `insert`。需要组合的场景（如"删除后重新写入同值"）由调用方顺序调两个方法，各自成事务——因为中间状态是合法的（旧行 DELETED、新行还没建）。

### 7.4 并发与线程

| 项 | 设计 | 理由 |
|---|---|---|
| 并发模型 | 依赖 SQLite 的单写者模型 + Android 的 `SQLiteDatabase` 连接池 | 不自建锁 |
| 写串行化 | 所有写方法在 `db.writableDatabase` 上开事务；SQLite 会在库级加写锁 | 不需要应用层 `synchronized` |
| 读 | `db.readableDatabase`，可与写并行（`journal_mode=delete` 下读会等写提交） | — |
| 长事务 | **禁止**在事务内做 IO（网络、文件、embed） | 会持有写锁 |
| 调用线程 | 允许任意线程；`hybridSearch` 是 `suspend`，但内部不切线程（保持现有行为） | 现有调用方已在协程/工作线程 |
| 主线程 | `onCreate`/`onOpen` 里的 `createAll`/`detectFtsMode`/`migrate` 都在打开库的线程上；C1-B 加**线程断言**（debug 下 `Looper.myLooper() == Looper.getMainLooper()` 时 `Log.w`） | A-4 未验证；不能假设不在主线程 |
| 死锁 | 不在事务内调 `db.writableDatabase`（会递归） | `InboxDatabase.kt:1290` 的注释已踩过 |

**`[实现补充]` 并发测试**：`T-D31` 的变体——两个线程同时对同一 `dedupe_key` 调 `insert`，断言最终只有 1 条 ACTIVE、另一条走 `Reinforced` 或被 SQLite 的写锁串行化后命中 R2。这条加进 `MemoryStoreTest`，不占 57 条的编号（属于"实现质量"而非"架构用例"）。

### 7.5 完整状态转换表（P1-4，v3 重写）

v2 的矩阵有三个问题：① 单元格混写"方法 + 结果"（`restore` 的 `Demoted` 写在 `to=CANDIDATE` 格子里，但它其实是"restore 的一个返回值"）；② `CANDIDATE → SUPERSEDED` 写"R7(替代候选? 否)"，等于没定；③ `CANDIDATE → REJECTED` 写"（C2）"，但没写清 C1 到底允许不允许。v3 拆成"合法转换"与"禁止转换"两张表，每条转换都有 SQL 前置条件。

#### 7.5.1 合法转换（10 条，覆盖全部写入路径）

| # | from | to | 方法 | 前置条件（`WHERE` 里的状态） | 同时写什么 | revision |
|---|---|---|---|---|---|---|
| 1 | ∅ | `ACTIVE` | `insert` R4b / R5 / R7 | — | `valid_from` 按 R0；`observed_at=now` | `CREATE` |
| 2 | ∅ | `CANDIDATE` | `insert` R8 / R9 / `forceCandidate` / R0 时间不自洽 | — | 同上 + `CONFLICTS_WITH`（R8/R9 时） | `CREATE` |
| 3 | `CANDIDATE` | `ACTIVE` | `confirm(id, now)` | `status='CANDIDATE'` | `last_confirmed_at=now`、`invalidated_at=0`、入向量队列 | `CONFIRM` |
| 4 | `CANDIDATE` | `SUPERSEDED` | `resolveConflict(keep, drop, now)` 的 drop 方 | `status='CANDIDATE'` | `superseded_by=keep`、`invalidated_at=now`、`SUPERSEDES` 关系 | `RESOLVE` |
| 5 | `ACTIVE` | `SUPERSEDED` | ① `insert` 的 R7（新值 rank 更高）② `correct()` 的旧行 ③ `resolveConflict()` 的 drop 方 | `status='ACTIVE'` | `superseded_by`、`invalidated_at=now`；`correct` 还要 `valid_until=新起点` | `SUPERSEDE` / `CORRECT` / `RESOLVE` |
| 6 | `ACTIVE` | `EXPIRED` | ① `insert` 的 R4a（`UB>0 AND UB<=now`）② `expire(id, now)`（TASK 到期 / STATE 关闭） | `status='ACTIVE'` | `valid_until=now`（若还是 0）、`invalidated_at=now` | `EXPIRE` |
| 7 | `ACTIVE` / `CANDIDATE` / `SUPERSEDED` / `EXPIRED` / `REJECTED` | `DELETED` | `softDelete(id, now)` | `status<>'DELETED'` | `pre_delete_status`（仅首次）、`deleted_at=now`、`invalidated_at`（仅 0 时）、删 FTS/向量/队列 | `DELETE` |
| 8 | `DELETED` | 见下 | `restore(id, now)` | `status='DELETED'` | `deleted_at=0`、`pre_delete_status=''`、`invalidated_at`（回 ACTIVE/CANDIDATE 时置 0） | `RESTORE` |
| 9 | ∅ | `REJECTED` | **C1 不产生**（无 reject 入口，C2 才有） | — | — | — |
| 10 | `CANDIDATE` / `ACTIVE` | `REJECTED` | **C1 不产生**（C2 的"否决"） | — | — | — |

**第 8 条 `restore` 的目标不是自由选择，而是查表**（这是 v2 矩阵最容易误读的地方）：

| `pre_delete_status` | 恢复目标 | 还要做什么 |
|---|---|---|
| `ACTIVE` | `ACTIVE` | 第 3 步查同槽不同值的 ACTIVE；命中 → 降为 `CANDIDATE` + `Demoted` |
| `CANDIDATE` | `CANDIDATE` | 不查槽位（候选本来就不占当前值） |
| `SUPERSEDED` | `SUPERSEDED` | 不查槽位；`invalidated_at` **保持原值**（不能清零） |
| `EXPIRED` | `EXPIRED` | 同上；`valid_until` 保持 |
| `REJECTED` | `REJECTED` | 同上 |
| `''`（迁移行 / 异常） | `CANDIDATE` | **不猜**（C0.2 §6.9）；写 `RESTORE` revision 时 `after` 记 `"CANDIDATE"` |

**为什么 `SUPERSEDED` 恢复后仍是 `SUPERSEDED`**：`pre_delete_status` 记录的是"删之前它是什么"，不是"删之前它是不是当前值"。把它恢复成 `ACTIVE` 会让一个槽位出现两个当前值——这正是 C0.2 的缺陷 #3。

#### 7.5.2 禁止转换（6 条）与失败方式

| # | 禁止 | 为什么 | 尝试时的结果 |
|---|---|---|---|
| 1 | `SUPERSEDED → ACTIVE`（不经 `restore`） | 会让历史值变当前值 | `UPDATE … WHERE status='SUPERSEDED'` 的 `changes()==0` → `IllegalStateException` |
| 2 | `EXPIRED → ACTIVE` | 同上 | 同上 |
| 3 | `REJECTED → ACTIVE` | "从源头否决"不能变成"当前为真" | 同上 |
| 4 | `ACTIVE → CANDIDATE`（除 `restore` 的 `Demoted`） | 会凭空制造"待确认"，且不写 `CONFLICTS_WITH` | C1 没有这个 API；`restore` 之外不存在这条路径 |
| 5 | `DELETED → ACTIVE`（除 `restore`） | 绕过 `pre_delete_status` 与唯一索引重查 | 同上 |
| 6 | 任何 → `UNKNOWN`（写库） | `UNKNOWN` 只产生于读取 | 枚举里没有写入路径；`MemoryStatus` 根本没有 `UNKNOWN` |

**"禁止"的实现方式（不靠调用方守规矩）**：每条 UPDATE 都带 `WHERE id=? AND status=?`（乐观锁），`changes()==0` 时抛 `IllegalStateException("状态已变更：期望 $expected，实际见 byId")`。**不用** `UPDATE … WHERE id=?` 然后 `if (changes()==0) 忽略`——那会把并发冲突变成静默失败。

#### 7.5.3 并发下的转换（v3 新增，对应"验收细节 1"）

上面的乐观锁只保证"状态没被别人改"，不保证"两个线程同时改同一条"时语义正确。三种真实场景：

| 场景 | 两个线程的动作 | 期望 | 靠什么 |
|---|---|---|---|
| 同值并发 insert | T1 `insert("快递到了")`，T2 `insert("快递到了")` | 1 条 item + 2 条 evidence（或 1 条 + `Duplicate`），**不抛 UNIQUE** | `idx_mi_dedupe` 唯一索引 + `INSERT OR IGNORE` 后重查；SQLite 写锁串行化 |
| 并发 confirm | T1 `confirm(id)`，T2 `confirm(id)` | 1 个成功，1 个返回 `false`（`changes()==0`） | `WHERE status='CANDIDATE'` |
| 删与恢复并发 | T1 `softDelete(id)`，T2 `restore(id)` | 按提交顺序二选一；后到者看到的状态与它预期不符 → 抛 `IllegalStateException` 或返回 `NotDeleted` | 乐观锁 + 单写者模型 |

这些不是"理论风险"：`ExtractPipeline`（通知）与 `ZhuliAgentTools`（用户工具）可能同时写同一条记忆。对应测试 `T-N01`–`T-N05` 见 §10.12。

> **不做应用层锁**：SQLite 的库级写锁已经把写串行化了，加 `synchronized` 只会把"数据库等待"变成"线程等待"，且掩盖真正的并发缺陷。C1-B 依赖单写者模型（§7.4）。

### 7.6 删除与恢复的时序

**软删**（`softDelete`，单事务）：

```
0  SELECT status, pre_delete_status, invalidated_at FROM item WHERE id=:id
   ├─ 无行 → NotFound
   └─ status='DELETED' → NotDeleted('DELETED')      // 幂等
1  UPDATE item SET
     pre_delete_status = CASE WHEN status='DELETED' THEN pre_delete_status ELSE status END,
     status='DELETED', deleted_at=:now,
     invalidated_at = CASE WHEN invalidated_at=0 THEN :now ELSE invalidated_at END,
     updated_at=:now
   WHERE id=:id AND status<>'DELETED'
2  DELETE FROM zhuli_memory_fts WHERE rowid=:id       // TRIGGER 模式下触发器也做，幂等
3  DELETE FROM zhuli_fact_vec WHERE fact_id=:id
4  DELETE FROM zhuli_vec_queue WHERE fact_id=:id
5  INSERT revision(change_kind='DELETE', actor='USER',
        before_json='{"status":"<pre>"}', after_json='{"status":"DELETED"}')
```

**恢复**（`restore`，单事务，8 步照 C0.2 §6.9）：

```
0  取行；不存在 → NotFound；status<>'DELETED' → NotDeleted(status)
1  target = pre_delete_status ∈ {ACTIVE,CANDIDATE,SUPERSEDED,EXPIRED,REJECTED} ? 它 : 'CANDIDATE'
2  target ∈ {ACTIVE,CANDIDATE}：
     查同 dedupe_key 的其它行（status IN ('ACTIVE','CANDIDATE') 且 id<>:id）
       命中 → 复制 evidence 给占用行（role='SUPPORTING'）→ AlreadyPresent(occupantId)
3  target=ACTIVE 且 cardinality=SINGLE：
     查同槽位不同值的 ACTIVE（slot_hash 相同、value 不同）
       命中 → target='CANDIDATE'，记 conflict
4  UPDATE status=target, deleted_at=0, pre_delete_status='', updated_at=:now,
          invalidated_at = CASE WHEN :target IN ('ACTIVE','CANDIDATE') THEN 0 ELSE invalidated_at END
5  若有 conflict → relation(min,max,'CONFLICTS_WITH')
6  revision('RESTORE', before='{"status":"DELETED"}', after='{"status":"<target>"}')
7  target=ACTIVE → 入向量队列；FTS 由触发器/APP_SYNC 重挂
8  返回 Restored / Demoted / AlreadyPresent
```

第 2 步的 SQL 必须**同时**含 `ACTIVE` 和 `CANDIDATE`——C0.2 §6.9 的探针实测证明 `CANDIDATE` 也占唯一索引。

### 7.7 来源关联与 GC

| 场景 | 处理 |
|---|---|
| 同值新来源（R2） | `INSERT OR IGNORE evidence(memory_id, source_id, role='SUPPORTING')`；`seen_count++`、`last_seen_at=now`；`times_reinforced` 由独立来源数派生 |
| 同一来源重复到达 | `idx_ms_dedupe` 命中 → 复用 source 行，`seen_count++` |
| 删除一条记忆 | evidence **保留**（审计；`excerpt` 已脱敏）；source 保留 |
| 硬删（C3） | 删 evidence → 无引用 source 才 GC |
| 来源无引用（C1 可见） | `selfCheck` 第 6 条只**报告**，不自动删（C1 没有硬删，自动删会删掉"将来可能被引用"的行） |

**引用计数不用列**：`COUNT(*) FROM evidence WHERE source_id=?` 走 `idx_mev_source`，比维护一个计数器更难出错。

### 7.8 查询 API

| 方法 | 语义 | SQL 要点 |
|---|---|---|
| `byId(id)` | 任意状态单条 | 不过滤状态（UI 要看历史） |
| `current(limit, scopes, maxSensitivity)` | `CURRENT` | `status='ACTIVE' AND LB<=now AND (UB=0 OR now<UB)` + scope OR 列表 |
| `asOfWorld(at, …)` | 世界线 | §2.3 的三段谓词 + `CORRECTS` 排除 |
| `asOfBelief(at, …)` | 信念线 | `observed_at<=at AND (invalidated_at=0 OR at<invalidated_at)` |
| `candidates(limit)` | 待确认列表 | `status='CANDIDATE'` |
| `history(slot)` | 同槽全部（含 DELETED） | 按 `recorded_at` 升序 |
| `search(text, now, limit, …)` | 检索 | 候选 200 → `MemoryRanker` 排序（复用现有纯函数） |
| `conflictFacts()` | **冲突候选行**（D-6 第 7 条） | `status IN ('ACTIVE','CANDIDATE') AND cardinality='SINGLE'`，且 `slot_hash` 在"同槽多 dedupe_key"集合里；按 `slot_hash, observed_at DESC` 排序 |
| `counts()` | 各状态计数 | 一条 `GROUP BY status` |
| `selfCheck()` | 10 组残留检查 | C0.2 §6.6，返回 `List<String>`（空 = 干净） |

**`ambiguous` / `boundary_unknown`**（C0.2 §4.10）在 `search`/`asOfWorld` 的返回类型上表达：

```kotlin
data class MemoryQueryResult(
    val items: List<MemoryItem>,
    val ambiguous: Boolean,          // 同 slot_hash + 不同 value + 至少一条 boundary_unknown
    val truncated: Boolean,          // token 预算截断
)
// 单条的 boundary_unknown 由 MemoryItem.boundaryUnknown 计算属性给出：
val boundaryUnknown get() = validFrom == 0L || timeConfidence == NONE || validUntil == 0L
```

### 7.9 Scope 与敏感级过滤（唯一一处拼接）

```kotlin
private fun scopeClause(q: MemoryQuery): Pair<String, Array<String>> {
    require(q.scopes.isNotEmpty()) { "scopes 不能为空：拒绝检索，而不是返回全部" }
    val parts = mutableListOf("(i.scope_type='GLOBAL')")
    val args = mutableListOf<String>()
    for (s in q.scopes) {
        require(MemoryGuard.isValidScope(s)) { "非法 scope: $s" }
        if (s.type == ScopeType.GLOBAL) continue
        parts += "(i.scope_type=? AND i.scope_id=?)"
        args += s.type.name; args += s.id
    }
    return "(" + parts.joinToString(" OR ") + ")" to args.toTypedArray()
}
```

四个约束缺一不可（C0.2 §13.6）：`status` / 生效区间 / `sensitivity <= max` / scope OR 列表。**降级到 LIKE 时这四条一个都不许少**。

### 7.10 错误模型

| 错误 | 类型 | 调用方怎么处理 |
|---|---|---|
| scope 非法 / 为空 | `IllegalArgumentException` | 编程错误，直接崩（debug）/ 记日志丢弃（release） |
| SECRET 写入 | **不抛**，返回 `Rejected(SECRET)` | 调用方按"没写进去"处理 |
| 状态迁移非法 | `IllegalStateException` | 编程错误 |
| 恢复时同值已存在 | **不抛**，返回 `AlreadyPresent` | UI 提示"已存在"（C3） |
| 迁移失败 | 抛 → `onUpgrade` 回滚 | 既有机制 |
| FTS 不可用 | **不抛**，`ftsMode` 降档 | 检索走 LIKE |
| 数据库只读 | 写方法抛 `SQLiteReadOnlyDatabaseException`（底层） | 不额外包装 |

**原则**：只有"编程错误"抛异常；"用户/环境导致的可预期状况"返回结果类型。这样调用方可以穷举 `when`，不会漏分支。

### 7.11 明确不做的事（C2 及以后）

| 不做 | 归属 |
|---|---|
| 从通知/对话自动抽取记忆（`MemoryExtractor` 已有，C1 只沿用其输出） | C2（编排） |
| 用户确认/否决的 UI 与批量流程 | C2 |
| `MemoryContextBuilder`（检索结果注入 prompt） | C3 |
| 自动冲突裁决（"选一个"） | C2 |
| 定期整理/合并/衰减 | C4 |
| 历史聊天回填 | C4 |
| 墓碑、硬删、`usage` 表、`ingest` 游标 | C3/C4 |
| PROJECT / CONVERSATION 的写入 | C2/C4 |

---

### 7.12 `Fact` 投影层与 `FactStatus` 兼容状态模型（D-6 落实）

#### 7.12.1 两套状态模型的分工（不可合并）

| | 权威模型 | 兼容模型 |
|---|---|---|
| 类型 | `MemoryStatus`（`MemoryItem.kt`，新增） | `FactStatus`（`MemoryTypes.kt`，修改） |
| 取值 | `ACTIVE` / `CANDIDATE` / `SUPERSEDED` / `EXPIRED` / `REJECTED` / `DELETED` | `ACTIVE` / `UNCONFIRMED` / `SUPERSEDED` / `EXPIRED` / `REJECTED` / `DELETED` / `UNKNOWN` |
| 存储 | `zhuli_memory_item.status`（6 个字符串，唯一权威） | **不存储**；运行时由投影函数从权威模型算出 |
| 谁写 | `MemoryStore` | 无人写 |
| 谁读 | `MemoryStore` / `MemoryQuery` / `MemoryMigrator` | `MemoryRepo` 的投影、`MemoryRanker`、UI 的 `Fact` 列表 |
| 生命周期 | 永久 | **C1-B 之后只减不增**：C4 删除 `MemoryRepo` 时一并消失 |

**为什么不能合并成一套**：`MemoryStatus` 的 `CANDIDATE` 与 `FactStatus` 的 `UNCONFIRMED` 是同一个概念的两个历史名字。合并意味着改 5 个生产调用方 + 15 个现有单测（§8.3 已拒绝）。兼容模型的价值就是"让旧调用方一行不改"，代价是必须有一层**显式、可测试**的投影。

**为什么不能"多加两个值就算完"**：如果只是加 `EXPIRED` / `REJECTED` 而不改 `of()`，未知值仍会变成 `ACTIVE`——那是 D-6 第 3 条明令禁止的。所以本次改动是**三个值 + 一个语义修正 + 一个冲突函数修正**，不是简单扩枚举。

#### 7.12.2 `MemoryTypes.kt` 的目标形态

```kotlin
enum class FactStatus {
    ACTIVE,

    /** 历史回填/低置信度：存下来但不参与检索、不覆盖任何现有记忆。对应 MemoryStatus.CANDIDATE。 */
    UNCONFIRMED,

    /** 被新值替代：曾经为真，现在不是当前值，但仍是历史。 */
    SUPERSEDED,

    /** 自然过期：世界时间越过 valid_until（任务到期、STATE 关闭），或时效耗尽。
     *  与 SUPERSEDED 的区别：没有"被谁替代"，是时间自己走到了尽头。 */
    EXPIRED,

    /** 用户/策略明确否决：不是"以前是、现在不是"，而是"这条本来就不该记"。
     *  与 DELETED 的区别：DELETED 是"删掉一条曾经接受的记忆"，REJECTED 是"从源头就不接受"。 */
    REJECTED,

    /** 用户删除（软删）。pre_delete_status 记录删除前的状态。 */
    DELETED,

    /** 兼容投影兜底：数据库里出现无法识别的状态值（手工改库、未来版本写入、数据损坏）。
     *  **非当前、不可自动激活、不参与检索与冲突配对**。绝不 fallback 到 ACTIVE。 */
    UNKNOWN;

    /** 只有 ACTIVE 是"当前为真"。EXPIRED / REJECTED / DELETED / UNKNOWN 全部为 false。 */
    val isCurrent: Boolean get() = this == ACTIVE

    /** 可以参与冲突配对的状态：当前值 + 待裁决候选。 */
    val isActionable: Boolean get() = this == ACTIVE || this == UNCONFIRMED

    companion object {
        /** 未知值（含 null、空串、大小写不符、未来新增值）一律 UNKNOWN。
         *  **禁止**写成 `?: ACTIVE` —— 那会把"过期/否决/未来状态"显示成当前值。 */
        fun of(raw: String?): FactStatus =
            entries.firstOrNull { it.name == raw } ?: UNKNOWN
    }
}
```

**改动清单（相对现状）**：

| 行 | 现状 | 改为 |
|---|---|---|
| 56–66 | 4 个枚举值 | 6 个业务值 + `UNKNOWN` |
| 68–70 | `fun of(raw: String) = entries.firstOrNull { … } ?: ACTIVE` | `fun of(raw: String?) = entries.firstOrNull { … } ?: UNKNOWN` |
| 新增 | — | `val isCurrent` / `val isActionable` 两个派生属性 |
| KDoc | 只描述 4 态 | 每个新值写"与相邻值的区别"（上面代码里的注释） |

**不做的改动**：`Fact` 的 16 个字段、`FactKind`、`SourceType`、`EntityKind`、`FactDraft`、`FactHit` 全部不动。`Fact.status` 的默认值仍是 `ACTIVE`。

#### 7.12.3 所有 `FactStatus` 使用方与处理方式（已核查，15 处）

`grep -rn "FactStatus" app/src` 的完整结果，逐处给出新状态下的行为。**这张表就是 D-6 第 6 条的答案**：

| # | 位置 | 现在怎么用 | 新状态下的处理 | 要改吗 |
|---|---|---|---|---|
| 1 | `MemoryTypes.kt:120` | `Fact.status` 默认 `ACTIVE` | 不变（新建 `Fact` 默认当前值） | 否 |
| 2 | `MemoryTypes.kt:69` | `of()` fallback `ACTIVE` | 改 `UNKNOWN` | **是** |
| 3 | `MemoryRepo.kt:478` | `read()` 用 `FactStatus.of(raw)` | **删掉 `of()`，改显式投影函数**（`CANDIDATE` → `UNCONFIRMED` 必须靠它） | **是** |
| 4 | `MemoryRepo.kt:116` | `correct()` 新行 `status = ACTIVE` | 转 `MemoryStore.correct()`，状态由 Store 定 | 随重写 |
| 5 | `MemoryRepo.kt:136` | `deleteFact` 写 `DELETED` | 转 `MemoryStore.softDelete()` | 随重写 |
| 6 | `MemoryRepo.kt:150` | `unconfirmed()` 查 `UNCONFIRMED` | 查新表 `status='CANDIDATE'` | 随重写 |
| 7 | `MemoryRepo.kt:157,162` | `confirmFact` 只认 `UNCONFIRMED` | 转 `MemoryStore.confirm()`；`UNKNOWN` 不放行（`WHERE status='CANDIDATE'`） | 随重写 |
| 8 | `MemoryRepo.kt:177,183,186` | `activeOf`/`active`/`activeCount` 查 `ACTIVE` | 改 `CURRENT`（多 `LB<=now` 与 `UB` 两个条件） | 随重写 |
| 9 | `MemoryRepo.kt:208,429` | `supersede`/`resolveConflict` 写 `SUPERSEDED` | 转 `MemoryStore`；`EXPIRED` 与 `SUPERSEDED` 不混用 | 随重写 |
| 10 | `MemoryRepo.kt:271-272` | `pendingVectorFacts` 的 JOIN 过滤 `ACTIVE` | 改 `CURRENT` | 随重写 |
| 11 | `MemoryRepo.kt:326` | `enqueueAllActive` 查 `ACTIVE` | 改 `CURRENT` | 随重写 |
| 12 | `MemoryRepo.kt:353` | `vectorText` 只对 `ACTIVE` 返回 | 改 `CURRENT`（与 10/11 一致，避免"入队了但取不到文本"） | 随重写 |
| 13 | `MemoryRanker.kt:97` | `search` 过滤 `ACTIVE` | **不变**：`UNKNOWN` / `EXPIRED` / `REJECTED` / `DELETED` 自动排除，正是要的 | 否 |
| 14 | `MemoryRanker.kt:112` | `conflicts` 过滤 `ACTIVE` | 改 `isActionable`（`ACTIVE` + `UNCONFIRMED`） | **是** |
| 15 | `MemoryBackfill.kt:20` | KDoc 提 `UNCONFIRMED` | 文案保留（`FactStatus.UNCONFIRMED` 仍是有效值），补一句"对应 `MemoryStatus.CANDIDATE`" | KDoc |

**不在这张表里的"使用方"**（已逐个核查，确认无依赖）：

| 文件 | 检查结果 |
|---|---|
| `bridge/ZhuliMemoryPage.kt` | grep 查 `UNCONFIRMED` / `SUPERSEDED` / `DELETED` / `FactStat` 四个字面量 = **0 命中**。UI 只消费 `active()` / `unconfirmed()` / `conflicts()` 的列表，不判状态字面量 → 新状态**零影响** |
| `agent/ZhuliAgentTools.kt` | 同上，0 命中（`status` 只出现在任务工具的 `it.status`，与 `FactStatus` 无关） |
| `memory/MemoryIndexer.kt` | 0 命中；只用 `activeCount()` 与向量方法 |
| `memory/MemoryExtractor.kt` / `MemoryEmbedding.kt` / `GemmaEmbedding.kt` | 0 命中 |
| `MemoryRankerTest.kt` | 用 `FactStatus.ACTIVE/DELETED/SUPERSEDED` 构造数据；新增值不破坏现有断言 |
| `MemorySqliteTest.kt:77,102,114` | 断言 `SUPERSEDED` / `DELETED`；语义未变，继续成立 |

**结论**：新状态对 UI 与 Agent 的运行时行为**没有任何影响**；唯一的行为变化在 `MemoryRanker.conflicts`（从"永远空"变成"真的有冲突"），那是修复而不是回归。

#### 7.12.4 投影矩阵与查询可见性（D-6 第 5 条）

**投影函数**（`MemoryRepo` 私有，唯一一处）：

```kotlin
/**
 * 权威状态 → 兼容状态。**禁止**用 FactStatus.of()：CANDIDATE 需要映射成 UNCONFIRMED，
 * 而 of("CANDIDATE") 会得到 UNKNOWN（正确但无用）。
 */
private fun projectStatus(raw: String): FactStatus = when (raw) {
    "ACTIVE"     -> FactStatus.ACTIVE
    "CANDIDATE"  -> FactStatus.UNCONFIRMED
    "SUPERSEDED" -> FactStatus.SUPERSEDED
    "EXPIRED"    -> FactStatus.EXPIRED
    "REJECTED"   -> FactStatus.REJECTED
    "DELETED"    -> FactStatus.DELETED
    else         -> FactStatus.UNKNOWN
}
```

**六态投影一一对应，不做任何合并**（这是 D-6 第 5 条的核心）：

| `zhuli_memory_item.status` | `FactStatus` | 在 `active()` | 在 `unconfirmed()` | 在 `history()` | 在 `search()` | 在 `conflicts()` |
|---|---|---|---|---|---|---|
| `ACTIVE` | `ACTIVE` | ✅ | ❌ | ✅ | ✅ | ✅ |
| `CANDIDATE` | `UNCONFIRMED` | ❌ | ✅ | ✅ | ❌ | ✅ |
| `SUPERSEDED` | `SUPERSEDED` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `EXPIRED` | `EXPIRED` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `REJECTED` | `REJECTED` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `DELETED` | `DELETED` | ❌ | ❌ | ✅ | ❌ | ❌ |
| 其它 / 空 | `UNKNOWN` | ❌ | ❌ | ✅ | ❌ | ❌ |

**四组语义的区别（写进 KDoc 与 UI 文案依据）**：

| 状态 | 一句话 | 与相邻状态的区别 | 用户能看到吗 |
|---|---|---|---|
| `ACTIVE` | 现在为真 | — | 记忆页主列表 |
| `EXPIRED` | 曾经为真，时间到了 | 不是"被谁替代"（`superseded_by=0`），是 `valid_until` 走到了 | 历史列表 |
| `REJECTED` | 从来没被接受 | 不是"删掉一条接受过的"（`deleted_at=0`），是源头否决 | 历史列表（C2 才产生） |
| `DELETED` | 接受过，用户删了 | `pre_delete_status` 记录删除前是什么 | 历史列表（默认折叠） |

> **C1 不产生 `REJECTED`**（没有 reject 入口，C2 才有）。但枚举值必须先有：否则 C2 一上线，`FactStatus.of("REJECTED")` 会把否决的记忆读成 `ACTIVE`——这正是 D-6 要提前堵住的坑。

#### 7.12.5 `MemoryRanker.conflicts` 的新语义（D-6 第 7 条）

**问题回顾**：R8/R9 把打不平的新值放进 `CANDIDATE`，槽内只留 1 条 `ACTIVE`。旧实现 `filter { status == ACTIVE }` 在槽内只剩一条时永远配不出对 → 记忆页的"需要你裁决"区域**静默消失**。

**新实现**（签名不变，纯函数）：

```kotlin
/**
 * 矛盾检测：同槽内"当前值"与"待裁决候选"值不同 → 需要用户裁决。
 *
 * v2 起（D-6 第 7 条）：
 * - 参与配对的状态是 ACTIVE + UNCONFIRMED（isActionable），不再只取 ACTIVE；
 * - 槽内有 ACTIVE 时，每个不同值的 UNCONFIRMED 都与它各成一对（UI 要逐条裁决）；
 * - 槽内全是 UNCONFIRMED（异常/连续 R9）时，退化为相邻两两配对；
 * - MULTI 槽位的排除不在这里做 —— 本函数拿不到 cardinality，
 *   由 MemoryStore.conflictFacts() 的 SQL 先过滤（见 §7.8）。
 */
fun conflicts(facts: List<Fact>): List<Pair<Fact, Fact>> {
    val actionable = facts.filter { it.status.isActionable }
    val out = ArrayList<Pair<Fact, Fact>>()
    for ((_, group) in actionable.groupBy { it.slot }) {
        val distinct = group
            .sortedByDescending { it.observedAt }
            .distinctBy { MemoryExtractor.norm(it.value) }
        if (distinct.size < 2) continue
        val actives = distinct.filter { it.status == FactStatus.ACTIVE }
        val candidates = distinct.filter { it.status == FactStatus.UNCONFIRMED }
        if (actives.isNotEmpty() && candidates.isNotEmpty()) {
            for (a in actives) for (c in candidates) out.add(a to c)
        } else {
            for (i in 0 until distinct.size - 1) out.add(distinct[i] to distinct[i + 1])
        }
    }
    return out
}
```

**配对数量示例**：槽内 1 ACTIVE + 3 个不同值的 CANDIDATE → **3 对**（旧实现 0 对）。UI 显示"需要你裁决（3 组）"，每组是"当前值 vs 某个候选"，用户可以逐条确认或删除。

**`MULTI` 槽位不误报**：`MemoryStore.conflictFacts()` 的 SQL 带 `cardinality='SINGLE'`，MULTI 槽（取件码、多笔账）根本不会进入 `conflicts()` 的输入。这是"存储语义留在 Store、纯函数只做配对"的分工。

**`EXPIRED` / `REJECTED` / `DELETED` / `UNKNOWN` 不参与**：`isActionable` 只含两个值，其余自动排除。特别是 `EXPIRED`——一条过期任务与当前值同槽不是"冲突"，是"历史"。

**与 `activateThreshold` 的交互（必须知道，不是缺陷）**：`MemoryBackfill` 用 `activateThreshold` 回填历史，会产生大量 `CANDIDATE`（预演库 20k 规模下约 1,810 条 `UNCONFIRMED`）。按本节语义，这些候选只要与同槽 ACTIVE 值不同，就会进入 `conflicts()`。后果：

| 后果 | 评估 |
|---|---|
| 记忆页的"需要你裁决"数量可能从 0 跳到几十上百组 | 符合"不静默丢失"的要求，但 UI 会拥挤 |
| 是否应该只展示 `R8`/`R9` 产生的（有 `CONFLICTS_WITH` 关系的）候选 | **不采纳**：① 依赖关系表会让"关系写失败"变成"冲突消失"，多一个失败点；② 回填候选与当前值同槽不同值，确实是两个值打架，用户有权知道；③ C1 的 `activateThreshold` 只被一次性回填使用，C2 的自动写入不走它 |
| 怎么缓解 | 不是隐藏冲突，而是 C2 的批量裁决 UI（多选确认/多选删除）+ 按"证据强度"排序。**这是 U-11** |

**`qualifier_key` 的已知限制**：`Fact.slot` = `subject|predicate`，不含 `qualifier_key`。C1 的 `qualifier_key` 恒为 `''`（迁移与运行时都不产生非空值），所以无差异；C2 引入 qualifier 后，`MemoryRanker` 需要改用 `slot_hash` 或给 `Fact` 加字段。**这条写进 §19.3 的遗留问题 U-7。**

#### 7.12.6 序列化 / 反序列化 / 数据库兼容规则（D-6 第 9 条）

| 方向 | 规则 | 测试 |
|---|---|---|
| 序列化（写库） | 只用 `status.name`；不写序号、不写自定义编码。`UNKNOWN` **不写回**（它只产生于读取） | `T-X03` |
| 反序列化（读库） | 权威表走 `projectStatus(raw)`（显式 7 分支）；旧 `zhuli_fact_v1` 若被读取则走 `FactStatus.of(raw)`（6 态 + UNKNOWN） | `T-X01` / `T-X02` / `T-X06` |
| 大小写 | 严格匹配 `name`。`"active"` → `UNKNOWN`（不静默纠正，避免"看起来对其实是脏数据"） | `T-X02` |
| 空值 | `null` / `""` → `UNKNOWN` | `T-X02` |
| 未知值 | → `UNKNOWN`；**禁止**映射到 `ACTIVE` | `T-X02` / `T-X04` |
| v16 库兼容 | 旧 4 个值（`ACTIVE`/`UNCONFIRMED`/`SUPERSEDED`/`DELETED`）迁移后投影不变 | `T-X05` |
| v16 库出现新值（手工改库） | 迁移按值映射：`EXPIRED`→`EXPIRED`，`REJECTED`→`REJECTED`，其它未知→`CANDIDATE` + 计数 | `T-X08` |
| 写回旧列 | C1-B 后**没有**任何写旧列的路径；若未来有人加，`UNKNOWN` 会以字符串 `"UNKNOWN"` 落库，再读仍是 `UNKNOWN`（不漂移） | `T-X03` |
| 序列化稳定性 | 枚举值顺序可读但不参与任何逻辑；**禁止**按 `ordinal` 存库 | review checklist |

#### 7.12.7 本节的禁止事项

| # | 禁止 | 理由 |
|---|---|---|
| 1 | `FactStatus.of()` 出现 `?: ACTIVE` | D-6 第 3 条；这正是 C0.2 缺陷 #2 的投影层翻版 |
| 2 | 用 `FactStatus.valueOf(raw)` | 未知值抛 `IllegalArgumentException`，让"一条脏数据"变成"记忆页崩溃" |
| 3 | 在投影里把 `EXPIRED` 合并成 `SUPERSEDED`、把 `REJECTED` 合并成 `DELETED` | D-6 第 5 条；合并后 UI 无法区分"过期"与"被替代"、"否决"与"删除" |
| 4 | 给 `FactStatus` 加 `CANDIDATE`（与 `UNCONFIRMED` 同义） | 两个同义值会让 `when` 出现两个分支做同一件事，且旧数据无法判断该用哪个 |
| 5 | 把 `UNKNOWN` 当作"待确认"放进 `unconfirmed()` | D-6 第 4 条；未知状态不可自动激活，也不该出现在"请用户确认"的列表里（用户没有可确认的对象） |
| 6 | 在 `MemoryRanker` 里判断 `cardinality` | 它拿不到；MULTI 排除是存储层职责（§7.8 的 `conflictFacts`） |
| 7 | 让 `MemoryRepo` 之外的类构造 `Fact` | `Fact` 是兼容视图，只能由投影函数产生；多一处构造就多一处状态映射漏判 |

---

## 8. 文件修改清单（具体到路径）

### 8.1 新增生产文件（7 个，全部在 `com/zhuli/memory/`）

| # | 路径 | 行数（估） | 归属阶段 | 内容 |
|---|---|---|---|---|
| 1 | `app/src/main/java/com/zhuli/memory/MemoryItem.kt` | 190 | B1 | `MemoryType` / `MemoryStatus` / `ConfidenceBasis` / `ScopeType` / `Cardinality` / `TimeConfidence` / `MemoryScope` / `MemoryItem` / `MemorySource` / `MemoryEvidence` / `MemoryDraft` / `InsertResult` / `DeleteResult` / `RestoreResult` |
| 2 | `app/src/main/java/com/zhuli/memory/MemoryText.kt` | 110 | B1 | `norm` / `searchText` / `queryExpr` / `dedupeKey` / `slotHash` / `sha256Hex` / `LB_SQL` / `UB_SQL` / `MS_PER_DAY` |
| 3 | `app/src/main/java/com/zhuli/memory/MemorySchema.kt` | 190 | B1+B3 | 7 段 DDL 常量 + `createAll` / `ensureAll` / `createFtsTable` / `createFtsTriggers` / `detectFtsMode` / `FtsModeCache` / `tableExists` |
| 4 | `app/src/main/java/com/zhuli/memory/MemoryGuard.kt` | 90 | B1 | `classify` / `excerpt` / `isValidScope` / `checkLength`（镜像 `SensitiveGuard`，不新建词表） |
| 5 | `app/src/main/java/com/zhuli/memory/MemoryPolicy.kt` | 140 | B4 | `rank` / `cardinalityOf` / `decide`（R4–R9）/ `R0` 的时间自洽修正 |
| 6 | `app/src/main/java/com/zhuli/memory/MemoryStore.kt` | 520 | B4 | 7 个写方法 + 9 个查询 + `selfCheck` + `syncFts` + `rebuildFts` |
| 7 | `app/src/main/java/com/zhuli/memory/MemoryMigrator.kt` | 250 | B2 | 13 步迁移 + 5 条时间断言 + 数量闭合 + 断链 + 合并 + 向量校验 |

小计 **1,490 行**（C0.2 估 1,420；增量来自 `InsertResult`/`DeleteResult` 等结果类型 ~40 行与 `FtsModeCache` ~30 行）。

> 比 C0.2 的 1,450 行预算多 40 行，仍在 §8.5 的容忍范围内（见该节的核算）。

### 8.2 修改生产文件（**恰好 4 个**，D-6/D-7 后由 2 个增至 4 个）

| # | 路径 | 改动 | 行数（估） |
|---|---|---|---|
| 1 | `app/src/main/java/com/zhuli/core/InboxDatabase.kt` | ① `DB_VERSION = 17`；② `createCoreTables` 拆成 `createCoreTablesV15` + `createFactTableV15`（**X-1 / D-7**，§4.8.3）；③ `onCreate` 内调用点 `:317` 换成 `createCoreTablesV15` + `MemorySchema.attach(prefs)` + `MemorySchema.createAll(db, withUnique=true)`；④ `onOpen` 内调用点 `:339` 换成 `createCoreTablesV15` + `MemorySchema.ensureAll(db)`；⑤ `onUpgrade` 内调用点 `:787`（`oldVersion<15` 分支）换成 `createCoreTablesV15` + `createFactTableV15`；⑥ `upgradeSteps` 末尾加 `if (oldVersion < 17) { MemorySchema.createAll(db, withUnique=false); MemoryMigrator.migrate(db, now) }` | **+34 / −10** |
| 2 | `app/src/main/java/com/zhuli/memory/MemoryRepo.kt` | 全量重写为投影层（27 个签名不变）；删掉 `insert`/`reinforce`/`supersede`/`upsertEntity` 等 5 个写旧表的私有方法（§4.8.4）；新增 `projectStatus()` 显式七分支（§7.12.4） | 485 → **~175**（净减 ~310） |
| 3 | `app/src/main/java/com/zhuli/memory/MemoryTypes.kt` | **D-6**：`FactStatus` 加 `EXPIRED` / `REJECTED` / `UNKNOWN`；`of(raw: String?)` 改为 `?: UNKNOWN`；新增 `isCurrent` / `isActionable`；KDoc 写清四组语义区别（§7.12.2） | 154 → **~172**（+18 / −6） |
| 4 | `app/src/main/java/com/zhuli/memory/MemoryRanker.kt` | **D-6 第 7 条**：`conflicts` 的过滤集合改为 `isActionable`；配对规则改为"ACTIVE × 每个候选"优先；KDoc 说明 MULTI 排除在 Store 层（§7.12.5） | 124 → **~132**（+8 / −3） |

改动总量：**+60 / −329**（净减 269 行，主要是 `MemoryRepo` 从 485 行缩到 175 行）。

> 为什么 `MemoryRanker.kt` 必须改而不能绕：D-6 第 7 条的原话是"`MemoryRanker.conflicts` 必须支持新记忆冲突模型，不能因为冲突败方处于 `CANDIDATE` 而静默丢失冲突展示"。绕到 `MemoryRepo` 里另写一套配对，等于让 `MemoryRanker.conflicts` 继续对 `CANDIDATE` 视而不见——任何其他调用方（或未来的 C2 编排）用它都会踩同一个坑。


### 8.3 删除文件

**0 个。**

理由：C0.2 没有要求删除任何文件；`zhuli_fact` 的"删除"是**运行时改名**，不是删源码。`MemoryTypes.kt` / `MemoryExtractor.kt` / `MemoryRanker.kt` 全部保留（§10.3 禁止清单）。

唯一可以讨论的是 `MemoryRepo.kt` 是否该整体删除、由调用方直连 `MemoryStore`——**不删**，因为 5 个生产调用方 + 15 个现有单测都按 `Fact` 模型编写，删了等于把 C1 扩成"改 5 个调用方 + 重写 15 个测试"，直接违反 §10.5 的 diff 预算。

### 8.4 新增/修改测试文件

| # | 路径 | 类型 | 行数（估） | 覆盖（逻辑条数） |
|---|---|---|---|---|
| 1 | `app/src/test/java/com/zhuli/memory/MemoryTextTest.kt` | 新增 | 140 | L1 纯函数：`norm` / `searchText` / `queryExpr` / `dedupeKey` / `slotHash`（含 CJK、全角、标点、单字） |
| 2 | `app/src/test/java/com/zhuli/memory/MemorySchemaTest.kt` | 新增 | 200 | DDL 一致性（40 列 + **15 显式 + 4 自动**索引，§4.5）、建表幂等、`detectFtsMode` 三档 + 缓存六道门（§6.4）、`T-M40` |
| 3 | `app/src/test/java/com/zhuli/memory/MemoryMigratorTest.kt` | 新增 | 580 | `T-M01`–`T-M10`、`T-M36`–`T-M38`（**13**）+ `T-M41`–`T-M45`（**5**）+ `T-M46`（**1**）= **19** |
| 4 | `app/src/test/java/com/zhuli/memory/MemoryStoreTest.kt` | 新增 | 790 | `T-S11`–`T-S20`（10）+ `T-S39`（1）+ `T-S21`–`T-S24`（4）+ `T-D31` / `T-D33` / `T-D44`–`T-D49`（8）+ `T-N01`–`T-N05`（5）= **28** |
| 5 | `app/src/test/java/com/zhuli/memory/MemoryTimeQueryTest.kt` | 新增 | 230 | `T-T21`–`T-T24`、`T-T40`–`T-T43`（8） |
| 6 | `app/src/test/java/com/zhuli/memory/MemoryScopeTest.kt` | 新增 | 200 | `T-C25`–`T-C30`、`T-C57`（7） |
| 7 | `app/src/test/java/com/zhuli/memory/MemoryFactStatusTest.kt` | **新增（D-6）** | 300 | `T-X01`–`T-X12`（12）；含 `MemoryFactStatusTriggerTest` / `…AppSyncTest` 两个子类（§10.9.3） |
| 8 | `app/src/test/java/com/zhuli/memory/MemoryTestFixture.kt` | 新增（**非测试类**） | 160 | `insertRaw` / `seedLegacy` / `seedItem`（C0.2 §14.3）+ `runConcurrently`（`T-N01`–`T-N05` 的 `CountDownLatch` 夹具） |
| 9 | `app/src/androidTest/java/com/zhuli/memory/MemoryMigrationTimingTest.kt` | 新增 | 160 | D2 设备耗时（1k/5k/20k）+ 迁移后不变量 + `thread=` 字段 |
| 10 | `app/src/test/java/com/zhuli/memory/MemorySqliteTest.kt` | **修改** | +20 / −5 | `v15 库升级` 用例断言 v17；新增 `zhuli_fact` 不存在的断言；`activeCount` 仍为 1 |

**小计（v3 重算）**：

| 口径 | 行数 |
|---|---|
| 新增测试代码（不含 fixture） | 140 + 200 + 580 + 790 + 230 + 200 + 300 + 160 = **2,600** |
| 含 fixture | **2,760** |
| 修改 | 1 个文件（`MemorySqliteTest.kt`，+20/−5） |
| 逻辑用例合计 | 5 + 1 + 19 + 28 + 8 + 7 + 12 = **75**（+5 组纯函数不计编号） |

> **v3 的 +420 行从哪来**（相对 v2 的 2,340）：`MemoryMigratorTest` +40（`T-M46`）、`MemoryStoreTest` +310（`T-S21`–`T-S24` 约 110 + `T-N01`–`T-N05` 约 200）、`MemoryFactStatusTest` +20（双类结构）、`MemorySchemaTest` +20（19 个索引的断言）、`MemoryTestFixture` +30（并发夹具）。
>
> **测试预算上限从 2,400 调到 2,800**：不是"范围扩张"，是 §10.12 那 10 条用例的必然成本。C1-B5 用 `git diff --stat` 核对，超 2,800 必须逐条解释，不能只说"写多了"。


### 8.5 diff 预算核算与自检

| 项 | 上限 | 本方案估算 | 是否超 |
|---|---|---|---|
| 生产新增 | ≤1,500（v2 调整，v1 为 1,450） | 1,490 | 否（余 10） |
| 生产修改 | **恰好 4 个文件** | 4 个（`InboxDatabase` +34/−10、`MemoryRepo` 485→175、`MemoryTypes` +18/−6、`MemoryRanker` +8/−3） | 否 |
| 测试新增 | ≤2,800（v3 调整，v2 为 2,400） | ~2,760（含 fixture） | 否（余 40） |
| 测试修改 | — | 1 个文件（`MemorySqliteTest.kt`，+20/−5） | — |
| 触碰 C0.2 §10.3 禁止清单 | 0，除 D-6 明确放行的 `MemoryTypes.kt` / `MemoryRanker.kt` | 0 | 否 |
| 数据库版本 | 只 +1 | 16→17 | 否 |

> **`MemoryTypes.kt` 的"禁止清单"冲突怎么处理**：C0.2 §10.3 写的是"`MemoryTypes.kt` 不动"，D-6 正式决定扩展 `FactStatus`。D-6 是**后出的、更具体的决策**，按"后出优先 + 具体优先"覆盖该条；C0.2 文档本身不改（它是冻结的架构依据），差异记在修订对照表 v3 的 §3。`MemoryRanker.kt` 同理（D-6 第 7 条）。

#### 8.5.1 两个白名单必须分开（P1-7）

v2 在 §9.1 与 §15 都写"`git diff` 只含 7 个文件（4 新增 + 3 修改）"，但 B1 阶段实际会动**测试文件**（`MemoryTextTest` / `MemorySchemaTest` / `MemoryFactStatusTest` 的解析部分 / `MemorySqliteTest` 的断言）。把生产与测试混在一个数字里，验收时无法判断"多出来的是测试还是生产"。

**B1 阶段的完整白名单（11 个文件）**：

| 类别 | 数量 | 文件 |
|---|---|---|
| **生产新增** | 4 | `MemoryItem.kt` / `MemoryText.kt` / `MemorySchema.kt` / `MemoryGuard.kt` |
| **生产修改** | 3 | `MemoryTypes.kt` / `MemoryRanker.kt` / `InboxDatabase.kt` |
| **测试新增** | 3 | `MemoryTextTest.kt` / `MemorySchemaTest.kt` / `MemoryFactStatusTest.kt`（解析部分先绿） |
| **测试修改** | 1 | `MemorySqliteTest.kt`（v15 升级断言） |
| **合计** | **11** | — |

**B2 阶段追加**：`MemoryMigrator.kt`（生产新增）+ `MemoryMigratorTest.kt`（测试新增）→ 13 个。
**B3 阶段追加**：无新文件（`MemorySchema.kt` 已含 `detectFtsMode` / `FtsModeCache`）。
**B4 阶段追加 7 个**：`MemoryPolicy.kt` / `MemoryStore.kt`（生产新增）、`MemoryRepo.kt`（生产修改，改投影层）、`MemoryTimeQueryTest.kt` / `MemoryScopeTest.kt` / `MemoryTestFixture.kt` / `MemoryStoreTest.kt`（测试新增）→ **20 个**。
**B5 阶段追加**：`MemoryMigrationTimingTest.kt`（androidTest 新增）→ **21 个 = 7 生产新增 + 4 生产修改 + 9 测试新增 + 1 测试修改**（`MemorySqliteTest`）。

> **v3 在这里修了一个加减错误**（P1-7 的连带问题）：v2 写"19 个 = 7 + 4 + 8 + 1"，但 7+4+8+1=20，等式本身不成立；且 B4 的追加清单漏了 `MemoryStoreTest.kt` 与 `MemoryRepo.kt`，B1 的"生产修改 3 个"里也没有 `MemoryRepo`（它在 B4 才动）。逐阶段累加与分类汇总两条路都必须得到 **21**，自检命令 #3 的期望值同步改成 10。这类"总数与明细对不上"正是 P1-7 要求分开白名单的原因——只给一个总数，错了也看不出来。

> 注意 `MemoryStoreTest.kt` 在 B4 追加，它的 28 条用例里有 4 条 `T-S21`–`T-S24` 需要先有 `syncFts`（B3）与 `MemoryStore`（B4）——顺序不能倒。

#### 8.5.2 自检命令（v3 修正期望值）

**共 17 条命令**：第 `0)` 条是 `git add -N` 的准备步骤（不做它，后面所有 `git diff` 都看不到 1,490 行新增，`--name-only` 也只数到 11 之外的 0 个），其余 16 条是自检。每条都写了期望值，**任何一条不符都停下来解释，不许"看起来没问题"就过**。

```bash
# 0) 必须先把新增文件放进 diff 视野，否则下面所有计数都会少 1,490 行
git add -N app/src

# 1) 生产文件的全部改动（7 新增 + 4 修改 = 11）
git diff --name-only HEAD -- app/src/main | wc -l          # 期望 11
git diff --name-only HEAD -- app/src/main                   # 逐行核对是否都在白名单

# 2) 生产改动量（含新增的 1,490 行）
git diff --stat HEAD -- app/src/main
# 期望：11 files changed, 约 +1,550 / −329
# （+60 是 4 个修改文件的增量，+1,490 是 7 个新增文件）

# 3) 测试文件（9 新增 + 1 修改 = 10）
git diff --name-only HEAD | grep -c 'src/test\|src/androidTest'   # 期望 10

# 4) 出了白名单的文件（应为空）
git diff --name-only HEAD | grep -v 'src/test\|src/androidTest\|docs/' | grep -v 'app/src/main'

# 5) 未跟踪/已暂存状态（`-N` 之后显示 AM，不是 ??）
git status --porcelain -- app/src/main | grep -c '^AM'      # 期望 7（新增）
git status --porcelain -- app/src/main | grep -c '^ M'      # 期望 4（修改）

# 6) 结构断言
grep -rn "TABLE_FACT" app/src/main/java/com/zhuli           # 期望 2 个文件：InboxDatabase（createFactTableV15）+ MemoryMigrator
grep -rn "zhuli_fact_v1" app/src/main                       # 期望 ≤3 处（Migrator + InboxDatabase 注释）
grep -rn "MemoryTestFixture" app/src/main                   # 期望 0
grep -n "DB_VERSION" app/src/main/java/com/zhuli/core/InboxDatabase.kt  # 期望 1 处 =17
grep -rn "fun createCoreTables\b" app/src/main              # 期望 0（旧函数名必须消失）
grep -rn "createCoreTablesV15\|createFactTableV15" app/src/main | wc -l  # 期望 5（2 定义 + 3 调用）
grep -rn "?: ACTIVE" app/src/main/java/com/zhuli/memory     # 期望 0（D-6 第 3 条）
grep -rn "FactStatus.valueOf" app/src/main                  # 期望 0
grep -rn 'status=="ACTIVE"\|status == "ACTIVE"' app/src/main  # 期望 0（查询一律走 CURRENT）
```

**v2 的三个期望值是错的，v3 逐个修正**：

| 命令 | v2 期望 | v3 期望 | 为什么错 |
|---|---|---|---|
| `git diff --name-only -- app/src/main \| wc -l` | 4 | **11** | `git add -N` 后新增文件也进 diff；4 只是"修改"的数量 |
| `git diff --stat -- app/src/main` | ~+60/−329 | **~+1,550/−329** | 同上，漏了 7 个新增文件的 1,490 行 |
| `git status --porcelain \| grep '^??'` | 7 个 `??` | **7 个 `AM`** | `git add -N` 之后状态是 `AM`（已 intent-to-add），不再是 `??` |


### 8.6 文档

| 路径 | 动作 |
|---|---|
| `docs/批次C1-B_记忆系统基础设施实施方案.md` | **本文**（C1-B 开工前交付；v2 落实 D-6/D-7） |
| `docs/批次C1-B_方案决策修订记录.md` | **v2 新增**（D-6/D-7 决策原文、落实映射、差异清单、测试重算） |
| `docs/批次C1-B_实施与验收报告.md` | C1-B5 产出（不在本轮） |

`docs/批次C0.2_*.md` / `docs/批次C1-A_*.md` **不改**（C0.2 是冻结的架构依据；C1-A 是已完成的事实记录）。D-6 覆盖 C0.2 §10.3 的"`MemoryTypes.kt` 不动"这一条，差异记在修订记录，不回写 C0.2。


---

## 9. 五个子阶段的实施顺序

### 9.1 C1-B1｜数据库结构

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | `MemoryTypes.kt`：**D-6** 加 `EXPIRED`/`REJECTED`/`UNKNOWN`，`of()` 改 `UNKNOWN`，加 `isCurrent`/`isActionable` | `MemoryFactStatusTest` 的解析与投影部分先绿 |
| 2 | `MemoryRanker.kt`：`conflicts` 过滤改 `isActionable` + ACTIVE×候选配对 | `MemoryRankerTest` 原有 6 例仍绿 + `T-X09`/`T-X10` |
| 3 | `MemoryItem.kt`：7 个枚举（含 `MemoryStatus` 六态）+ 6 个数据类 + **3 个 sealed 结果类型**（`InsertResult` 5 变体 / `DeleteResult` 3 变体 / `RestoreResult` 5 变体，共 13 个变体） | 可编译；`when` 穷举无 `else` |
| 4 | `MemoryText.kt`：5 个纯函数 + `LB_SQL`/`UB_SQL` | `MemoryTextTest` 全绿 |
| 5 | `MemorySchema.kt`：7 段 DDL + `createAll(withUnique)` + `ensureAll` + `tableExists` | `MemorySchemaTest` 建表部分全绿 |
| 6 | `MemoryGuard.kt`：分级、摘录、scope 校验 | 与 `SensitiveGuardTest` 的判定对齐（复用同一批样本） |
| 7 | `InboxDatabase.kt`：`DB_VERSION=17`、**D-7 拆分** `createCoreTablesV15` + `createFactTableV15`、三个调用点改造（§4.8.2）、`onCreate` 接入 `MemorySchema.createAll` | 新装库能建出 7 张表、无 `zhuli_fact` |
| 8 | 验证 | `MemorySchemaTest` + `MemorySqliteTest` 的 v15 升级用例 + `T-M40` |

**完成条件**：
- 新装（`ctx.deleteDatabase` 后打开）有 7 张表 + **15 个显式索引 + 4 个自动索引**，`user_version=17`，**无** `zhuli_fact`、**无** `zhuli_fact_v1`（`T-M40`）；
- 重复 `onOpen` 三次不报错、对象数不变（幂等）；
- `MemoryTextTest` 的 5 个函数在 CJK / 全角 / 标点 / 单字 / 超长 5 类样本上全绿；
- **`FactStatus` 的 7 个值（6 业务 + `UNKNOWN`）解析正确，未知输入一律 `UNKNOWN`，`isCurrent`/`isActionable` 的四个真值组合全对**（`T-X01`/`T-X02`/`T-X04`）；
- **`git diff` 只含 §8.5.1 的 11 个文件**（生产 7 = 4 新增 + 3 修改；测试 4 = 3 新增 + 1 修改）。多一个都要解释，特别是不能出现 `MemoryStore.kt` / `MemoryMigrator.kt`（它们属于 B2/B4）。


### 9.2 C1-B2｜v16→v17 数据迁移

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | `MemoryMigrator.kt` 骨架：**第 0 步断言（`oldVersion` 分支 + `journal_mode` 拒绝 WAL）** + 13 步空实现 + `MigrationReport` | 可编译，跑通空迁移 |
| 2 | 逐行映射（§5.3 的 40 列）+ **逐行账本（§5.9 的 `RowOutcome`）** | 真实副本 2 行迁完；6 类终态计数正确 |
| 3 | 来源与证据（§5.5，含 SECRET 先判后写） | `evidence` 行数 = `inserted`；SECRET 行零写入 |
| 4 | 替代链 + 断链（§5.7） | edge 库的断链行被修 |
| 5 | 合并（§5.6） | 5k 库的重复组被合并 |
| 6 | 向量校验（§5.8） | 真实库 1 条保留 |
| 7 | FTS 灌数据 + 触发器 + 唯一索引 + 改名 | `T-M09` 通过 |
| 8 | 账本闭合 + 5 条时间断言 + 第 6 条状态断言 | 4 档库全 0（第 3 条允许告警） |
| 9 | `sqlite_sequence` 对齐（`INSERT OR REPLACE`）+ `T-M46` | 新插入 id 不冲突 |
| 10 | `InboxDatabase.upgradeSteps` 接入（含备份位置不变、`onUpgrade` 内不做 checkpoint） | v16→v17 自动发生 |

**完成条件**：
- 4 档库（2 / 11 / 5,002 / 20,002 行）全部 `inserted + quarantined + skippedNoSubject + skippedNoValue + failed = src`，且 `failed=0`、`merged ≤ inserted`；
- C0.2 §6.6 的 10 组（11 条 SQL）在 4 档库上全 0；
- 5 条时间断言：1/2/4/5 为 0，第 3 条 ≤ 已知值且记入 `vf_mismatch`；第 6/6b 条（状态不变量，含 `NULL`）为 0；
- `PRAGMA integrity_check` = `ok`；`sqlite_sequence.seq >= MAX(item.id)` 且新插入 id 不冲突（`T-M46`）；
- 故障注入 5 种全部回滚到 v16（§11.3）；
- `oldVersion=14` / `15` / `16` 三条升级路径各跑一次，全部成功（P0-7：v14/v15 不得因版本断言失败）；
- 设备耗时 5k ≤ 3s（§5.13），超门槛按 §5.13.1 处置（**不得**改成分批提交）。

### 9.3 C1-B3｜FTS 探测与同步

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | `FtsModeCache`（`SharedPreferences` 三键） | 读写可测 |
| 2 | `detectFtsMode` 七步（§6.2） | 三档可复现 |
| 3 | `createFtsTable` / `createFtsTriggers` / `syncFts` / `rebuildFts` | 幂等可测 |
| 4 | `MemorySchema.attach(prefs)` 接入 `InboxDatabase` | 探测只跑一次 |
| 5 | 负向验证：在 `androidTest` 里人为把 `trusted_schema` 关掉（C1-A 的 step8 复用） | 探测返回 `APP_SYNC` 而不是 `TRIGGER` |

**完成条件**：
- 同一台设备连续打开 5 次，探测只执行 1 次（日志计数）；
- **缓存的六道门逐条可验**（§6.4）：① 无缓存 → 重探；② 枚举值被改写 → 重探；③ `DB_VERSION` 变 → 重探；④ `sqlite_version` 变 → 重探；⑤ 缓存说 `TRIGGER` 但**手工删掉 1 个触发器** → 重探且结论降为 `APP_SYNC`；⑥ 缓存说 `TRIGGER` 但**手工 `DROP` 掉 FTS 表** → 重探且结论降为 `NONE`；
- 人为制造坏环境（`PRAGMA trusted_schema=OFF`）时，探测返回 `APP_SYNC`（**这是最关键的一条**，它就是 C1-A 教训的落地）；
- `ftsMode=NONE` 时检索走 `LIKE` 且 **`sensitive` / `scope` / `status` / 生效区间四个约束**仍在（v2 写"三个"，漏了生效区间）；
- `LIKE` 的用户输入已转义：搜 `100%` 与 `a_b` 不产生通配符误命中（§6.3.1）；
- `selfCheck` 的第 1、2 条在"人为删掉一条 FTS 行"后能报出来。

### 9.4 C1-B4｜Store / Manager

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | `MemoryPolicy`（rank / cardinality / R4–R9） | 纯函数单测 |
| 2 | `MemoryStore.insert`（R0+R2+R3+R4+R5+R7+R8+R9 + 证据 + revision） | `T-S11`–`T-S20` |
| 3 | `correct` / `confirm` / `resolveConflict` | `T-S16` / `T-S18` |
| 4 | `softDelete` / `restore` | `T-D31` / `T-D44`–`T-D49` |
| 5 | 查询：`current` / `asOfWorld` / `asOfBelief` / `search` / `candidates` / `history` / **`conflictFacts`** | `T-T21`–`T-T43` / `T-X11` |
| 6 | `MemoryRepo` 重写为投影层（27 个签名）+ `projectStatus()` 显式七分支 | 现有 15 个 `MemorySqliteTest` 不动就绿 |
| 7 | `MemoryTestFixture` + scope 用例 | `T-C25`–`T-C30` / `T-C57` |
| 8 | `selfCheck` 10 组 | 空库返回空列表 |
| 9 | **D-6 六态投影与冲突用例** | `T-X05`–`T-X08`、`T-X12` |
| 10 | **FTS 写入职责（§6.3 的 W-1/W-2/W-3）** | `T-S21`–`T-S24` |
| 11 | **并发 5 条（§7.5.3）** | `T-N01`–`T-N05` |

**完成条件**：
- `MemoryRepo` 的 27 个签名逐条对照无变化（§8.5 的 review checklist）；
- 现有 `MemorySqliteTest` 的 15 例**只改 1 处**（v15 升级用例的版本断言 + `zhuli_fact` 不存在）就全绿——这是"兼容层有效"的最强证据；
- `T-C57` 通过：`PROJECT`/`CONVERSATION` 写入抛 `IllegalArgumentException`；
- `selfCheck()` 在干净库返回 `emptyList()`，在人为破坏的 4 种场景各返回 1 条；
- **六态投影一一对应、无合并**（`T-X06`）；`active()`/`unconfirmed()`/`history()`/`search()`/`conflicts()` 的可见性矩阵与 §7.12.4 逐格一致（`T-X07`）；
- **`conflicts()` 在"1 ACTIVE + 1 CANDIDATE"下返回 1 对**（旧实现返回 0 对，`T-X09`）；
- **FTS 三档写入职责全绿**（`T-S21`–`T-S24`）：`TRIGGER` 不双写、`APP_SYNC` 7 条路径不漏写、`NONE` 仍写 `search_text` 且四约束不丢；
- **并发 5 条全绿**（`T-N01`–`T-N05`，每条跑 50 次，0 次出现两条 ACTIVE / 未捕获 `SQLiteConstraintException`）；
- **B4 结束时的执行计数**：逻辑 75 条 / 执行 126 次（§10.11.2）。


### 9.5 C1-B5｜回归与验收

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | 全量 `:app:testDevDebugUnitTest` | **XML `tests` ≥ 457**（逻辑 75 条 / 执行 126 次，§10.11） |
| 2 | `:app:connectedDevDebugAndroidTest` | D1–D4、D6、D7、D9、D10 |
| 3 | B1 的 12 个验收场景 | D8 |
| 4 | 飞行模式手测 | D5 |
| 5 | `git diff --stat` / `--check` / `--name-only` | §8.5 的自检清单 |
| 6 | 写 `docs/批次C1-B_实施与验收报告.md` | 含 D2 实测耗时 |

**完成条件**：§16 的 9 条退出标准全部满足，且每条有证据。

### 9.6 依赖与不可并行项

| 关系 | 说明 |
|---|---|
| B1 → B2/B3/B4 | 都要 `MemoryItem` / `MemorySchema` / `MemoryText` |
| B2 ∥ B3 | 互不引用（迁移自己建 FTS，不复用探测结果） |
| B3 → B4 | `MemoryStore` 要按 `ftsMode` 分支 |
| B2 → B4 | `MemoryRepo` 的 `byId` 等要读迁移后的数据（测试数据由 fixture 造，但表结构来自 B2） |
| B4 → B5 | 回归要跑完整实现 |

**唯一可以完全并行的是 B2 与 B3**。B2 的开发者不需要知道 `detectFtsMode` 怎么写，B3 的开发者不需要知道迁移怎么映射——两条线在 `MemorySchema` 的 DDL 常量上汇合（B1 已定）。

---

## 10. 测试矩阵

### 10.1 分层与文件分配

| 层 | 手段 | 文件 | 逻辑条数 | 执行次数 |
|---|---|---|---|---|
| L0 能力门槛 | Robolectric + androidTest | `MemoryFtsRobolectricTest` / `MemoryFtsInstrumentedTest`（C1-A 已交付） | 7（已完成） | 18（已完成） |
| L1 纯函数 | JVM 单测 | `MemoryTextTest` + `MemoryFactStatusTest` 的解析部分 | 5 组 + 4 条（`T-X01`–`T-X03`、`T-X10`） | 同左（不双跑） |
| L2 真实 SQLite | Robolectric `NATIVE` | `MemoryMigratorTest` / `MemoryStoreTest` / `MemoryTimeQueryTest` / `MemoryScopeTest` / `MemorySchemaTest` / `MemoryFactStatusTest` | **75** | **126** |
| L3 设备 | 模拟器 | `MemoryMigrationTimingTest` + 手工 | D1–D10 | — |
| L4 回归 | 现有套件 | `MemorySqliteTest` + B1 的 12 场景 | 331（XML 基线） | 331 |

> 75 = 架构用例 47（A/B/C/D/E 组）+ D-6 的 12（`T-X01`–`T-X12`）+ D-7 的 6（`T-M40`–`T-M45`）+ v3 的 10（`T-M46` / `T-S21`–`T-S24` / `T-N01`–`T-N05`）。126 的逐组明细见 §10.11.2。
>
> `T-X01`/`T-X02` 是纯函数，但为了"一个决策一个文件"的追溯性，放在 `MemoryFactStatusTest` 里统一维护，不拆到 `MemoryTextTest`。


### 10.2 A 组｜迁移（13 条，`MemoryMigratorTest`）

| # | 用例 | 断言（逐字照 C0.2 §11.2） | 造数据方式 |
|---|---|---|---|
| T-M01 | v15 库直接打开 | 事实不丢、`activeCount=1`、value 一致 | `MemorySqliteTest` 现有用例（改造） |
| T-M02 | 真实副本迁移 | 行数一致、替代链 1→2、向量指向 2、FTS 2 行 | `seedLegacy(2)` 复刻真实库形态 |
| T-M03 | 断链 `superseded_by` | 清 0，`invalidated_at` 保留 | 造 `superseded_by=999999` |
| T-M04 | SECRET 行 | 不迁入，`quarantined=1`，总数闭合 | 造 16 位卡号 |
| T-M05 | 空 subject / 空 value | 跳过，**分别**计入 `skippedNoSubject` / `skippedNoValue`（不是笼统的 `skipped`），且两者之和与本表两行相等 | 造 2 行空值（一行空 subject、一行空 value） |
| T-M06 | 同键重复合并 | 保留强证据、区间取最早起点、保留行得 SUPPORTING **且**被合并行 PRIMARY 仍在、`SAME_AS` | 造 USER_STATED + INBOX 同值 |
| T-M07 | MULTI 槽位两个值 | 都 ACTIVE，不合并 | 造两个取件码 |
| T-M08 | 唯一索引建立失败 | 抛 → 回滚 → `user_version=16` | 人为造重复且跳过合并（直接插） |
| T-M09 | 迁移后表名 | `zhuli_fact` 不存在，`zhuli_fact_v1` 存在且 0 写入 | 迁移后查 `sqlite_master` |
| T-M10 | 二次打开 | `onOpen` 不重复建表、不重复插数据 | 连开两次 |
| T-M36 | 旧 `valid_from` 不得升级为确定时间 | **每一行** `time_confidence='NONE'`，含 `valid_from>0` 的行 | `seedLegacy` |
| T-M37 | 旧 `valid_to` 的去向 | SUPERSEDED 行 `valid_until=0`、`invalidated_at=原 valid_to`；DELETED 行 `valid_until=0`、`invalidated_at=原 updated_at` | 造两组 |
| T-M38 | 未知写入方的告警 | 人造 `valid_from≠observed_at`：不中止、仍 `NONE`、`vf_mismatch+1`、原值保留 | 直接 UPDATE 造 |

**D-7 建表与升级回归（5 条，追加在 A 组之后）**：

| # | 用例 | 断言 | 造数据方式 |
|---|---|---|---|
| T-M41 | v16→v17 后关闭并重开（3 次） | `zhuli_fact` 始终不存在；`zhuli_fact_v1` 行数不变；`user_version` 始终 17；无异常 | `seedLegacy(11)` + 连续 `InboxDatabase(ctx)` 3 次 |
| T-M42 | 旧写路径 fail-loud | 迁移后 `INSERT INTO zhuli_fact(...)` 抛 `SQLiteException`，消息含 `no such table`；**不**静默成功 | `db.writableDatabase.execSQL` 直写 |
| T-M43 | `onOpen` 幂等 | 连开 3 次：`sqlite_master` 的对象名集合与数量不变；`COUNT(*)` 三张主表不变 | 同 `T-M41`，比较三次快照 |
| T-M44 | 重开后新记忆数据完整 | 逐列抽检 12 列 + 向量行数 + evidence 行数；与重开前一致 | `seedLegacy` 覆盖 ACTIVE/DELETED/SUPERSEDED/UNCONFIRMED 各一 |
| T-M45 | v16 库缺 `zhuli_fact` 时升级失败且不建空表 | 抛异常 → 回滚 → `user_version=16`；`zhuli_fact` 仍不存在（**没有**被补建） | 手工造只含 `inbox` 的 v16 库 |

> `T-M40`（新装库结构）不在本表：它测 `onCreate`，归 `MemorySchemaTest`（§4.8.6 已说明）。


### 10.3 B 组｜存储策略（11 条，`MemoryStoreTest`）

| # | 用例 | 断言 |
|---|---|---|
| T-S11 | 同值两来源 | 1 item、2 evidence（1 PRIMARY + 1 SUPPORTING）、`times_reinforced=1`、`ingest` 返回 0（第二来源不新建） |
| T-S12 | rank 更高 | 旧 SUPERSEDED + `SUPERSEDES` + `invalidated_at=now` |
| T-S13 | 旧 USER_CONFIRMED，新 RULE_EXTRACTED | 旧保持 ACTIVE，新 CANDIDATE + `CONFLICTS_WITH` |
| T-S14 | MULTI 槽位 | 两条 ACTIVE，`superseded_by=0` |
| T-S15 | 同 rank × 间隔 3 天 | 不替代，新 CANDIDATE + 冲突（**且用世界时间算 Δt**，见 §2.4 注） |
| T-S16 | `correct()` | 新行继承 `valid_from` 与 `time_confidence`；旧行 `valid_until=新 valid_from`；`CORRECTS` |
| T-S17 | STATE 变更 | 旧 EXPIRED（`valid_until=now`），新 ACTIVE，`WORLD_ASOF` 无重叠 |
| T-S18 | EVENT 纠错 | 旧 `event_start/event_end` 一字不变，`SUPERSEDED`，`CORRECTS` |
| T-S19 | 删除后重新写入同值 | 部分唯一索引允许插入 |
| T-S20 | DELETED/REJECTED 同值再提取 | 抑制；用户显式陈述允许复活（`RESTORES`） |
| T-S39 | R0 时间自洽 | `valid_from=now+30d` + `NONE` → 清零**并强制 CANDIDATE**；同条件 `DAY` → 保留 |

### 10.4 C 组｜时间与 as-of（8 条，`MemoryTimeQueryTest`）

| # | 用例 | 断言 |
|---|---|---|
| T-T21 | `WORLD_ASOF` 历史点 | `at<X` 命中旧值、不命中新值（有置信度的数据） |
| T-T22 | `BELIEF_ASOF` | 失效前命中、失效后不命中 |
| T-T23 | `valid_from=0` | CURRENT 可见；WORLD 以 `observed_at` 为下界；`boundary_unknown=true` |
| T-T24 | 纠正后 `WORLD_ASOF` | 旧值不出现（`CORRECTS` 排除），`BELIEF_ASOF` 仍可见 |
| T-T40 | CURRENT 排除未来生效 | ① `DAY` + `now+1d` → 0 命中；`now` 推后 → 命中。② `NONE` + 未来 → 清零且 CANDIDATE → **0 命中** |
| T-T41 | 半开区间精确边界 | `at=X-1` 只旧、`at=X` 只新、`at=X+1` 只新；任何时刻不出现两条 |
| T-T42 | 未知边界 → ambiguous | 迁移行在 WORLD 下与后续值并存 → `ambiguous=true` 且每条 `boundary_unknown=true`；BELIEF 同一时刻只 1 条 |
| T-T43 | 任务到期不被起点置信度挡住 | `valid_from=0 & NONE` + `valid_until=finishedAt+7d`：到期前命中、到期后 0 |

### 10.5 D 组｜Scope（7 条，`MemoryScopeTest`）

| # | 用例 | 断言 | 造数据 |
|---|---|---|---|
| T-C25 | 项目 A / B | 互不命中 | `MemoryTestFixture.insertRaw(PROJECT,"pA")` |
| T-C26 | 助手 A / B（含切换） | 互不命中，`GLOBAL` 仍可见 | fixture |
| T-C27 | 普通聊天读项目私有 | 0 命中 | fixture |
| T-C28 | TASK 过期 | `valid_until` 过后 0 命中 | fixture + 显式 `valid_until` |
| T-C29 | GLOBAL 进入项目任务 | 命中 | fixture |
| T-C30 | 非法 scope（`PROJECT` + 空 id） | 写入抛异常；检索抛异常 | `MemoryStore.insert` |
| T-C57 | 生产写入对 PROJECT/CONVERSATION 关闭 | 两者均抛 `IllegalArgumentException` | 直接调 `MemoryStore.insert` |

**14.3 的约束必须遵守**：`T-C25`–`T-C29` 的数据**只经** `MemoryTestFixture.insertRaw` 构造；`T-C30`/`T-C57` 测的正是"生产 API 拒绝"，两者不矛盾——前者绕过写入层直接造行测检索层，后者验证写入层不放行。

### 10.6 E 组｜删除与恢复（9 条，C1 实现 8 条——`T-D32` 明列但归 C3，`MemoryStoreTest`）

> 本节列 9 条是为了让 C0.2 的编号链完整；**C1-B 实际编写 8 条**（`T-D31`/`T-D33`/`T-D44`–`T-D49`）。`T-D32` 只留设计（§17），不计入 §10.11 的 47 条架构用例，也不占测试预算。

| # | 用例 | 断言 |
|---|---|---|
| T-D31 | 软删 | 行在（DELETED）；`pre_delete_status` 正确；FTS/向量/队列/检索四路 0 |
| T-D32 | 硬删（store 级，C3 预演） | **C1 不实现**，只留设计（§17） |
| T-D33 | 共享来源 | 删 A 后 source 仍在（B 引用）；删 B 后 source 可 GC（C1 只断言"仍存在"，GC 在 C3） |
| T-D44 | 删 ACTIVE 再恢复 | `ACTIVE`、`deleted_at=0`、`invalidated_at=0`、`pre_delete_status=''`、向量重新入队、FTS 重命中 |
| T-D45 | 删 SUPERSEDED 再恢复 | `SUPERSEDED`（不是 ACTIVE）、`invalidated_at` 仍是替代时间、检索 0 |
| T-D46 | 删 CANDIDATE 再恢复 | `CANDIDATE`（不是 ACTIVE）、不进 CURRENT、在待确认列表 |
| T-D47 | 恢复时槽位被占 | 同槽不同值 ACTIVE → `Demoted(CANDIDATE)` + `CONFLICTS_WITH`；MULTI → `Restored(ACTIVE)` |
| T-D48 | 恢复时同值已占用 | `AlreadyPresent(id)`、不抛 UNIQUE、本行仍 DELETED、证据复制到占用行 |
| T-D49 | 删除/恢复的 revision | 两条（`DELETE`→`RESTORE`），`before_json`/`after_json` 成对，`actor='USER'` |

### 10.7 F 组｜Agent（2 条，**C3**）

`T-A34`（注入记忆块不改状态机轨迹）、`T-A35`（prompt injection 不改工具选择）——**C1 不实现**，因为 C1 不改 `AgentLoop`、不做记忆注入。

### 10.8 G 组｜FTS（7 条，**C1-A 已完成**）

`T-F50`–`T-F56` 在 C1-A 的 `MemoryFtsInstrumentedTest` / `MemoryFtsRobolectricTest` 中已全部通过（设备端 9/9）。C1-B **不重写**，只在 B5 回归时重跑一次确认没被后续改动破坏。

### 10.9 H 组｜兼容状态与建表回归（18 条，**D-6 / D-7 新增**）

这一组不属于 C0.2 的 57 条编号体系，编号独立：`T-X*` 是状态兼容，`T-M4x` 是建表回归。

#### 10.9.1 D-6｜状态兼容（12 条，`MemoryFactStatusTest`）

| # | 用例 | 断言 | 层 |
|---|---|---|---|
| T-X01 | 六个业务状态名解析 | `of("ACTIVE")`=ACTIVE、`of("UNCONFIRMED")`=UNCONFIRMED、`of("SUPERSEDED")`、`of("EXPIRED")`、`of("REJECTED")`、`of("DELETED")` 全部命中；`of("UNKNOWN")`=UNKNOWN | L1 |
| T-X02 | 未知值一律 UNKNOWN | `null` / `""` / `"active"` / `"CANDIDATE"` / `"garbage"` / `"EXPIRED "`（尾空格）→ 全部 UNKNOWN；**没有一个是 ACTIVE** | L1 |
| T-X03 | 序列化往返 | 7 个值 `name` → `of(name)` 回到原值；`ordinal` 变化不影响（禁止按序号存） | L1 |
| T-X04 | UNKNOWN 非当前、不可激活 | `UNKNOWN.isCurrent=false`、`isActionable=false`；`search`/`active`/`conflicts` 均不返回；`confirmFact` 对 UNKNOWN 行返回 false（`WHERE status='CANDIDATE'` 挡住） | L1+L2 |
| T-X05 | v16 旧四态兼容 | 迁移后 `ACTIVE`→ACTIVE、`UNCONFIRMED`→UNCONFIRMED、`SUPERSEDED`→SUPERSEDED、`DELETED`→DELETED；`activeCount` 与旧 `status='ACTIVE'` 行数一致 | L2 |
| T-X06 | 六态投影一一对应 | 新表 6 个字符串 → `FactStatus` 6 个不同值，两两不相等；`CANDIDATE`→`UNCONFIRMED`（不是 UNKNOWN） | L2 |
| T-X07 | 四态语义区分（可见性矩阵） | 按 §7.12.4 的表逐格断言 6 态 × 5 查询 = 30 格；`EXPIRED`/`REJECTED`/`DELETED` 三者在 `history` 里可见且互不相等 | L2 |
| T-X08 | 迁移遇未知状态值 | v16 库手工写 `status='garbage'` → 迁移后 `status='CANDIDATE'`、`unknown_status=1`、`payload_json` 含 `legacy_status`；**不落 ACTIVE** | L2 |
| T-X09 | ACTIVE + CANDIDATE 同槽冲突 | 1 ACTIVE + 1 CANDIDATE（值不同）→ `conflicts()` 返回 1 对；旧实现返回 0 对 | L1+L2 |
| T-X10 | 两条 CANDIDATE 同槽 | 0 ACTIVE + 2 CANDIDATE（值不同）→ 返回 1 对（相邻配对兜底） | L1 |
| T-X11 | MULTI 槽位不误报 | `cardinality='MULTI'` 的同槽两条 ACTIVE → `MemoryRepo.conflicts()` 返回空 | L2 |
| T-X12 | 非行动态不参与冲突 | `EXPIRED`/`REJECTED`/`DELETED`/`UNKNOWN` 各与 ACTIVE 同槽 → `conflicts()` 返回空 | L1+L2 |

> `T-X09` 是这一组的"回归锚点"：它断言的正是 v1 方案里 X-3 描述的那个 bug（旧实现 0 对）。C1-B4 完成后如果这条失败，说明 `MemoryRanker.conflicts` 的过滤没改对。

#### 10.9.2 D-7｜建表回归（6 条，`MemorySchemaTest` + `MemoryMigratorTest`）

已在 §4.8.6 与 §10.2 列出：`T-M40`（Schema）、`T-M41`–`T-M45`（Migrator）。此处不重复。

#### 10.9.3 两组用例的"双模式"跑法

`T-X04`–`T-X09`、`T-X11`、`T-X12`（**8 条**）走真实 SQLite，**必须在 `TRIGGER` 与 `APP_SYNC` 两种 `ftsMode` 下各跑一遍**（K-7 的要求）。`T-X01`–`T-X03`、`T-X10`（**4 条**）是纯函数，只跑一遍。

> v2 把 `T-X09` 归到"纯函数"里，导致 7 + 4 = 11 ≠ 12。`T-X09`（1 ACTIVE + 1 CANDIDATE → `conflicts()` 返回 1 对）既要用 `MemoryRanker` 纯函数，也要用 `MemoryStore.conflictFacts()` 的 SQL，所以是 **L2**，要双跑。

因此执行次数 = 8 × 2 + 4 × 1 = **20 次**。

**实现方式（v3 明确，避免口径争议）**：用**两个独立的测试类**继承同一抽象基类，而不是 JUnit `Parameterized`：

```kotlin
abstract class MemoryFactStatusTestBase {
    abstract val ftsMode: FtsMode
    @Test fun `T-X04 UNKNOWN 非当前不可激活`() { … }
    // … 其余 7 条 L2 用例
}
class MemoryFactStatusTriggerTest : MemoryFactStatusTestBase() { override val ftsMode = FtsMode.TRIGGER }
class MemoryFactStatusAppSyncTest : MemoryFactStatusTestBase() { override val ftsMode = FtsMode.APP_SYNC }
```

为什么不用 `@RunWith(Parameterized::class)`：
- 参数化会把类名变成 `MemoryFactStatusTest$0` / `$1`，报告里看不出是哪一档，出问题要翻 XML 才能定位；
- 两个具名类的失败信息直接写 `MemoryFactStatusAppSyncTest.T-X04 失败`，一眼知道是 `APP_SYNC` 档；
- Gradle 的 `tests` 计数在两种方式下**都按实际执行数算**（每参数组 + 每个 `@Test` 各计 1），所以"双跑 = 2 倍"这个结论不受实现方式影响。

> **v2 的口径错误**：v2 写"`tests` 计数按 `@Test` 方法算，不按参数化展开算"。这句是错的——Gradle 的 `testsuite tests` 属性统计的是**执行过的测试用例数**，参数化展开的每一组都计入。正因为这句错误，才会得出 396 这个数（把 65 条逻辑用例当成 65 次执行）。

### 10.10 设备验收 D1–D10

| # | 项 | 通过标准 | 证据 |
|---|---|---|---|
| D1 | 覆盖升级 v16→v17 | 首启不崩；记忆页显示迁移后的条目 | `adb install -r` + 截图 + 迁移日志 |
| D2 | 迁移耗时 | 5k ≤3s；记录实测 | `MemoryMigrationTimingTest` 的 `Log.i` |
| D3 | FTS4 建表 | `sqlite_master` 有 `zhuli_memory_fts` + 3 触发器（或 `APP_SYNC` 时无触发器但检索正确） | `adb shell` 查询 |
| D4 | 设备检索 | 搜"验收"命中迁移来的两条 | 手工 + 截图 |
| D5 | 本地模式 | 飞行模式完整可用，无网络调用 | 飞行模式手测 + `cloud_log` 无新行 |
| D6 | 严格模式/隐私 | `save_fact` 传手机号 → SENSITIVE；传 16 位卡号 → 拒绝 | 日志 + 库查询 |
| D7 | StrictMode | 无主线程磁盘读写告警 | logcat |
| D8 | B1 回归 | 12 个场景全过 | B1 的验收脚本 |
| D9 | 时间语义端到端 | 不显示伪造的"何时开始"；CURRENT 不返回未来生效项 | 截图 + 库查询 |
| D10 | 恢复端到端 | 删 SUPERSEDED → 恢复 → 没变成当前值 | 截图 |

### 10.11 数量核算（P1-6，v3 重算）

#### 10.11.1 逻辑用例数

| 组 | 全口径条数 | C1 实现 | 已在 C1-A | C1-B 新增 | v3 新增 |
|---|---|---|---|---|---|
| A 迁移 | 13 | 13 | 0 | **13** | 0 |
| B 存储 | 11 | 11 | 0 | **11** | 0 |
| C 时间 | 8 | 8 | 0 | **8** | 0 |
| D Scope | 7 | 7 | 0 | **7** | 0 |
| E 删除恢复 | 9 | 8（`T-D32` 归 C3） | 0 | **8** | 0 |
| F Agent | 2 | 0（归 C3） | 0 | **0** | 0 |
| G FTS | 7 | 7 | **7** | 0 | 0 |
| **架构小计** | **57** | **54** | **7** | **47** | 0 |
| H-1 D-6 状态兼容 | 12 | 12 | 0 | **12** | 0 |
| H-2 D-7 建表回归 | 6 | 6 | 0 | **6** | 0 |
| H-3 v3 补充 | 10 | 10 | 0 | **10** | **10** |
| **决策小计** | **28** | **28** | **0** | **28** | **10** |
| **合计** | **85** | **82** | **7** | **75** | **10** |

H-3 的 10 条：

| 编号 | 用例 | 归属文件 | 对应本轮的哪一项 |
|---|---|---|---|
| `T-M46` | 迁移后自动 ID 不冲突 | `MemoryMigratorTest` | P1-8（`sqlite_sequence`） |
| `T-S21` | `TRIGGER` 档不双写 | `MemoryStoreTest` | P0-5（W-1） |
| `T-S22` | `APP_SYNC` 档 7 条路径都不漏写 | `MemoryStoreTest` | P0-5（W-2） |
| `T-S23` | `NONE` 档 `search_text` 仍写、LIKE 命中且四约束不丢 | `MemoryStoreTest` | P0-5（W-3）+ §6.3.1 |
| `T-S24` | 同一 source 重复 → `Duplicate` 零副作用 | `MemoryStoreTest` | P1-5 |
| `T-N01` | 并发 `correct` | `MemoryStoreTest` | 验收细节 1 |
| `T-N02` | 并发 `confirm` | `MemoryStoreTest` | 验收细节 1 |
| `T-N03` | 并发 `resolveConflict` | `MemoryStoreTest` | 验收细节 1 |
| `T-N04` | 并发 `softDelete` | `MemoryStoreTest` | 验收细节 1 |
| `T-N05` | 并发 `restore` | `MemoryStoreTest` | 验收细节 1 |

#### 10.11.2 执行次数（XML 口径）

**判据只有一条**：这条用例是否会调用 `MemoryStore` 的写方法（`insert` / `correct` / `confirm` / `resolveConflict` / `expire` / `softDelete` / `restore`）。会 → 双跑（因为 `APP_SYNC` 档在这 7 个方法里各多一次 `syncFts`）；不会 → 单跑。

| 组 | 逻辑 | 关联 `ftsMode` | 执行 | 理由 |
|---|---|---|---|---|
| A 迁移 13 | 13 | ❌ | 13 | 迁移"先灌 FTS 后建触发器"，与运行时 `ftsMode` 无关（§5.2 第 7–9 步） |
| B 存储 11 | 11 | ✅ | 22 | 全部走 `insert` / `correct` |
| C 时间 8 | 8 | ✅ | 16 | 用 `insert` 造数据 |
| D Scope 7 | 7 | ✅ | 14 | 用 `insert` 造数据 |
| E 删除恢复 8 | 8 | ✅ | 16 | 走 `softDelete` / `restore` |
| D-6 的 4 条 L1（`T-X01`–`T-X03`、`T-X10`） | 4 | ❌ | 4 | 纯函数 / 枚举解析 |
| D-6 的 8 条 L2（`T-X04`–`T-X09`、`T-X11`、`T-X12`） | 8 | ✅ | 16 | 要读真实 FTS 与 `conflicts()` |
| D-7 的 6 条 | 6 | ❌ | 6 | `onCreate` / `onUpgrade` / `onOpen` 的结构断言 |
| `T-M46` | 1 | ❌ | 1 | 迁移后插一行，不涉及 FTS 检索 |
| `T-S21`–`T-S24` | 4 | ✅ | 8 | 本来就是"两档对比"，双跑是设计本身 |
| `T-N01`–`T-N05` | 5 | ✅ | 10 | 并发写，`APP_SYNC` 下多一次 `syncFts` |
| **合计** | **75** | — | **126** | 单跑 24 + 双跑 51×2 |

```
C1-B 结束时预期 XML 计数 = 331（本轮实测基线）+ 126 = 457
逻辑新增 75 条，实际执行 126 次，差值 51 是双跑的第二遍
```

#### 10.11.3 三个数字的关系（防止后面看混）

| 数字 | 含义 | 谁在用 |
|---|---|---|
| `57` | C0.2 §11.2 定义的**架构用例**总数 | C0.2 的验收口径，不变 |
| `85` | C1-B 的**全口径逻辑用例数**（57 + 28） | 本文 §15 的阶段验收 |
| `75` | C1-B **新增**逻辑用例（47 架构 + 28 决策） | 逻辑口径的加数 |
| `126` | C1-B **新增执行次数**（含双跑） | XML 口径的加数 |
| `457` | **XML 目标**（331 + 126） | §16 退出条件 #2 |

> **v2 的 396 错在哪**：它等于 `331 + 65`，其中 65 是逻辑数、331 是 XML 数。加法把两种口径混在一起，导致"双跑"在门禁里不可见。v3 的 457 与 75 各自内部自洽，且 457 可以用 `grep -c "@Test"` 与实际执行报告交叉验证。
>
> **如果将来决定不双跑全部 51 条**（例如只双跑 B/E 两组），`126` 会下降，但**必须同时改 §10.11.2 的表**，不能只改总数——否则就又回到"总数与明细对不上"的老问题。

### 10.12 v3 新增用例（10 条）

这 10 条不是"补充覆盖"，每一条都对应本轮修掉的一个缺陷；**去掉任何一条，那个缺陷就会重新变成"文档里写了、没人验"**。

#### 10.12.1 `T-M46`｜迁移后自动 ID 不冲突（P1-8）

| 项 | 内容 |
|---|---|
| 前置 | `seedLegacy(11)` 造 v16 库，`id` 含大值（如 1001–1011） |
| 动作 | 迁移 → `INSERT INTO zhuli_memory_item(subject, predicate, value, …)` **不指定 id** |
| 断言 | ① 新 id == `MAX(迁移 id) + 1`；② `sqlite_sequence.seq >= MAX(id)`；③ 再插一行，id 继续 +1；④ 两次插入不抛 `UNIQUE constraint failed: zhuli_memory_item.id` |
| 反例 | 若迁移用 `UPDATE sqlite_sequence` 且无行可更，新 id 会从 1 开始 → 撞已迁移的 id=1 → 断言 ④ 失败 |

#### 10.12.2 `T-S21`–`T-S24`｜FTS 写入职责（P0-5）

| 编号 | 用例 | 前置 | 断言 |
|---|---|---|---|
| `T-S21` | `TRIGGER` 档不双写 | `ftsModeOverride=TRIGGER`，插入 1 条 | FTS 行数 == 1（不是 2）；不抛约束；`syncFts` 调用计数 == 0（用可注入的计数钩子或 `grep` 保证代码路径） |
| `T-S22` | `APP_SYNC` 档 7 条路径都不漏写 | `ftsModeOverride=APP_SYNC`；依次执行 insert / correct / confirm / resolveConflict / expire / softDelete / restore | 每一步后 FTS 行数 == `COUNT(item WHERE status<>'DELETED')`；任一步不等即失败 |
| `T-S23` | `NONE` 档 `search_text` 仍写 | `ftsModeOverride=NONE`，插入含中文的值 | `search_text` 非空；`search("关键词")` 能命中；**同时**断言 scope / sensitivity / status / 生效区间四个约束仍在（造一条越界数据，确认不被捞出） |
| `T-S24` | 同一 source 重复 → `Duplicate` | 同一 `SourceRef`（同 `refId` + 同 `contentHash`）连续 `insert` 两次同值 | 第一次 `Created`，第二次 `Duplicate`；item 行数 1；evidence 行数 1；`seen_count=2`；**revision 行数仍为 1**（`Duplicate` 不写 revision） |

> `T-S22` 是唯一一条"一条用例覆盖 7 个方法"的用例。它不是偷懒：7 条路径的断言完全相同（FTS 行数 == 未删行数），拆成 7 条会让 §10.11.2 的双跑数从 8 涨到 14，而覆盖的缺陷是同一个（漏调 `syncFts`）。如果将来某条路径出现独立缺陷，再拆。

#### 10.12.3 `T-N01`–`T-N05`｜并发（验收细节 1）

5 条用例共用一个夹具：`CountDownLatch` 让两个线程在同一点同时开始，`ExecutorService(2)`，每线程 `db.writableDatabase` 各开事务。

| 编号 | 场景 | 两线程动作 | 期望 |
|---|---|---|---|
| `T-N01` | 并发 `correct` | T1 与 T2 同时对同一 `ACTIVE` 行 `correct` 成不同值 | 恰好 1 个成功（返回新 id）；另 1 个抛 `IllegalStateException`（乐观锁 `WHERE status='ACTIVE'` 失败）或 `changes()==0`；最终该槽位只有 1 条 `ACTIVE` + 1 条 `SUPERSEDED`（不是 2 条 ACTIVE） |
| `T-N02` | 并发 `confirm` | T1 与 T2 同时对同一 `CANDIDATE` `confirm` | 1 个返回 `true`，1 个返回 `false`；`ACTIVE` 行数 1；`revision` 里 `CONFIRM` 只有 1 条 |
| `T-N03` | 并发 `resolveConflict` | T1 `resolveConflict(a,b)`，T2 `resolveConflict(b,a)` | 只有 1 个成功；`SUPERSEDED` 行数 1；不存在"两条都 SUPERSEDED"（那会让槽位没有当前值） |
| `T-N04` | 并发 `softDelete` | T1 与 T2 同时 `softDelete(id)` | 1 个返回 `Deleted`，1 个返回 `NotDeleted`；`pre_delete_status` 是**同一个值**（不被第二次覆盖）；`deleted_at` 只写一次 |
| `T-N05` | 并发 `restore` | T1 `softDelete` 后立刻 `restore`，T2 同时 `restore` | 1 个成功；另 1 个返回 `NotDeleted` 或 `AlreadyPresent`；**不抛 UNIQUE**；`pre_delete_status` 被清空一次 |

**断言"不抛 UNIQUE"是重点**：并发下若两个线程都通过"查 `dedupe_key` 没有占用"的检查，然后都去 UPDATE，第二个会撞 `idx_mi_dedupe`。`T-N05` 就是盯这个窗口——正确实现要么用 `INSERT OR IGNORE` + 重查，要么把检查与更新放在同一事务里（本方案是后者，SQLite 的写锁保证串行）。

**这 5 条不追求"可复现的随机失败"**：用 `CountDownLatch` 把窗口缩到最小，跑 50 次；只要 1 次出现"两条 ACTIVE"或"未捕获的 `SQLiteConstraintException`"就失败。50 次 × 5 条在 Robolectric 上约 2–3 秒，可接受。


---

## 11. 数据迁移预演计划

### 11.1 四档规模（沿用 C0.2 §9 的库，不新造）

| 档 | 库 | 行数 | 用途 | 判定 |
|---|---|---|---|---|
| S0 | `Temp/c02/zhuli_inbox_v16_real.db` | 2 | 真实形态（1 ACTIVE + 1 SUPERSEDED + 1 向量） | 必须全绿 |
| S1 | `Temp/c02/v16_edge.db` | 11 | 边界样本（空值、断链、SECRET、重复、MULTI） | 必须全绿 |
| S2 | `Temp/c02/v16_5k.db` | 5,002 | 中等规模 + 耗时长 | 必须全绿 |
| S3 | `Temp/c02/v16_20k.db` | 20,002 | 大规模 + 体积 | 必须全绿 |

**C1-B2 的做法**：不直接改这 4 个文件（C0.2 的预演产物要保留原样），而是 `cp` 到 `Temp/c1b/` 再跑。产物命名 `v17_b1_{s0,s1,s2,s3}.db`。

### 11.2 边界样本清单（S1 必须覆盖）

| # | 形态 | 期望 |
|---|---|---|
| 1 | 空 `subject` | `skipped` |
| 2 | 空 `value` | `skipped` |
| 3 | 16 位卡号 | `quarantined`，不进新表 |
| 4 | `superseded_by=999999`（断链） | 清 0，`invalidated_at` 保留 |
| 5 | 两条同值不同来源 | 合并，保留强者 |
| 6 | 两条取件码 | 不合并，都 ACTIVE（MULTI） |
| 7 | `valid_from != observed_at`（人为） | `vf_mismatch+1`，不中止 |
| 8 | `status='UNCONFIRMED'` | → `CANDIDATE` |
| 9 | `status='DELETED'` | `deleted_at=updated_at`，`pre_delete_status=''` |

### 11.3 故障注入（5 种，全部要回滚到 v16）

| # | 注入方式 | 期望 |
|---|---|---|
| F1 | 人为造重复 + 跳过合并 → 建唯一索引失败 | 抛 → 回滚 → `user_version=16`，`zhuli_fact` 仍叫原名 |
| F2 | 单行插入失败（用一条超长 `value` 触发约束或人为 mock） | `rowErrors+1` 继续 → 数量不闭合 → 回滚 |
| F3 | 建 FTS 前抛异常（`DROP` 掉临时能力） | 回滚（FTS 阶段不在"可降级"范围，只有"建不出来"才降级） |
| F4 | 改名失败（人为先建一张 `zhuli_fact_v1`） | 抛 → 回滚 → 数据仍在事务里撤销 |
| F5 | 迁移中途 `kill -9`（设备上 `am force-stop`） | 下次打开重跑；无半迁移状态（SQLite 回滚日志） |

> F5 无法在 Robolectric 里模拟，只能在设备上做（`MemoryMigrationTimingTest` 的 `@Test` 里不能真杀进程，所以 F5 走**手工步骤**，写进 C1-B5 报告的"手工验证"小节）。

### 11.4 不变量校验（迁移末尾 + 预演脚本各跑一遍）

```
① 数量闭合（6 终态账本，见 §5.9）：
   src == inserted + quarantined + skippedNoSubject + skippedNoValue + failed
   且 merged <= inserted（merged 是 inserted 的子集，不单独加）
   且 failed == 0（>0 时迁移已回滚，走不到这一步）
② C0.2 §6.6 的 10 组（11 条 SQL）全部返回 0 行
③ 5 条时间断言：1/2/4/5 = 0；3 允许 >0 但必须记入 vf_mismatch
④ PRAGMA integrity_check = ok
⑤ PRAGMA foreign_key_check = 空（虽然没建外键，仍跑一次防将来加）
⑥ sqlite_sequence.seq == MAX(id)（9 张 AUTOINCREMENT 表逐张；表不存在则跳过，见 §5.7）
⑦ zhuli_fact 不存在 / zhuli_fact_v1 存在
⑧ FTS 行数 == COUNT(item WHERE status<>'DELETED')（TRIGGER 与 APP_SYNC 两档都要满足）
```

### 11.5 设备耗时测试

见 §5.13。要点重复一次：**桌面 Python 的 0.17–0.21 ms/行不能当设备数据**，必须 `MemoryMigrationTimingTest` 在模拟器/真机上跑 1k/5k/20k 三档，记录 `ms` 与 `pss_kb`。

### 11.6 产物与路径

| 产物 | 路径 | 入库 |
|---|---|---|
| 迁移后的 4 个库 | `Temp/c1b/v17_b1_s{0..3}.db` | **否** |
| 耗时日志 | `Temp/c1b/timing_*.txt` | 否 |
| 故障注入日志 | `Temp/c1b/fault_*.txt` | 否 |
| 汇总结论 | C1-B5 报告的一节 | 是（报告里） |

`Temp/c02/` 与 `Temp/c1a/` **一律不动**（本轮明令禁止清理）。

---

## 12. 回滚方案

### 12.1 三个层次

| 层 | 触发 | 动作 | 数据影响 |
|---|---|---|---|
| **L1 事务回滚** | 迁移中任何未预期异常 | `onUpgrade` 的 `finally` 自动 `endTransaction()` 未提交 → SQLite 回滚 | 库仍是 v16，`zhuli_fact` 原样 |
| **L2 备份恢复** | L1 之后仍打不开（极端：回滚日志损坏） | 用 `databases/pre_migration/v16_*.db` 覆盖主库 | 回到升级前 |
| **L3 APK 降级** | 已升级到 v17，想退回旧版本 App | **不支持**（`onDowngrade` 未实现，`SQLiteOpenHelper` 抛异常） | 需要卸载重装（数据丢失） |

### 12.2 迁移失败的完整路径

```
migrate() 抛异常
  └─ onUpgrade catch → lastMigrationError = "v16→v17：<原因>"
      └─ Log.e（message 已 redact）
          └─ rethrow
              └─ SQLiteOpenHelper: T0 未 setSuccessful → 回滚
                  └─ 抛给 getWritableDatabase 调用方
                      └─ 现有机制：App 显示"数据库打不开"的降级页（B 批次已实现）
```

**关键**：`lastMigrationError` 是**静态字段**（`InboxDatabase.lastMigrationError`，已存在），下次启动时自检页能读到——用户不需要连电脑就能看到"上次升级失败，原因 X"。

### 12.3 为什么不做 `onDowngrade`

| 理由 | 说明 |
|---|---|
| 数据不可逆 | v17 的新表在降级后会变成"未知表"，`zhuli_fact` 已被改名，旧代码会 `no such table` 直接崩 |
| 迁移是可选的单行道 | 一旦写入新表，回退意味着丢新数据 |
| 替代方案 | 卸载重装（用户明确知道会清空）；或从 L2 备份恢复（需要 root/adb） |

C1-B 在 `onDowngrade` 里**不做任何事**（保持 `SQLiteOpenHelper` 默认抛异常），但把异常消息改成可读的中文提示 `[实现补充]`。这条改动在 `InboxDatabase.kt` 内，不新增文件。

### 12.4 回滚演练（C1-B5 必做）

| # | 演练 | 判据 |
|---|---|---|
| R1 | 用 §11.3 的 F1 注入，确认回滚到 v16 | `user_version=16`，`zhuli_fact` 有全部行 |
| R2 | 用 L2 备份覆盖后打开 | App 正常启动，记忆页显示 v16 的数据（旧代码路径） |
| R3 | 迁移成功后主动 `DROP` 新表再打开 | `onOpen` 的 `ensureAll` 能把表补回来（数据不回来，但结构自愈） |

---

## 13. 风险清单（15 项）

### 13.1 总表

| # | 风险 | 等级 | 主要防线 | 验收证据 |
|---|---|---|---|---|
| K-1 | 迁移时旧数据被错误解释为当前事实 | **高** | `time_confidence='NONE'` + `LB` 退回 `observed_at` | `T-M36` / D9 |
| K-2 | 迁移后未来生效事实提前生效 | **高** | R0 时间自洽（清零 + 强制 CANDIDATE） | `T-S39` / `T-T40` |
| K-3 | 多来源共享导致误删 | 中 | C1 无硬删；软删保留 evidence/source | `T-D33` |
| K-4 | SUPERSEDED 恢复后重新成为当前值 | **高** | `pre_delete_status` + 恢复状态表 | `T-D45` / D10 |
| K-5 | CANDIDATE 状态与唯一索引冲突 | **高** | 恢复第 2 步同时查 ACTIVE **和** CANDIDATE | `T-D48` |
| K-6 | 软删/恢复与 FTS 索引不一致 | 中 | 触发器 + `APP_SYNC` + `selfCheck` 第 1/2 条 | `T-D31` / `T-D44` |
| K-7 | TRIGGER 与 APP_SYNC 两条路径行为不一致 | **高** | 同一套用例双跑（`ftsModeOverride`） | §10 的全部 L2 用例 ×2 |
| K-8 | SQLite 版本差异导致迁移失败 | 中 | 只用 3.22 起可用的语法；两版本实测 | `MemorySchemaTest`（3.32）+ 设备（3.42） |
| K-9 | SECRET 在备份/临时文件/日志中残留 | **高** | 迁移不迁入 + 日志无原文 + 备份清理在 C3 明确 | `T-M04` + 日志 grep + 14.2 阶段表 |
| K-10 | Scope 过滤遗漏导致跨项目读取 | **高** | `scopeClause()` 单点 + 空列表抛异常 | `T-C25`–`T-C30` / `T-C57` |
| K-11 | 旧 `Fact` 字段变更导致下游崩溃 | 中 | 27 个签名不变 + 投影显式 `when` | 现有 15 例全绿 + grep |
| K-12 | 迁移与 Agent 正在执行的任务冲突 | 中 | 单事务 + 不改 `zhuli_agent_task` + 耗时上界 | D2 / D8 |
| K-13 | `FactStatus` 扩展后，未知状态被当作当前值 | **高** | `of()` 返回 `UNKNOWN` + 投影显式七分支 + `isCurrent`/`isActionable` | `T-X01`–`T-X07` |
| K-14 | 建表拆分不彻底：旧表复活或核心表缺失 | **高** | 三入口职责表（§4.7）+ `createCoreTablesV15` 保留全部非 fact 表 | `T-M40`–`T-M45` |
| K-15 | FTS 缓存说 `TRIGGER`、实际触发器已丢，导致索引**静默**不同步 | **高**（v3 新增） | 缓存六道门（§6.4 第 5/6 项）+ `ensureAll` 补建后清缓存 + `selfCheck` 第 1/2 条 | `T-S21` + §9.3 的缓存失效五场景 |

### 13.2 逐项展开

#### K-1｜旧数据被错误解释为当前事实

| 项 | 内容 |
|---|---|
| **发生条件** | 迁移时按"`valid_from>0` 就说明知道起点"处理，或把 `valid_to` 当世界时间终点 |
| **影响范围** | 所有 v16 存量行（20k 规模下 100%）；记忆页显示"2026-11-25 起使用的手机"，而用户 2024 年就换了 |
| **代码位置** | `MemoryMigrator.migrateRow()` 的 `time_confidence` 与 `valid_until` 两列 |
| **预防** | ① `time_confidence = "NONE"` 写成常量，**不放在 `if` 里**；② `valid_until = 0` 写成常量；③ 旧 `valid_to` 只进 `invalidated_at`；④ 查询端只认 `LB`（`MemoryText.LB_SQL` 唯一来源） |
| **验收** | `T-M36`（每一行 `NONE`，含 `valid_from>0` 的行）；`T-M37`（`valid_to` 的去向）；D9（记忆页不显示伪造的"何时开始"）；迁移日志 `vf_mismatch` |

#### K-2｜未来生效事实提前生效

| 项 | 内容 |
|---|---|
| **发生条件** | ① `CURRENT` 只看 `valid_until`；② R0 清零 `valid_from` 但没改状态（清零后 `LB` 退回 `observed_at`，立刻变成当前值） |
| **影响范围** | 任何 `valid_from > now` 的行；C1 里只有"用户说'明年起'"这一条路径，但 C2 的抽取会大量产生 |
| **代码位置** | `MemoryStore.current()` 的谓词；`MemoryPolicy` 的 R0 |
| **预防** | ① `CURRENT` 含 `LB <= now`；② R0 清零**并强制 `CANDIDATE`**（C0.2 §4.5 的注）；③ `WB` 不设 `time_confidence` 门槛（否则任务永不过期） |
| **验收** | `T-T40` 的两段（`DAY` 保留 + 排除；`NONE` 清零 + CANDIDATE + 0 命中）；`T-T43`（任务到期不被起点置信度挡住） |

#### K-3｜多来源共享导致误删

| 项 | 内容 |
|---|---|
| **发生条件** | 删除记忆 A 时连带删掉 source，而记忆 B 的 evidence 还引用它 → B 的"凭什么"变空 |
| **影响范围** | `zhuli_memory_source` 与 `evidence`；C1 只有软删，风险有限，但 C3 接硬删后会变高 |
| **代码位置** | `MemoryStore.softDelete()`（**不删** source/evidence）；C3 的 `purgeSources()`（不在 C1） |
| **预防** | ① C1 软删只删 FTS/向量/队列，**保留** evidence 与 source；② 硬删的 GC 用 `NOT EXISTS` 引用计数（C0.2 §5.5），不用计数器；③ `selfCheck` 第 6 条报告无引用来源，但**不自动删** |
| **验收** | `T-D33`（删 A 后 source 仍在） |

#### K-4｜SUPERSEDED 恢复后重新成为当前值

| 项 | 内容 |
|---|---|
| **发生条件** | 恢复一律写 `status='ACTIVE'`（C0.1 的写法） |
| **影响范围** | 一个槽位两个 ACTIVE → `CURRENT` 返回两个矛盾值 → 模型静默出错；这是 C0.2 缺陷 #3 |
| **代码位置** | `MemoryStore.restore()` 第 1 步（`target = pre_delete_status`） |
| **预防** | ① `pre_delete_status` 仅首次记录；② `target` 按状态表（C0.2 §6.9）；③ `''` → `CANDIDATE`（不猜）；④ 第 3 步查槽位，占用则 `Demoted` |
| **验收** | `T-D45`（`SUPERSEDED` 恢复后仍是 `SUPERSEDED`，检索 0 命中）；`T-D46`（`CANDIDATE` 不进 CURRENT）；D10 |

#### K-5｜CANDIDATE 状态与唯一索引冲突

| 项 | 内容 |
|---|---|
| **发生条件** | 以为"降级成 CANDIDATE 就不会撞唯一索引"，于是恢复时直接 UPDATE |
| **影响范围** | `idx_mi_dedupe` 是 `WHERE status IN ('ACTIVE','CANDIDATE')` 的**部分**唯一索引，两态都占索引 → `SQLiteConstraintException` |
| **代码位置** | `MemoryStore.restore()` 第 2 步的查询条件 |
| **预防** | ① 第 2 步查询**同时**含 `ACTIVE` 与 `CANDIDATE`；② 命中时返回 `AlreadyPresent` 并把 evidence 复制给占用行（不丢"这条值被说过"）；③ **不**用 `INSERT OR REPLACE`（会静默覆盖占用行） |
| **验收** | `T-D48`（`AlreadyPresent`，不抛 UNIQUE，本行仍 DELETED，证据已复制）；C0.2 §6.9 的探针结果作为设计依据 |

#### K-6｜软删/恢复与 FTS 索引不一致

| 项 | 内容 |
|---|---|
| **发生条件** | ① `ftsMode=APP_SYNC` 时漏调 `syncFts`；② 软删后索引行没删（`au` 触发器没覆盖 `status`）；③ 恢复后没重挂 |
| **影响范围** | 检索"看起来正常但漏结果"或"删了还能搜到"——后者是隐私问题 |
| **代码位置** | `MemoryStore` 的 4 个写路径（insert/update/softDelete/restore）；`MemorySchema` 的触发器 DDL |
| **预防** | ① 触发器用 `AFTER UPDATE OF search_text, status`（含 status）；② `APP_SYNC` 下 4 处显式 `syncFts`，**集中在 `MemoryStore`**，不散落；③ `selfCheck` 第 1/2 条兜底；④ `rebuildFts()` 提供修复入口 |
| **验收** | `T-D31`（四路 0 命中）；`T-D44`（恢复后重命中）；`T-F54`/`T-F55`（C1-A 已验）；`selfCheck` 在人为破坏后能报出 |

#### K-7｜TRIGGER 与 APP_SYNC 两条路径行为不一致

| 项 | 内容 |
|---|---|
| **发生条件** | 只在 `TRIGGER` 下跑测试（C1-A 的结论是 `TRIGGER`，很容易忘了另一条路） |
| **影响范围** | 换一台 ROM 落到 `APP_SYNC` 时行为不同 → 用户侧"有的机器能搜到、有的搜不到" |
| **代码位置** | `MemoryStore` 的构造参数 `ftsModeOverride`；`MemorySchema.detectFtsMode` |
| **预防** | ① `MemoryStore(db, ftsModeOverride: FtsMode? = null)`——测试传 `APP_SYNC`，生产不传；② **全部 L2 用例跑两遍**（TRIGGER / APP_SYNC）；③ `NONE` 单独一组 3 条（检索降级但权限不降） |
| **验收** | 双跑的 51 条（§10.11.2 名单）× 2 模式全绿 = 102 次执行；单跑 24 条各 1 次；合计 126 次。v2 写"47 条 × 2"是错的：A 组迁移 13 条与 D-7 的 6 条不碰 `ftsMode`，只跑一遍；`grep "ftsModeOverride" app/src/main` 只命中构造参数与 `if` 分支（无生产调用方传值） |

> `[实现补充]` 这是 C0.2 没写、但必须有的测试夹层。没有它，"两种模式行为一致"只是一句口号。

#### K-8｜SQLite 版本差异导致迁移失败

| 项 | 内容 |
|---|---|
| **发生条件** | 用了高于 minSdk 对应 SQLite 版本的语法；或 ROM 裁掉了 FTS4 |
| **影响范围** | 迁移整体回滚（不会半迁移），但用户升级失败 |
| **代码位置** | `MemorySchema` 的全部 DDL；`MemoryMigrator` 的 13 步 |
| **预防** | ① 只用 SQLite **3.22**（API 28）起可用的语法：`CREATE TABLE` / 部分索引（3.8+）/ `ALTER TABLE RENAME`（3.1+）/ FTS4（3.x）/ `DROP TABLE IF EXISTS`；**不用** `UPSERT`（3.24+）、`RETURNING`（3.35+）、窗口函数（3.25+）；② FTS 建不出不中止（降 `NONE`）；③ 迁移前断言能建临时表 |
| **验收** | `MemorySchemaTest` 在 Robolectric `NATIVE`（3.32.2）跑一遍；设备（3.42.0）跑一遍；**API 28（3.22）未验证**，列进 §19 的遗留风险 |

#### K-9｜SECRET 在备份/临时文件/日志中残留

| 项 | 内容 |
|---|---|
| **发生条件** | ① 迁移把 SECRET 降级成 SENSITIVE 迁入；② 日志打了 `subject`/`value`；③ 备份目录 `databases/pre_migration/v16_*.db` 里还有一份完整旧库；④ 开发机的 `Temp/` 副本 |
| **影响范围** | 用户设备上的明文敏感数据；`adb backup` / 云备份 / 换机迁移会带走 |
| **代码位置** | `MemoryMigrator` 的 `quarantined` 分支与日志语句；`backupBeforeMigration`（既有） |
| **预防** | ① `MemoryGuard.classify(content)==SECRET` → 跳过 + 计数，**禁止降级**；② 日志只写计数（§5.10 的字段清单），`message` 过 `redact`；③ `selfCheck` 报 `zhuli_fact_v1` 剩余行数（只报数）；④ C0.2 §14.2 的阶段 3 把"清理备份"列为验收条件（C3 执行，但 C1 要**告知**） |
| **验收** | `T-M04`（不迁入 + 计数闭合）；`grep -n "quarantined" MemoryMigrator.kt` 的语句里无 `content`/`subject`/`value`；C1-B5 报告里明确写"迁移后有 N 条敏感旧数据留在 `zhuli_fact_v1`，将在 C3 清理时一并删除" |

#### K-10｜Scope 过滤遗漏导致跨项目读取

| 项 | 内容 |
|---|---|
| **发生条件** | ① 某个查询忘了带 scope 子句；② 用 `scope_type != 'PROJECT'` 这种白名单倒推（会把 `CONVERSATION` 放进来）；③ `scopes` 为空时"返回全部" |
| **影响范围** | 项目 A 的记忆出现在项目 B；这是隐私事故，不是功能 bug |
| **代码位置** | `MemoryStore.scopeClause()`（唯一拼接点）；`MemoryQuery.init` |
| **预防** | ① 所有查询走同一个 `scopeClause()`，**没有第二个入口**；② 空列表抛 `IllegalArgumentException`（不是返回空）；③ 显式 OR 列表，永不倒推；④ `LIKE` 降级时同样带这四个约束 |
| **验收** | `T-C25` / `T-C26` / `T-C27` / `T-C30`；`grep -c "scope_type" MemoryStore.kt` 只出现在 `scopeClause` 与 `selfCheck` |

#### K-11｜旧 `Fact` 字段变更导致下游崩溃

| 项 | 内容 |
|---|---|
| **发生条件** | 重写 `MemoryRepo` 时顺手把 `Fact.validTo` 删了，或把 `status` 换成新枚举 |
| **影响范围** | `ZhuliMemoryPage` / `ZhuliAgentTools` / `MemoryRanker` / `MemoryBackfill` 编译失败或运行时 `NoSuchField` |
| **代码位置** | `MemoryRepo` 的投影函数；`MemoryTypes.kt`（**D-6 授权修改**：只加 3 个枚举值与 `of()` 的兜底，不动 `Fact` 的 16 个字段） |
| **预防** | ① `Fact` 的 16 个字段与 `FactKind` / `SourceType` / `EntityKind` / `FactDraft` / `FactHit` 全部不动（§7.12.2 的"不做的改动"）；② 27 个签名逐条对照（§8.5）；③ 投影用显式 `when(status)`，**禁止 `FactStatus.of(raw)`**（`CANDIDATE` 必须映射成 `UNCONFIRMED`，`of()` 做不到）；④ `validFrom`/`validTo` 保留字段（值可能恒 0） |
| **验收** | 现有 `MemorySqliteTest` 15 例只改 1 处版本断言即全绿；grep 查 `.validFrom` / `.validTo` 仍只命中写入点；编译通过 |

#### K-12｜迁移与 Agent 正在执行的任务冲突

| 项 | 内容 |
|---|---|
| **发生条件** | ① 迁移在后台 Worker 里触发，主线程同时在跑 Agent 任务，写 `zhuli_agent_task` 被写锁挡住；② 通知回调（~10s 预算）撞上 3s 迁移 → 丢通知；③ 迁移失败导致任务状态机读到"库打不开" |
| **影响范围** | 任务步骤延迟、通知丢失；**不会**损坏任务数据（迁移不碰 `zhuli_agent_task`） |
| **代码位置** | `InboxDatabase.onUpgrade`；`agent/SqlAgentStore`；`ZhuliAgentRuntime.init` 的 `markOrphansPaused` |
| **预防** | ① 迁移在**单个事务**内完成，失败整体回滚（不会半迁移）；② 迁移范围**不含** `zhuli_agent_task` / `zhuli_agent_step`（`git diff` 可证：`MemoryMigrator` 只碰 `zhuli_fact*` 与 `zhuli_memory_*`）；③ 耗时上界 3s（D2），超则启用 C0.2 §9.6 的分批方案；④ debug 下加线程断言，把"迁移发生在哪个线程"打进日志（A-4）；⑤ B1 的 12 场景回归（D8）验证状态机轨迹不变 |
| **验收** | D2（耗时）+ D8（12 场景）；`MemoryMigrationTimingTest` 打印 `thread=` 字段；迁移日志的 `ms` ≤ 3000 |

> 一个尚未验证的点：迁移期间若有 `NotificationListenerService` 回调，`getWritableDatabase` 会阻塞到迁移提交。C1-B5 的 D1 手工步骤里加一条"迁移进行中发一条通知"，观察是否丢失。**这是 §19 的遗留项 U-3。**

#### K-13｜`FactStatus` 扩展后，未知状态被当作当前值

| 项 | 内容 |
|---|---|
| **发生条件** | ① 某处仍用 `FactStatus.of()` 且它 fallback 到 `ACTIVE`；② 有人用 `FactStatus.valueOf(raw)`，未知值直接抛异常；③ `when(status)` 没有 `else`，新增枚举值后编译失败或走错分支；④ 投影时把 `EXPIRED` 合并进 `SUPERSEDED`、"反正都是过去" |
| **影响范围** | 记忆页把已过期/已否决/无法识别的行显示成**当前值**；模型拿到过期任务当成待办；这是 C0.2 缺陷 #2 在兼容层的翻版 |
| **代码位置** | `MemoryTypes.kt:56-71`（枚举与 `of()`）；`MemoryRepo.kt:478`（`read()`）；`MemoryRanker.kt:97/112`（过滤）；`MemoryRepo.projectStatus()`（新增） |
| **预防** | ① `of()` 改 `?: UNKNOWN`，KDoc 写明"禁止 fallback"；② 投影用**显式七分支 + else**，不用 `of()`（因为 `CANDIDATE` 必须映射成 `UNCONFIRMED`）；③ `isCurrent` / `isActionable` 只在枚举内部定义，调用方不许自己拼"等于 ACTIVE 或等于 UNCONFIRMED"的判断；④ 禁止 `valueOf`；⑤ review checklist 加一条"新增枚举值后，所有 `when(FactStatus)` 是否都有 else" |
| **验收** | `T-X01`（六态解析）；`T-X02`（未知值不落 ACTIVE）；`T-X04`（UNKNOWN 非当前、不可激活）；`T-X06`（六态一一对应）；`T-X07`（30 格可见性矩阵）；`grep "?: ACTIVE"` = 0；`grep "FactStatus.valueOf"` = 0 |

#### K-14｜建表拆分不彻底：旧表复活或核心表缺失

| 项 | 内容 |
|---|---|
| **发生条件** | ① 只改 `onCreate` 忘了 `onOpen`（或反之）；② 拆分时把 `createCoreTables` 整段删掉，连带丢了 `zhuli_agent_task` / `zhuli_agent_step` / `zhuli_fact_vec`；③ `onUpgrade` 的 `oldVersion<15` 分支漏了 `createFactTableV15`，v14 库升级时迁移前置断言失败；④ 为了"干净"顺手删掉 `zhuli_entity` 的建表 |
| **影响范围** | ②③④ 都是**灾难级**：Agent 状态机表缺失 → 任务系统崩；向量表缺失 → 索引器崩；`zhuli_entity` 缺失 → C2 的实体写入无处可去。①会让 `T-M09` 必然失败、fail-loud 失效 |
| **代码位置** | `createCoreTables` 的 3 个调用点：`InboxDatabase.kt:317`（`onCreate` 内）/ `:339`（`onOpen` 内）/ `:787`（`onUpgrade` 的 `oldVersion<15` 分支）；定义在 `:501`，函数声明在 `:269` / `:332` / `:673` |
| **预防** | ① §4.8.1 的 20 表分类表，拆分前后逐表核对；② `createCoreTablesV15` 是原函数的**真子集**（只删 fact 表 + 3 个索引），不许动别的；③ 三个调用点用 §4.8.2 的对照表逐行替换；④ `grep "fun createCoreTables\b"` 必须为 0（旧函数名彻底消失，防止有人只改一半） |
| **验收** | `T-M40`（新装无旧表、有全部核心表）；`T-M41`（重开不复活）；`T-M42`（旧写 fail-loud）；`T-M43`（onOpen 幂等）；`T-M45`（缺表不补建）；`T-M01`（v15→v17 存量事实与任务都不丢） |

#### K-15｜FTS 缓存说 `TRIGGER`、实际触发器已丢（v3 新增）

| 项 | 内容 |
|---|---|
| **发生条件** | ① 缓存写入后，触发器被外部删掉（用户清库、部分恢复、ROM 工具）；② `ensureAll` 尝试补建触发器但失败（磁盘满 / `trusted_schema=OFF` 的坏环境）；③ 缓存命中时只看 `mode` 字符串，不校验结构 |
| **影响范围** | `MemoryStore` 认为"触发器会同步"，于是**不调** `syncFts` → 新写入的记忆永远搜不到，且没有任何报错。这是"看起来正常但漏结果"里最难查的一类 |
| **代码位置** | `FtsModeCache.get()`（§6.4 第 5/6 道门）；`MemorySchema.ensureAll()` 的补建分支；`MemoryStore` 的 `if (ftsMode == APP_SYNC)` 分支 |
| **预防** | ① `get()` 在 `mode==TRIGGER` 时必须 `countFtsTriggers(db)==3`，否则返回 null 触发重探；② `mode!=NONE` 时必须 `tableExists(zhuli_memory_fts)`；③ `ensureAll` 补建过任何 FTS 对象后**主动 `prefs.remove(KEY_MODE)`**；④ `selfCheck` 第 1/2 条作为事后兜底 |
| **验收** | §9.3 的缓存失效五场景（第 ⑤ 条"删 1 个触发器"、第 ⑥ 条"DROP FTS 表"）；`T-S21`（TRIGGER 档写一次只产生 1 行）；`selfCheck` 在人为破坏后返回非空 |

---


## 14. 性能验收标准

### 14.1 迁移

| 指标 | 门槛 | 测量方式 | 依据 |
|---|---|---|---|
| 5,000 行 | **≤ 3,000 ms** | `MemoryMigrationTimingTest`（设备） | C0.2 A-3 |
| 20,000 行 | **≤ 12,000 ms** | 同上（线性外推 4×） | 本方案 |
| 单行平均 | ≤ 0.60 ms | `ms / rows` | 本方案 |
| 峰值内存增量 | ≤ 40 MB | `Debug.MemoryInfo.totalPss` 前后差 | 本方案 |
| 失败率 | 0 | 4 档库各跑 3 次 | — |

超门槛的处置见 **§5.13.1**（v3 修正）：只做"分批处理、单事务提交"，**不做**"`onUpgrade` 内分批提交"——后者在 helper 的事务模型下不成立。真正的分批提交（进度表 + 幂等重入）不在 C1-B。

### 14.2 查询

| 场景 | 数据量 | 门槛 | 索引 |
|---|---|---|---|
| `CURRENT`（单 scope，limit 200） | 20k 行 | ≤ 15 ms | `idx_mi_scope` |
| `search`（FTS MATCH + 200 候选 + 排序） | 20k 行 | ≤ 40 ms | `zhuli_memory_fts` |
| `search`（`NONE` 档，LIKE 全表） | 20k 行 | ≤ 300 ms（可接受降级） | 无 |
| `byId` | — | ≤ 1 ms | 主键 |
| `asOfWorld`（单槽） | 20k 行 | ≤ 10 ms | `idx_mi_slot` |
| `selfCheck`（10 组） | 20k 行 | ≤ 500 ms（只在自检页跑） | — |

### 14.3 体积

| 数据量 | v16 | v17 上限 | C0.2 实测 |
|---|---|---|---|
| 2 行 | 0.26 MB | 0.40 MB | — |
| 5,000 行 | 1.12 MB | 7.0 MB | 5.14× |
| 20,000 行 | 3.72 MB | **24 MB** | 23.06 MB（5.91×） |

超上限的处置（P1 修正：v2 写"核查新增的 5 个索引是否都被用到，删掉无人用的"，把"必用的 5 个"和"可评估的 7 个"说反了）：

**19 个索引按"能不能删"分三类**：

| 类 | 数量 | 清单 | 能删吗 |
|---|---|---|---|
| **A 必用（有明确且高频的查询路径）** | **5** | `idx_mi_slot`（每次 R4 取同槽）、`idx_mi_scope`（每次 scope 过滤）、`idx_mi_observed`（`active()` 排序）、`idx_mr_to`（每次 `WORLD_ASOF` 的 `NOT EXISTS`）、`idx_mev_source`（引用计数 / `selfCheck`） | **不能**。删了就是全表扫 |
| **B 低频/预防性（C1 没有查询走它，C2/C3 会用）** | **7** | `idx_mi_type_time`、`idx_mi_source`、`idx_ms_ref`、`idx_mr_from`、`idx_mrev_mem`、`idx_mrev_at`、`idx_me_entity` | **可评估**。在 20k 预演库上用 `EXPLAIN QUERY PLAN` 确认"确实无人用"，再决定是否推迟到 C2 建 |
| **C 完整性约束（唯一 / 主键自动）** | **7** | `idx_mi_dedupe`、`idx_ms_dedupe`、`idx_mev_primary`（唯一索引）+ 4 个 `sqlite_autoindex_*` | **绝对不能删**。它们不是性能优化，是数据正确性的最后一道防线（防重复、一条记忆一个主来源、复合主键唯一） |

处置顺序：

1. 先量体积（§14.3 的上限）；
2. 超了 → 只对 **B 类 7 个**跑 `EXPLAIN QUERY PLAN`，逐个确认"当前代码里没有查询用到它"；
3. 确认无人用的，**不删 DDL**，而是推迟到 C2 需要时再建（`MemorySchema` 里留注释 + 一个 `DDL_INDEX_DEFERRED_C2` 常量），这样 C2 加回来时不用改迁移；
4. **不许**为了体积删列、删唯一索引、删 `AUTOINCREMENT`——那三个都是架构（C0.2 §2.2 / §8.6）。

> 一句话：**体积问题只能从 7 个预防性索引里省，不能从"约束"和"高频查询"里省。** v2 的"核查 5 个、删掉无人用的"如果照做，最坏情况是删掉 `idx_mi_dedupe`——那不是省空间，是让重复数据重新变成可能。

### 14.4 启动（P1：v2 的算术矛盾已修）

v2 写"冷启动总增量 ≤ 300 ms = 上述之和的上界"，但表里 200 + 80 + 2 + 30 = **312**，比 300 大。矛盾的原因是：**这四个指标不在同一条路径上**——`onCreate` 与 `onOpen` 是互斥的（新装走 `onCreate`，之后每次走 `onOpen`），`detectFtsMode` 首次与缓存命中也是互斥的。把它们加起来当"总增量"本身就不对。

**按三条真实路径分别给门槛**：

| 路径 | 组成 | 门槛 | 说明 |
|---|---|---|---|
| **A 首次安装首启** | `onCreate`（7 表 + 15 索引 + `createCoreTablesV15`）200 + `detectFtsMode` 首次 80 | **≤ 280 ms** | `ensureAll` 不在 `onCreate` 里跑（表刚建完） |
| **B 升级后首启**（v16→v17） | `onUpgrade`（迁移，**另计**，见 §14.1）+ `onOpen` 的 `ensureAll` 30 + `detectFtsMode` 首次 80（`DB_VERSION` 变了，缓存必失效） | **≤ 110 ms（不含迁移）** | 迁移的 3–12s 单独看，不能混进"启动增量" |
| **C 平时冷启动** | `onOpen` 的 `ensureAll` 30 + `detectFtsMode` 缓存命中 2 | **≤ 32 ms** | 最常见路径，也是用户唯一能感知"变慢"的路径 |

| 单项指标 | 门槛 | 测量方式 |
|---|---|---|
| `onCreate` | ≤ 200 ms | `System.nanoTime()` 包住 `onCreate` |
| `detectFtsMode` 首次 | ≤ 80 ms | 建临时表 + 3 触发器 + 6 次写入 |
| `detectFtsMode` 缓存命中 | ≤ 2 ms | 读 `SharedPreferences` + 3 次 `sqlite_master` 查询（§6.4 第 5/6 道门） |
| `onOpen` 的 `ensureAll` | ≤ 30 ms | 都是 `IF NOT EXISTS` |

> **为什么缓存命中也要 2ms 而不是 0.2ms**：v3 的 `get()` 多了第 5/6 道门（查 3 个触发器 + 1 张 FTS 表），是 4 次 `sqlite_master` 查询。这 2ms 是**故意花**的——它换来"结论失效时能自愈"，比省 1.8ms 重要。
>
> **B 路径的 110ms 不含迁移**：把 3s 的迁移算进"启动增量"会让这个指标永远不达标，也会掩盖真正的启动开销。迁移是"一次性、可感知"的，它的门槛在 §14.1。

---

## 15. 各子阶段的完成条件（汇总）

| 阶段 | 完成条件（全部满足才算完成） | 证据 |
|---|---|---|
| **C1-B1** | ① 新装库有 7 表 + **15 显式 + 4 自动**索引、`user_version=17`、无 `zhuli_fact` 与 `zhuli_fact_v1`；② 连开 3 次幂等；③ `MemoryTextTest` 5 组全绿；④ `FactStatus` 七值解析 + 未知值兜底全绿（`T-X01`/`T-X02`/`T-X04`）；⑤ `git diff` 只含 **§8.5.1 的 11 个文件**（生产 7 + 测试 4） | `MemorySchemaTest` 输出 + `sqlite_master` 查询 + `T-M40` |
| **C1-B2** | ① 4 档库账本闭合（6 类终态，`failed=0`）；② 10 组不变量 + 5 条时间断言 + **第 6/6b 条状态断言（含 `NULL`）**；③ `integrity_check=ok`；④ 5 种故障注入全回滚；⑤ 设备 5k ≤3s；⑥ **`T-M41`–`T-M46` 全绿**；⑦ `oldVersion=14/15/16` 三条路径各成功一次 | `Temp/c1b/` 日志 + `MemoryMigrationTimingTest` + `MemoryMigratorTest` |
| **C1-B3** | ① 探测缓存命中（5 次打开只探 1 次）；② **缓存六道门逐条可验**（§6.4）；③ 坏环境（`trusted_schema=OFF`）返回 `APP_SYNC`；④ 三档检索都带全 4 个约束；⑤ `LIKE` 已转义；⑥ `selfCheck` 第 1/2 条能报出人为破坏 | `MemorySchemaTest` + 设备日志 |
| **C1-B4** | ① 27 个签名不变；② 现有 15 例只改 1 处即绿；③ 架构 47 条（双跑 34 条）全绿；④ `T-C57` 通过；⑤ `selfCheck` 干净库返回空；⑥ 六态投影与冲突 12 条全绿；⑦ **FTS 写入职责 4 条（`T-S21`–`T-S24`）全绿**；⑧ **并发 5 条（`T-N01`–`T-N05`）全绿** | `testDevDebugUnitTest` 报告 |
| **C1-B5** | ① 全量 **XML ≥ 457** 全绿（逻辑 75 / 执行 126）；② D1–D10 全过；③ §8.5.2 的 17 条命令（1 准备 + 16 自检）全部符合期望值；④ 报告交付 | 报告 + 命令输出 |


---

## 16. C1-B 总体退出条件

逐条对应 C0.2 §11.4 的 9 条，并给出**证据形式**：

| # | C0.2 的条件 | 本方案的证据 | 现状 |
|---|---|---|---|
| 1 | `ftsMode` 结论给出，G 组 7 条在设备通过 | C1-A 报告 §7.1（`TRIGGER`）+ `MemoryFtsInstrumentedTest` 9/9 | **已完成**（C1-A） |
| 2 | `testDevDebugUnitTest` 全绿，**XML `tests` ≥ 331 + 126 = 457**（逻辑 75 条，§10.11） | 测试报告 XML 的 `testsuite tests` 属性 | 未执行 |
| 3 | 无 `@Ignore`、无 `assumeTrue` 跳过 | grep 查 `@Ignore` 与 `assumeTrue` = 0 | 未执行 |
| 4 | `git diff --name-only BASE_IMPL..HEAD -- app/` ⊆ §8.5.1 的白名单（**21 个文件** = 7 生产新增 + 4 生产修改 + 9 测试新增 + 1 测试修改） | 命令输出逐行比对；`BASE_IMPL` 由 C1-B5 报告第一节填写 | 未执行 |
| 5 | 真实副本迁移：账本闭合、10 组 + 5 条 + 第 6/6b 条断言全 0、`integrity_check=ok` | `Temp/c1b/` 的 4 份日志 | 未执行 |
| 6 | `PRAGMA user_version=17`、`zhuli_fact` 不存在、`zhuli_fact_v1` 存在 | `MemoryMigratorTest` 的 `T-M09` | 未执行 |
| 7 | 不存在对 `zhuli_fact` 的写入路径（无双写） | `grep -rn "TABLE_FACT" app/src/main` 只剩 `createFactTableV15` 与 `MemoryMigrator` | 未执行 |
| 8 | 所有新增 public API 有 KDoc（说明"为什么"） | `grep -c "fun "` 与 `grep -c "/\*\*"` 粗核 + 人工 review | 未执行 |
| 9 | 设备 D1–D10 全过，D2 实测耗时写进报告 | C1-B5 报告 | 未执行 |

**额外增加 9 条（本方案 + D-6/D-7 + v3）**：

| # | 条件 | 理由 |
|---|---|---|
| 10 | **双跑的 51 条**在 `TRIGGER` 与 `APP_SYNC` 下都绿（K-7） | v2 写"47 条架构用例都绿"，但 A 组迁移 13 条与 D-7 6 条与 `ftsMode` 无关、只跑一遍。实际双跑的是 51 条（§10.11.2） |
| 11 | `git diff --check` 无空白错误 | 交付卫生 |
| 12 | 生产代码新增 ≤ 1,500 行（含 §8.1 的 40 行超支） | 防止"顺手重构" |
| **13** | **X-1 关闭（D-7）**：v17 库（新装与升级两条路径）都不存在 `zhuli_fact`；`onOpen` 连开 3 次不复活；旧写路径抛 `no such table` 而非静默写入；`createCoreTablesV15` 保留 entity/task/step/fact_vec 四张表与 3 个索引 | D-7 的 9 条要求与 6 项回归测试，逐条见 §4.8；证据 `T-M40`–`T-M45` |
| **14** | **X-2 关闭（D-6）**：`FactStatus` 含 6 个业务值 + `UNKNOWN`；`of()` 未知值不落 `ACTIVE`；六态投影一一对应不合并；`UNKNOWN` 非当前不可激活；序列化/反序列化/DB 兼容有测试 | D-6 的 9 条要求，逐条见 §7.12；证据 `T-X01`–`T-X08` |
| **15** | **X-3 关闭（D-6 第 7 条）**：冲突败方为 `CANDIDATE` 时 `conflicts()` 仍返回冲突对；MULTI 槽位不误报；`EXPIRED`/`REJECTED`/`DELETED`/`UNKNOWN` 不参与配对 | D-6 第 7 条；证据 `T-X09`–`T-X12` |
| **16** | **FTS 写入职责无歧义（P0-5）**：`TRIGGER` 档不双写（W-1）、`APP_SYNC` 档 7 条路径不漏写（W-2）、`NONE` 档 `search_text` 仍写且四约束不丢（W-3） | v2 只列 4 条路径、漏了 `confirm`/`resolveConflict`/`expire`；证据 `T-S21`–`T-S23` |
| **17** | **并发下状态合法（验收细节 1）**：5 个写方法各 50 次并发，0 次出现"两条 ACTIVE"、0 次未捕获 `SQLiteConstraintException` | 通知流水线与 Agent 工具可能同时写；证据 `T-N01`–`T-N05` |
| **18** | **迁移原子性不被"分批"破坏（P0-3）**：`onUpgrade` 全程单事务；`journal_mode=wal` 时显式拒绝并给出可读原因；不存在"批间提交"代码 | v2 的"`onUpgrade` 内分批提交"在 Android 嵌套事务模型下不成立；证据 = §5.1 时序表的 9 个步骤 + 代码 review |

> 用户指令只点名"X-1、X-2 纳入退出条件"。第 15 条（X-3）是同一条决策（D-6 第 7 条）的一部分；第 16–18 条是本轮 P0-3/P0-5 与验收细节 1 的产物——它们对应的缺陷在 v2 里是"文档写了但无法验证"的状态。


---

## 17. 不属于 C1-B 的后续工作

| # | 工作 | 归属 | 为什么不在 C1-B |
|---|---|---|---|
| 1 | 从通知/对话自动抽取记忆的**编排** | C2 | `MemoryExtractor` 已有抽取能力，C1 只沿用其输出；"什么时候抽、抽几条、如何合并"是编排问题 |
| 2 | 用户确认/否决的 UI 与批量流程 | C2 | C1 不改 UI（10.3） |
| 3 | 记忆注入（`MemoryContextBuilder`） | C3 | 要改 `AgentLoop`，C1 明确禁止 |
| 4 | 自动冲突裁决（"选一个"） | C2 | C1 只把冲突方放进 `CANDIDATE` 并建 `CONFLICTS_WITH`，裁决权在用户 |
| 5 | 定期整理 / 合并 / 衰减 | C4 | 需要"使用审计"（`usage` 表，C3） |
| 6 | 历史聊天回填 | C4 | 需要 `ingest` 游标表 |
| 7 | 墓碑、硬删、`usage` 表、`ingest` 游标 | C3/C4 | C0.2 §2.1 已把它们排除在 C1 之外 |
| 8 | `zhuli_fact_v1` 的清理（含备份清理） | C3 | 要先有用户二次确认的 UI |
| 9 | PROJECT / CONVERSATION 的写入 | C2/C4 | 没有可信的 id 来源（C0.2 §7.1） |
| 10 | `MemoryRepo` 的最终删除（调用方直连 Store） | C4 之后 | 现在删会把 C1 扩成"改 5 个调用方 + 重写 15 个测试" |
| 11 | 宿主 `MemoryRepository`（Room）与新表的合并 | 未排期 | 两套并存是既有决策，合并需要迁移宿主 v39 |
| 12 | `T-D32`（硬删）/ `T-A34` / `T-A35` | C3 | C0.2 已明确"C1 只留设计" |

---

## 18. 决策落定与遗留待决

### 18.1 已决定（D-6 / D-7，本轮落实）

| 决策 | 结论 | 落实位置 | 验收 |
|---|---|---|---|
| **D-6** `FactStatus` 状态投影 | 扩展为 6 个业务值 + `UNKNOWN`；`of()` 未知值不落 `ACTIVE`；`EXPIRED` / `REJECTED` / `DELETED` 语义不合并；`MemoryRanker.conflicts` 支持 `CANDIDATE`；旧 `Fact` 只做兼容投影 | §7.12（全部 9 条要求）、`MemoryTypes.kt`、`MemoryRanker.kt`、`MemoryRepo.projectStatus()` | `T-X01`–`T-X12`；退出条件 #14 / #15 |
| **D-7** `createCoreTables` 拆分 | 拆成 `createCoreTablesV15` + `createFactTableV15`；三入口职责分离；v17 起不再创建 `zhuli_fact`；其余核心表初始化一行不删 | §4.8（20 表分类 + 3 调用点 + 6 条回归）、`InboxDatabase.kt` | `T-M40`–`T-M45`；退出条件 #13 |

**决策对原方案的实质影响**（不是文字游戏，逐条可核对）：

| 项 | v1 | v2 | v3 | 原因 |
|---|---|---|---|---|
| 修改生产文件 | 2 个 | 4 个 | **4 个（不变）** | D-6 要改 `MemoryTypes.kt` 与 `MemoryRanker.kt`；v3 不新增 |
| C1-B 新增**逻辑**用例 | 47 | 65 | **75** | v2 的 +18（D-6 12 + D-7 6）；v3 再 +10（`T-M46` / `T-S21`–`T-S24` / `T-N01`–`T-N05`） |
| C1-B 新增**执行**次数 | — | 未区分 | **126** | 51 条双跑 + 24 条单跑（§10.11.2） |
| 目标 XML `tests` | 378 | 396（口径错误） | **457** | 331 + 126；v2 把逻辑数加到 XML 数上 |
| `zhuli_fact` 在 v17 的存在性 | 计划"不存在"，但方案自相矛盾（onOpen 会重建） | 结构上保证不存在，且有 4 条测试盯住 | **不变** | D-7 |
| `EXPIRED` / `REJECTED` 的投影 | 有损合并 | 一一对应，不合并 | **不变** | D-6 第 5 条 |
| 冲突展示 | 改 `MemoryRepo` 另算 | 改 `MemoryRanker.conflicts` | **不变** | D-6 第 7 条 |
| 测试预算上限 | 2,000 行 | 2,400 行 | **2,800 行** | 10 条新用例 + 白名单修正（§8.4） |
| WAL 下的迁移 | 未提 | checkpoint 后继续（**不成立**） | **拒绝** | §5.12 |
| 迁移超时处置 | 未提 | 分批提交（**不成立**） | **分批处理、单事务提交** | §5.13.1 |

### 18.2 仍然有效的 C0.2 待决（D-1–D-5）

C0.2 §12.1 的 D-1–D-5 与本文不冲突，仍然有效，需要在对应阶段前答复（不阻塞 C1-B1）：SECRET 处置的 UI 表现、旧表（`zhuli_fact_v1`）的删除时机、TASK 记忆的生命周期、物理擦除的触发方式、`AlreadyPresent` 的 UI 文案。

### 18.3 本方案不新增待决项

D-6 / D-7 落定后，C1-B **没有**新的"等你决定"事项。§19.3 的 U-1–U-15 是"未验证/解决不了"的技术事实，不是待决选择——它们不影响开工，只影响发布前的补测清单。

### 18.4 v3 的两项"撤销"（不是新增决策，是纠正错误）

v3 没有推翻任何 D-6 / D-7 的结论，但**撤销了 v2 自己在实现层面写错的两条**。它们必须显式记录，因为 C1-B1 开工时会照着 v2 写代码：

| # | v2 的错误结论 | v3 的结论 | 依据 |
|---|---|---|---|
| 1 | §5.12："检测到 WAL 就 `wal_checkpoint(FULL)`，然后继续" | **拒绝**：`require(journal_mode in 回滚日志族)`，WAL → 抛异常并给出可读原因 | `onUpgrade` 时 helper 的 T0 已持有写锁，事务内 checkpoint 自己的库不可行；且备份早在 checkpoint 之前就做完了（§5.1 / §5.12） |
| 2 | §14.1 / §5.13："超门槛 → `onUpgrade` 内的分批提交" | **只做"分批处理、单事务提交"**；真正的分批提交需要进度表 + 幂等重入，不在 C1-B | Android 嵌套事务不发 `COMMIT`；裸 `COMMIT` 会让 `setVersion(17)` 落在事务外（§5.13.1） |

> 这两条不是"方案选择"，是"技术上不成立"。v3 保留它们的错误描述在修订对照表里，方便核对"为什么改"；正文只保留正确版本，避免实现者看到两种说法。


---

## 19. 附录

### 19.1 实施顺序一览（可直接当 checklist）

```
□ B1-0  MemoryTypes.kt（D-6 枚举 + of() + isCurrent/isActionable）
□ B1-1  MemoryItem.kt        □ B1-5  MemoryGuard.kt
□ B1-2  MemoryText.kt        □ B1-6  InboxDatabase.kt（DB_VERSION + D-7 拆分 + 三入口接入）
□ B1-3  MemorySchema.kt      □ B1-7  MemoryRanker.kt（conflicts 新语义）
□ B1-4  MemoryTextTest       □ B1-8  MemorySchemaTest / MemoryFactStatusTest（解析部分）全绿

□ B2-1  MemoryMigrator 骨架（第 0 步断言：oldVersion 分支 + 拒绝 WAL）
□ B2-2  逐行映射 + RowOutcome 账本
□ B2-3  来源与证据（SECRET 先判后写）
□ B2-4  替代链 + 断链
□ B2-5  合并 + 向量
□ B2-6  FTS + 唯一索引 + 改名
□ B2-7  账本闭合 + 6 条断言（含第 6/6b 条，覆盖 NULL）
□ B2-8  sqlite_sequence（INSERT OR REPLACE）+ T-M46
□ B2-9  MemoryMigratorTest 13 + 5 + 1 条（T-M41..T-M46）

□ B3-1  FtsModeCache（六道门）    □ B3-4  attach 接入
□ B3-2  detectFtsMode 七步        □ B3-5  负向验证（trusted_schema=OFF）
□ B3-3  createFts/syncFts         □ B3-6  缓存失效五场景（§6.4）

□ B4-1  MemoryPolicy          □ B4-6  MemoryRepo 投影重写 + projectStatus()
□ B4-2  insert                □ B4-7  MemoryTestFixture + scope 用例
□ B4-3  correct/confirm/resolve□ B4-8  selfCheck
□ B4-4  softDelete/restore    □ B4-9  conflictFacts + 六态投影（T-X05..T-X12）
□ B4-5  查询 API（7 个）       □ B4-10 FTS 写入职责（T-S21..T-S24）
                              □ B4-11 并发 5 条（T-N01..T-N05）

□ B5-1  全量 XML ≥457（逻辑 75 / 执行 126）
□ B5-2  D1–D4/D6/D7/D9/D10    □ B5-5  §8.5.2 的 17 条命令全过
□ B5-3  D8（B1 12 场景）       □ B5-6  C1-B 报告
□ B5-4  D5（飞行模式）
```


### 19.2 关键命令

```bash
# 全量单测（不需要设备）
cd zhuli/lastchat
JAVA_HOME="C:/Program Files/Android/Android Studio/jbr" ./gradlew --no-daemon \
  :app:testDevDebugUnitTest -x :app:buildWebUi -x :speech:downloadSherpaOnnxAar

# 设备测试（迁移耗时 + 不变量）
./gradlew --no-daemon :app:connectedDevDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.zhuli.memory.MemoryMigrationTimingTest

# 迁移前后的表结构对照
adb -s emulator-5554 shell "run-as lastchat.rikkafork.cocolal.dev \
  sqlite3 databases/zhuli_inbox.db 'SELECT name,type FROM sqlite_master ORDER BY type,name'"

# 双写检查（应为 0）
grep -rn "INSERT INTO zhuli_fact\|UPDATE zhuli_fact " app/src/main || echo "0 处"

# D-6：禁止 fallback 到 ACTIVE（应为 0）
grep -rn "?: ACTIVE" app/src/main/java/com/zhuli/memory || echo "0 处"
grep -rn "FactStatus.valueOf" app/src/main || echo "0 处"

# D-7：旧建表函数名必须彻底消失（应为 0）
grep -rn "fun createCoreTables\b" app/src/main || echo "0 处"

# D-7：迁移后旧表不该存在（设备上，期望只返回 zhuli_fact_v1）
adb -s emulator-5554 shell "run-as lastchat.rikkafork.cocolal.dev \
  sqlite3 databases/zhuli_inbox.db \"SELECT name FROM sqlite_master WHERE name IN ('zhuli_fact','zhuli_fact_v1')\""

# 敏感日志检查（应无 content/subject/value）
grep -n "quarantined" app/src/main/java/com/zhuli/memory/MemoryMigrator.kt
```

### 19.3 仍未解决的问题（C1-B 也解决不了的）

| # | 问题 | 为什么解决不了 | 谁来解决 |
|---|---|---|---|
| U-1 | `trusted_schema` 在 API 28/29 与 34+ 的默认值是否一致 | 手上没有 API 28/29 设备 | 有设备时补测；`detectFtsMode` 已能兜住 |
| U-2 | Robolectric 为何复现不出 `trusted_schema=OFF` 的失败 | 属 Robolectric/AOSP 构建差异，不影响生产 | 记录即可 |
| U-3 | 迁移期间的通知回调是否会被写锁阻塞到丢消息 | 需要真机 + 人为制造长迁移 | C1-B5 的 D1 手工步骤 |
| U-4 | API 28（SQLite 3.22）上的 DDL 兼容性 | 无设备 | 发布前用云真机补测 |
| U-5 | `PRAGMA defensive` 读不出值（C1-A §6 A-6） | SQLite 语义 | 只能"试"，已写进探测 |
| U-6 | 20k 行迁移在低端真机（非模拟器）的耗时 | 只有模拟器 | 云真机补测 |
| U-7 | `Fact.slot` 不含 `qualifier_key`，C2 引入 qualifier 后冲突配对可能误报 | C1 的 `qualifier_key` 恒为 `''`，无差异；改 `Fact` 会破兼容层 | C2 改用 `slot_hash` 或给 `Fact` 加字段 |
| U-8 | `FactStatus` 的六态区分只存在于数据层，UI 不显示状态标签 | C1 禁止改 UI（§7.11） | C2 的记忆页改版；在那之前用户看不出"过期"与"被替代" |
| U-9 | 回填产生的大量 `CANDIDATE` 会进入 `conflicts()`，"需要裁决"可能一次显示几十上百组 | C1 的 UI 只能列表展示，没有批量操作（§7.11） | C2 的批量裁决 UI + 按证据强度排序 |
| U-10 | `createFactTableV15` 的 DDL 与 v16 现状是两份代码，未来若改 v16 会漂移 | 迁移源表结构必须逐字一致 | `T-M02` 兜住；review 时逐列对照 |
| U-11 | `conflicts()` 返回 3 组以上时 UI 的排序未定义 | 只影响展示顺序，不影响正确性 | C2 的冲突面板设计 |
| U-12 | `trusted_schema` 是**连接级** pragma，探测只证明了"探测那条连接"可用；连接池在并发时会给不同线程不同连接 | Android 没有 API 在 `SQLiteDatabaseConfiguration` 里统一设置 `trusted_schema`；C1 代码里也没有任何地方改它 | 靠 `selfCheck` 第 1/2 条事后发现；若真出现，只能降级为 `APP_SYNC`（`MemorySchema.invalidateFtsCache()` + 重探） |
| U-13 | 若将来切 WAL，`backupBeforeMigration` 在 `onUpgrade` 内的时机不再安全（WAL 未 checkpoint 的页不在主库） | 修它需要 `open class InboxDatabase` + `override getWritableDatabase/getReadableDatabase`，影响 30+ 构造点，超出 C1-B 的 diff 预算 | 单开一批：前移备份 + `T-M47`（WAL 下备份可恢复）；在那之前 §5.12 的断言会显式拒绝 WAL 迁移，不会静默生成坏备份 |
| U-14 | `SensitiveGuard.redact` 对 SENSITIVE 行的 `excerpt` 不保证 100% 脱敏（正则能力边界） | 不是 C1 引入的问题；C1 只承诺"SECRET 整行不迁入" | 与 C0.2 §14.2 阶段 3 的备份清理同批处理（C3） |
| U-15 | `T-N01`–`T-N05` 的并发窗口靠 `CountDownLatch` 缩小，不能证明"任意调度下都正确" | 无法穷举线程调度；50 次是工程折中 | 若线上出现，先看 `lastMigrationError` / revision 链是否符合 §7.5.1 |

### 19.4 术语

| 词 | 含义 |
|---|---|
| `LB` | 世界时间下界：`CASE WHEN valid_from>0 AND time_confidence<>'NONE' THEN valid_from ELSE observed_at END` |
| `UB` | 世界时间上界：裸 `valid_until`（**不设**置信度门槛） |
| `CURRENT` | 现在什么为真：`status='ACTIVE' AND LB<=now AND (UB=0 OR now<UB)` |
| `WORLD_ASOF` | 某时刻世界上什么为真（半开区间 + `CORRECTS` 排除） |
| `BELIEF_ASOF` | 某时刻我们以为什么为真（`observed_at`/`invalidated_at`） |
| `ftsMode` | `TRIGGER` / `APP_SYNC` / `NONE`，运行时探测 |
| `pre_delete_status` | 软删前的状态，恢复回它（不一律回 ACTIVE） |
| `boundary_unknown` | 单条：起点靠推断/不可信/终点未知 |
| `ambiguous` | 整次结果：同槽多值且至少一条边界未知 |
| `MemoryStatus` | 权威六态：`ACTIVE` / `CANDIDATE` / `SUPERSEDED` / `EXPIRED` / `REJECTED` / `DELETED` |
| `FactStatus` | 兼容七值：上述六态的业务映射 + `UNKNOWN`；`CANDIDATE` 映射为 `UNCONFIRMED` |
| `isActionable` | `FactStatus` 的派生属性：`ACTIVE` 或 `UNCONFIRMED`，即"可以参与冲突配对" |
| `projectStatus()` | `MemoryRepo` 私有的权威→兼容映射函数；**禁止**用 `FactStatus.of()` 代替 |
| `createCoreTablesV15` | v15 核心表（entity/task/step/fact_vec），v17 起仍创建 |
| `createFactTableV15` | 旧 `zhuli_fact` 表，**只允许** `onUpgrade` 且 `oldVersion<15` 时调用 |

---

**本文件到此结束。它是实施方案，不是实施结果。C1-B 尚未开始，生产代码未改动，`DB_VERSION` 仍为 16。**

> v2 的边界：只落实 D-6 / D-7 两项已决事项，未新增任何架构设计，未修改任何生产代码。§19.1 的 checklist 已包含 D-6/D-7 的步骤；开工前请以本文件 + 修订记录为准，C0.2 只在本文未覆盖处作为依据。

---

**—— 第二部分：v2 → v3 修订对照表（410 行）——**

# 批次 C1-B 实施方案：v2 → v3 修订对照表

> 对象：`docs/批次C1-B_记忆系统基础设施实施方案.md`
> 起点：v2 = 提交 `ba2c2f0`（2,827 行）
> 终点：v3 = 本文件同批提交（3,637 行）
> 用途：逐项回答"改了哪、为什么改、怎么验"。**不是修改摘要**——每条都能回到正文的章节号去核对。
>
> 本文分两部分：
> 1. **§1–§9 修订对照**（32 项：P0 七项、P1 八项、验收细节五项、自查追加十二项）；
> 2. **§10 仍需人工核对清单**（分"必须核对 / 需设备 / 已核对建议复核 / 无法核对"四类）。
>
> **引用约定**：本文里的 `§x.y` **全部指实施方案文档**（`批次C1-B_记忆系统基础设施实施方案.md`）的章节，不是本文的章节；写作 `C0.2 §x.y` 的是指架构依据文档；写作 `本文 §N` 的是指本文件的章节。

---

## 1. 修订总览

| 类别 | 条数 | 来源 | 落在正文 |
|---|---|---|---|
| **P0（阻断级技术错误）** | 7 | 用户指令（一） | §1.1 / §5.1 / §5.13.1 / §6.2.1 / §6.4 / §6.3 / §4.5 / §5.2 |
| **P1（口径与闭合问题）** | 8 | 用户指令（二） | §5.3.1 / §5.9 / §5.3.2 / §7.5 / §7.2 / §10.11 / §8.5.1 / §5.7 |
| **验收细节** | 5 | 用户指令（三） | §10.12 / §14.3 / §14.4 / §5.2 / §15+§16+§19 |
| **自查追加** | 12 | 本轮回读时发现 | §10.6 / §8.5.1 / §8.5.2 / §10.9.3 / §3.2 / §13.2 / §10.2 / §11.4 / §13 / §5.2 |
| **合计** | **32** | — | — |

**三个数字（v3 的全文统一口径，见 §10.11）**：

| 口径 | 含义 | v3 |
|---|---|---|
| 逻辑用例数 | 一个编号一条用例 | **75** |
| 执行次数 | 双跑展开后的实际执行 | **126**（单跑 24 + 双跑 51×2） |
| Gradle XML `tests` | `testDevDebugUnitTest` 报告的计数 | **457**（331 基线 + 126） |

**没有变的东西（必须明写，否则"修订"会被误读为"扩范围"）**：

| 项 | v2 | v3 |
|---|---|---|
| 修改的生产文件 | 4 个 | **4 个**（`InboxDatabase` / `MemoryRepo` / `MemoryTypes` / `MemoryRanker`）|
| `MemoryRepo` 公开签名 | 27 个 | **27 个，一个不改** |
| 数据库版本 | 16 → 17 | **16 → 17** |
| C0.2 架构 | 7 表 / 40 列 / 三档 FTS | **不变** |
| D-6 / D-7 决策 | 已落定 | **不撤销、不改写** |

---

## 2. P0 逐项对照（7 项）

### P0-1｜统一 Git 基线

| 项 | 内容 |
|---|---|
| **原章节** | §1.1（Git）、文件头、§16 退出条件 #4 |
| **v2 的写法** | 只写一个"当前 HEAD = `6a66c1e`"；交付报告里把 `202f2c5` 与 `394023c` 并列成 tag 的哈希，读起来像同一层级 |
| **v3 改成** | 六个具名基线：`BASE_B1=3597904` / `BASE_C1A=6a66c1e` / `BASE_C1B_V1=90088f8` / `BASE_C1B_V2=ba2c2f0` / `BASE_IMPL`（提交后产生）/ `HEAD_NOW`；加 tag 两形态表（`batch-a` 是 annotated tag，对象 `202f2c5`，指向 commit `9555bab`；`v16-baseline-20261124` 是 lightweight，直接是 commit `394023c`）；提交链扩到近 7 个 |
| **为什么** | "当前 HEAD"是个会动的值。v2 写作时确实是 `6a66c1e`，但 v2 提交后 HEAD 变成 `ba2c2f0`。若照 v2 写死，`git diff 6a66c1e..HEAD` 会把 C1-B 自己那一个 docs 提交也算进去。另外 `git rev-parse <tag>` 在附注标签上返回**标签对象**而非提交，这是 v2 那张表最容易被误读的地方 |
| **验收证据** | `git cat-file -t batch-a` → `tag`；`git cat-file -p batch-a` 的 `object` 行 → `9555bab`；`git rev-list -n1 v16-baseline-20261124` → `394023c`；§16 #4 改为 `git diff BASE_IMPL..HEAD`，`BASE_IMPL` 由 C1-B5 报告第一节现填 |

### P0-2｜迁移备份 / WAL / checkpoint / onUpgrade 事务的执行顺序

| 项 | 内容 |
|---|---|
| **原章节** | §5.1（触发点与事务边界）、§5.12（WAL） |
| **v2 的写法** | ① "嵌套事务 T1（Android 用 savepoint 实现）"；② "检测到 WAL 就 `wal_checkpoint(FULL)`，然后继续" |
| **v3 改成** | ① 显式纠正：AOSP `SQLiteSession.beginTransactionUnchecked()` 只在 `mTransactionStack == null` 时真正 `BEGIN EXCLUSIVE`，**嵌套不发 SAVEPOINT**；② 9 步时序表，逐步标注"是否在 helper 的 T0 内"与"失败后的状态"；③ 第 3 步（`onUpgrade` 第一行的文件拷贝）是**唯一合法备份点**；④ 第 7 步 `db.setVersion(17)` 在**同一事务内**；⑤ 第 9 步 `onOpen` 在**提交后**；⑥ WAL 改为**断言拒绝**（§5.12） |
| **为什么** | 三重理由：**(a) 位置错**——备份在 `onUpgrade` 第一行就做完了，checkpoint 在备份之后跑，救不了已经拷出去的半旧文件；**(b) 技术不可行**——T0 内当前连接持写锁，`wal_checkpoint` 需要写锁，checkpoint 自己的库会 `SQLITE_BUSY`；**(c) 成本不可接受**——切 WAL 前移备份要 `override getWritableDatabase()`，而 `InboxDatabase` 是 Kotlin `final class`（`InboxDatabase.kt:19`），改 `open` 会影响 30+ 处构造点 |
| **验收证据** | §5.12 的 `require(journal_mode in setOf("delete","truncate","persist"))` 代码块；§16 额外条件 #18（"`onUpgrade` 全程单事务；WAL 时显式拒绝；不存在'批间提交'代码"）；§18.4 的撤销记录 #1；U-13 记录"将来切 WAL 要做什么" |

### P0-3｜迁移超时的处置（不得声称可在 onUpgrade 内分批提交）

| 项 | 内容 |
|---|---|
| **原章节** | §5.13（耗时测试）、§14.1（耗时门槛） |
| **v2 的写法** | "超门槛 → `onUpgrade` 内的分批提交" |
| **v3 改成** | 新增 **§5.13.1**，把两件事分开：**"分批处理、单事务提交"（✅ 只做这一种）** vs **"真正分批提交"（❌ 不做）**。`batchSize` 只控制 Cursor 窗口大小，不产生任何中间 COMMIT。超门槛时按四级优先处置：① 提高门槛并记录；② 减小单批内存；③ 设备分档（低端机延后迁移）；④ 单开一批做"进度表 + 幂等重入" |
| **为什么** | Android 的嵌套事务不发 `COMMIT`。裸写 `COMMIT` 会把 helper 的 T0 提前结束，`db.setVersion(17)` 落到事务外——迁移失败时会留下"表已建、版本还是 16"的脏状态，下次开库重跑并撞上已存在的表。要做真分批，必须有进度表 + 幂等重入 + 每批独立事务，那是另一批工作的规模 |
| **验收证据** | §5.13.1 的两列对比表；§18.4 撤销记录 #2；§16 额外条件 #18；§14.1 的超门槛处置段改为引用 §5.13.1 |

### P0-4｜FTS 探测与缓存的关系

| 项 | 内容 |
|---|---|
| **原章节** | §6.2（`detectFtsMode`）、§6.4（`FtsModeCache`） |
| **v2 的写法** | 缓存列 3 个键；失效条件只写"见下"，没有可实现的 `get()`；探测结论与缓存写入的关系未定 |
| **v3 改成** | ① 新增 **§6.2.1**：缓存是**加速器不是真相源**——三有效性条件表、写入门槛表、连接级配置变化（U-12）、降级条件（唯一禁止 `APP_SYNC→NONE`）；② §6.4 重写为**完整契约**：4 个键（新增 `mem_fts_probe_db_version`）、`get()` 的**六道门**（无键 / 枚举非法 / `DB_VERSION` 变 / `sqlite_version` 变 / `TRIGGER` 档但触发器 ≠ 3 / 非 `NONE` 档但 FTS 表不存在）、`put()` 的写入顺序、4 个强制重探时机（含"`ensureAll` 补建后清缓存"）、`invalidateFtsCache()` |
| **为什么** | v2 的缓存只有一个"模式"字符串，没有任何自校验。当触发器被外部工具删掉、或 OTA 换了 SQLite 版本、或库被降级恢复时，缓存会持续返回一个**错误的**结论，而代码没有任何机会发现。六道门的成本是 4 次 `sqlite_master` 查询（约 2ms），换来"结论失效时能自愈" |
| **验收证据** | §6.4 的 `get()` 代码块；§9.3 完成条件"缓存六道门逐条可验（六场景）"；§15 B3 条件 ②；K-15（新增风险项）；U-12（连接级 pragma 的残余风险） |

### P0-5｜TRIGGER / APP_SYNC / NONE 三档的写入职责

| 项 | 内容 |
|---|---|
| **原章节** | §6.3（三档写路径）、§6.3.1（新增）、§9.3（B3 完成条件） |
| **v2 的写法** | 只列 4 个写路径（`insert` / `softDelete` / `restore` / 迁移），漏了 `confirm` / `resolveConflict` / `expire` |
| **v3 改成** | 10 行完整职责表（含 `correct` / `confirm` / `resolveConflict` / `expire` / `touchUsed` / 迁移 / 硬删 / 检索）；三条可检验的不变量 **W-1**（`TRIGGER` 档一次都不调 `syncFts`）、**W-2**（`APP_SYNC` 档凡改 `status` 或 `search_text` 必在同一事务内 `syncFts`）、**W-3**（任何档 `search_text` 都必须写）；`deleteFts` 合并进 `syncFts`（API 面 2→1）；`_au` 必须 `AFTER UPDATE OF search_text, status`；新增 §6.3.1 `NONE` 档 LIKE 必须 `ESCAPE '\'` 且转义 `%`/`_` |
| **为什么** | 漏掉的 3 个方法**都改 `status`**。`APP_SYNC` 档下漏调 `syncFts` 的直接后果是"用户点了确认，但那条记忆还是搜不到"——这是功能缺陷，不是性能问题。`_au` 不加 `OF` 限定则 `touchUsed` 每次 UPDATE 都触发全行 DELETE+INSERT，20k 库上一次检索结果的 `touchUsed` 会变成 20 次 FTS 写 |
| **验收证据** | `T-S21`（不双写）/ `T-S22`（7 条路径不漏写）/ `T-S23`（`NONE` 档四约束不丢）/ `T-S24`（同源重复 → `Duplicate` 零副作用）；§15 B4 条件 ⑦；§16 额外条件 #16；`grep -n "syncFts" MemoryStore.kt` 的每处都在 `if (ftsMode == APP_SYNC)` 分支内 |

### P0-6｜15 个索引的完整清单与创建时机

| 项 | 内容 |
|---|---|
| **原章节** | §4.5（索引与约束清单）、§4.4（建表顺序） |
| **v2 的写法** | 表里 15 行，只写名字和列；没写创建时机、没区分显式/自动、没核对唯一约束的 DDL 是否真的存在 |
| **v3 改成** | §4.5 拆三小节：**4.5.1** 15 个显式索引（12 普通 + 3 唯一）逐行给"创建时机"（`createAll(withUnique)` 的哪个分支 / 迁移第 10 步），并核对三个唯一约束在 C0.2 的 DDL 行号（`201` / `260` / `286`，三个都是 `CREATE UNIQUE INDEX`，其中两个是带 `WHERE` 的部分索引）；**4.5.2** 4 个 `sqlite_autoindex_*` 及 `PRAGMA index_list` 的断言方式；**4.5.3** 5 个 `INTEGER PRIMARY KEY AUTOINCREMENT` 是 rowid 别名、不建独立索引。总数口径写死为 **19 = 15 显式 + 4 自动** |
| **为什么** | "15 个索引"这个数字在 v2 里出现三处、含义各不相同（有时指显式、有时指含自动）。`MemorySchemaTest` 的 `EXPECTED_INDEXES` 到底该断言 15 个还是 19 个，实现者只能猜。另外三个唯一索引**不是表内 `UNIQUE` 约束**，而是独立的 `CREATE UNIQUE INDEX`（两个还带 `WHERE`），所以它们必须跟"索引"走，不能塞进 `CREATE TABLE` |
| **验收证据** | §4.5 三小节；§4.4 建表顺序补 `S7 普通索引 × 12`、小计"6 表 + 12 普通 + 3 唯一，另 4 自动"；`T-M40` + `MemorySchemaTest` 的 19 项断言；§15 B1 条件 ① |

### P0-7｜v14 / v15 升级路径的版本断言

| 项 | 内容 |
|---|---|
| **原章节** | §5.2 第 0 步（前置断言） |
| **v2 的写法** | 第 0 步断言 `PRAGMA user_version=16` |
| **v3 改成** | 删掉这条断言，改为**按 `oldVersion` 参数分支**：`1..13` 与 `14` → 源表不存在，由 `createFactTableV15` 补建，补建后断言"存在且 16 列"；`15` / `16` → 进来就应存在，缺即损坏，**抛异常不补建**（`T-M45`）；`0` 与 `>17` → 抛 `IllegalStateException` |
| **为什么** | `upgradeSteps()` 支持 `oldVersion` 从 1 到 16 的全部路径。v14/v15 库升到 v17 时 `user_version` 是 **14 或 15**，不是 16。照 v2 断言，**所有 v14/v15 存量用户升级必然失败**。而且"缺表要不要补"与"缺表算不算损坏"是两条不同的断言，v2 把它们揉成了一句 |
| **验收证据** | §5.2 的 `oldVersion` 分支表；`T-M01`（v15 库直开）/ `T-M41`（v16 重开 3 次）/ `T-M45`（v16 缺表 → 抛 + 回滚 + 不补建）；§15 B2 条件 ⑦"`oldVersion=14/15/16` 三条路径各成功一次" |

---

## 3. P1 逐项对照（8 项）

### P1-1｜未知状态统计必须覆盖 NULL

| 项 | 内容 |
|---|---|
| **原章节** | §5.3（逐字段迁移的 status 行） |
| **v2 的写法** | `status NOT IN (...)` 计未知值；未考虑 `NULL` 与空串 |
| **v3 改成** | 新增 **§5.3.1"状态映射的三种输入"**：`NULL` / 空串 / 其它 → 一律 `CANDIDATE` 且 `unknown_status++`；并给出 SQL 三值逻辑的反例——`status NOT IN ('ACTIVE',...)` 对 `NULL` 求值为 `NULL`（不真），该行既不进"已知"也不进"未知"，统计凭空少一行 |
| **为什么** | 旧列是 `TEXT NOT NULL DEFAULT 'ACTIVE'`，理论上没有 `NULL`。但手工改库、历史 beta 包、`ALTER TABLE` 补列都可能留下 `NULL`。更关键的是**统计口径**：SQL 三值逻辑会让 `NULL` 从"未知值计数"里漏掉，导致 §5.9 的账本无法闭合——而账本闭合是迁移成功的判据 |
| **验收证据** | §5.3.1 的 `IS NULL OR TRIM(...) NOT IN (...) OR TRIM(...)=''` 写法；§15 B2 条件 ②"第 6/6b 条状态断言（**含 `NULL`**）"；`T-X08`（`status='garbage'` → `CANDIDATE` + `unknown_status=1`） |

### P1-2｜迁移逐行处置账本与数量闭合

| 项 | 内容 |
|---|---|
| **原章节** | §5.9（数量闭合）、§5.10（日志）、§5.11（失败与回滚）、§11.4（不变量） |
| **v2 的写法** | 恒等式 `src == inserted + quarantined + skipped` |
| **v3 改成** | §5.9 重写为**六终态账本**：`inserted` / `merged` / `quarantined` / `skippedNoSubject` / `skippedNoValue` / `failed`；主恒等式 `src == inserted + quarantined + skippedNoSubject + skippedNoValue + failed`，另有 `merged <= inserted`（merged 是子集，不单独加）与 `failed == 0`；给出 `RowOutcome` sealed interface（`Inserted` / `Merged` / `Quarantined` / `SkippedNoSubject` / `SkippedNoValue` / `Failed`），**每条源记录恰好产生一个 `RowOutcome`**；§5.10 日志改 6 个必填账本字段；§5.11 失败表补 `failed>0` 也回滚；§11.4 不变量 ① 同步改写 |
| **为什么** | v2 的恒等式**是漏的**：① 它没给"插入失败但仍继续"的行留位置（v2 §5.2 第 2 步明确允许 `failed`）；② 它把两种完全不同的跳过（无 subject / 无 value）合成一个 `skipped`，出问题时无法定位；③ `merged` 没有位置，只能被偷偷算进 `inserted`。结果是 20k 库只要有 1 行失败，恒等式要么不成立（迁移失败），要么被人为调平（**数据说谎**） |
| **验收证据** | `RowOutcome` 代码块；6 终态表 + 主恒等式；§5.10 的日志字段表（六字段全必填）；§11.4 ①；`T-M04`（`quarantined=1`）/ `T-M05`（两个 skipped 分别计数）/ F2（单行插入失败 → 数量不闭合 → 回滚）；§15 B2 条件 ①"4 档库账本闭合（6 类终态，`failed=0`）" |

### P1-3｜SECRET 分类检查的覆盖面与发生位置

| 项 | 内容 |
|---|---|
| **原章节** | §2.7（SECRET 与来源）、§5.3（逐字段迁移）、§5.5（来源与证据伪码） |
| **v2 的写法** | 只对 `content` 判 SECRET；在映射过程中判 |
| **v3 改成** | ① §2.7 拆两行：**运行时只看 `content`**；**迁移查四个字段**（`subject` / `predicate` / `value` / `content`）；② 新增 §5.3.2"SECRET 检查的位置与顺序"——用 `classifyForMigration` 在**第 1 步**判，判中即 `return null`，**零写入**（item / source / evidence / relation / revision 全都不写）；③ §5.5 伪码改为 `classifyForMigration` 先行 |
| **为什么** | 旧表把值放在 `value` 里，`content` 只是证据摘录。只查 `content` 等于**放过整行最敏感的那个字段**（16 位卡号几乎必然出现在 `value`）。位置也必须前移：若先写 evidence 再判 SECRET，隔离行已经在库里留下了原文片段——"不迁入"就成了假话 |
| **验收证据** | §5.3.2 的四字段清单 + 零写入要求；`T-M04`（SECRET 行 → `quarantined=1`，且 item/source/evidence 三张表都查不到该行）；§15 B2 条件 ① |

### P1-4｜完整状态转换表（尤其 CANDIDATE）

| 项 | 内容 |
|---|---|
| **原章节** | §7.5（状态转换） |
| **v2 的写法** | 只有 ACTIVE / SUPERSEDED / DELETED 之间的少数转换，`CANDIDATE` 的去向散落在各处 |
| **v3 改成** | §7.5 拆三小节：**7.5.1** 10 条合法转换（每条给 WHERE 前置条件、revision 类型、`invalidated_at` / `valid_until` 的处置），含 `CANDIDATE → ACTIVE`（`confirm`）、`CANDIDATE → SUPERSEDED`（`resolveConflict` 的败方）、`CANDIDATE → DELETED`（`softDelete`）、`CANDIDATE → CANDIDATE`（更强的候选替代：旧行 SUPERSEDED + `CONFLICTS_WITH`）；**7.5.2** 6 条禁止转换 + 乐观锁失败的表现（`UPDATE ... WHERE revision=?` 影响 0 行 → 抛 `ConcurrentModification`）；**7.5.3** 并发三场景 |
| **为什么** | `CANDIDATE` 是回填与冲突的**主战场**：迁移会把大量行落成 `CANDIDATE`，`conflicts()` 又会让 ACTIVE×CANDIDATE 成对出现。v2 没有写"候选被确认/被否决/被更强的候选替代"分别怎么变，实现者只能临场发挥，而任何一种发挥都会让 `T-D46`（恢复 CANDIDATE 仍是 CANDIDATE）或 `T-X09`（ACTIVE+CANDIDATE 冲突）失败 |
| **验收证据** | §7.5.1 的 10 行表；§7.5.3 的并发三场景；`T-D44`–`T-D49`（恢复系列覆盖 4 种 `pre_delete_status`）、`T-S13`（旧 USER_CONFIRMED + 新 RULE_EXTRACTED → 新 CANDIDATE + `CONFLICTS_WITH`）、`T-X06`（`CANDIDATE` → `UNCONFIRMED`，不是 `UNKNOWN`） |

### P1-5｜`InsertResult` 的类型数量与副作用

| 项 | 内容 |
|---|---|
| **原章节** | §7.2（写入 API） |
| **v2 的写法** | 代码块里有 5 个 `data class/object`，正文却写"**四种**结果直接对应 R0–R9 的四个出口"；`Duplicate` 没有触发条件 |
| **v3 改成** | 定死为 **5 个类型**：`Created` / `Reinforced` / `Suppressed` / `Rejected` / `Duplicate`；配 `RejectReason`（`SECRET` / `EMPTY_TEXT` / `TOO_LONG`）与 `SuppressReason`（`TOMBSTONED` / `DELETED_WITHOUT_RESTORE` / `REJECTED_WITHOUT_RESTORE`）；给出"五类型 × 5 张表（item/source/evidence/relation/revision）"的完整副作用表；三条必写进 KDoc 的边界 |
| **为什么** | 代码与正文对不上，实现者会纠结 `Duplicate` 到底存不存在。更重要的是**副作用没说**：`Suppressed` 要写 revision（用于审计"为什么没记"），`Rejected` 不写（输入不合法，没有可审计对象）；`Duplicate` 零副作用，`Reinforced` 要加 evidence。不写清这些，`T-S11` 与 `T-S24` 就没有区分度 |
| **验收证据** | §7.2 的 5 类型代码块 + 副作用表；三条 KDoc 边界；`T-S11`（同值两来源 → 1 item / 2 evidence / `ingest` 返回 0）、`T-S24`（同源重复 → `Duplicate` 零副作用）；§15 B4 条件 ① |

### P1-6｜统一测试数量口径（并重算 396）

| 项 | 内容 |
|---|---|
| **原章节** | §0.1、§1.5、§9.5、§10.11、§16 #2 |
| **v2 的写法** | "65 条新增"、"目标 396"；§10.9.3 写"`tests` 计数按 `@Test` 方法算，不按参数化展开算" |
| **v3 改成** | 三口径分离：**逻辑 75 / 执行 126 / XML 457**；§10.11 拆 10.11.1（逻辑计数，含 H-3 的 10 条明细）、10.11.2（逐组执行次数：单跑 24 + 双跑 51×2 = 126）、10.11.3（三个数字的关系 + v2 的 396 错在哪）；§1.5 新增常量 `C1B_V3_CASES=10` / `C1B_LOGICAL_CASES=75` / `C1B_EXECUTED=126` / `C1B_TARGET_TESTS=457` |
| **为什么** | **396 = 331 + 65 是把两种口径相加**：65 是逻辑编号数，331 是 XML 执行数。这个加法让"双跑"在门禁里彻底不可见——51 条用例跑了两遍，CI 却只多算了一遍。另外 v2 那句"`tests` 不按参数化展开算"是**错的**：Gradle 的 `testsuite tests` 统计的就是执行过的用例数，参数化每组都计入。正因为这句错，才会算出 396。**396 不是"差一点"，是口径错** |
| **验收证据** | §10.11.1 / §10.11.2 / §10.11.3 三张表；§15 B5 条件 ①"XML ≥ 457 全绿（逻辑 75 / 执行 126）"；§16 #2；`grep -c "@Test"` 与 XML 报告的交叉验证 |

### P1-7｜生产文件白名单与测试文件白名单分开

| 项 | 内容 |
|---|---|
| **原章节** | §8.5（diff 预算）、§8.5.2（自检命令）、§9.1（B1 完成条件）、§15（阶段完成条件） |
| **v2 的写法** | §9.1 与 §15 都写"`git diff` 只含 **7 个文件**（4 新增 + 3 修改）"，把生产与测试混在一个数字里 |
| **v3 改成** | 新增 **§8.5.1"两个白名单必须分开"**：B1 阶段 = **11 个文件**（生产新增 4 + 生产修改 3 + 测试新增 3 + 测试修改 1），逐阶段追加到 B2 13 → B4 20 → B5 **21**；新增 **§8.5.2 自检命令修正期望值**（`--name-only` 期望 11 不是 4、`--stat` 期望 ~+1,550/−329 不是 +60/−329、`git status` 期望 `AM` 不是 `??`、测试文件期望 10） |
| **为什么** | B1 阶段实际会动测试文件（`MemoryTextTest` / `MemorySchemaTest` / `MemoryFactStatusTest` 的解析部分 / `MemorySqliteTest` 的断言）。混在一个数字里，验收时**无法判断"多出来的是测试还是生产"**——而这正是"不得顺手扩大生产代码范围"的唯一可执行检查点。另外 v2 的三个期望值都算错了，照抄会让自检一直红 |
| **验收证据** | §8.5.1 的两张分阶段表；§8.5.2 的 17 条命令（每条带期望值）；§15 B1 条件 ⑤；§16 #4（21 个文件） |

### P1-8｜`sqlite_sequence` 的处理前提

| 项 | 内容 |
|---|---|
| **原章节** | §5.7（ID 保真与断链） |
| **v2 的写法** | 只写"对齐 `sqlite_sequence`"，没写前提、没写写法 |
| **v3 改成** | 三点：① **前提**——本库有 9 张 `AUTOINCREMENT` 表（`inbox` / `reminders` / `ledger` / `chat` / `zhuli_entity` / `zhuli_fact` / `zhuli_agent_step` / `skill` / `cloud_log`），所以 v1–v16 的真实库**必然**有 `sqlite_sequence`；但 `T-M45` 的人造库可能绕过 `onCreate`，所以先 `tableExists` 再操作、不存在就跳过；② **写法**——必须 `INSERT OR REPLACE INTO sqlite_sequence(name, seq)`（`UPDATE` 在无行时静默不生效，会让新 ID 从 1 开始）；③ **验证**——`T-M46` 断言"迁移后不指定 id 插入，新 id == MAX(迁移 id)+1，且不撞 UNIQUE" |
| **为什么** | `sqlite_sequence` 只在库里存在 `AUTOINCREMENT` 表时才会被创建。若哪个测试用 `CREATE TABLE` 手搓一个没有 `AUTOINCREMENT` 的库，`UPDATE sqlite_sequence` 会报 `no such table`——而如果写了 `INSERT` 又没 `OR REPLACE`，会撞 UNIQUE。更隐蔽的是 `UPDATE` 在表存在但**没有该表名的行**时静默影响 0 行：代码不报错，但新插入的 ID 从 1 开始，直接撞上已迁移的 id=1 |
| **验收证据** | §5.7 的三点；§11.4 ⑥"9 张表逐张、不存在则跳过"；`T-M46`（4 条断言 + 反例说明）；§15 B2 条件 ⑥ |

---

## 4. 验收细节对照（5 项）

### A-1｜补代表性并发测试

| 项 | 内容 |
|---|---|
| **原章节** | §7.5（状态转换）、§9.4（B4 步骤）、§10.12（新增编号） |
| **v2 的写法** | 只说"每个写方法一个事务，不嵌套"，没有任何并发验证 |
| **v3 改成** | §7.5.3 给并发三场景；§10.12 新增 **`T-N01`–`T-N05`**（并发 `correct` / `confirm` / `resolveConflict` / `softDelete` / `restore`），每条 50 次、用 `CountDownLatch` 同时放行；§9.4 加 step 11；§19.3 加 U-15 |
| **为什么** | 通知流水线（`inbox` 写入后抽取）与 Agent 工具调用**可能同时写同一条记忆**。单线程测试全绿不代表并发安全：两个 `confirm` 同时跑可能出现"两条 ACTIVE"；两个 `restore` 同时跑可能出现 `AlreadyPresent` 与 `Restored` 都成功 |
| **验收证据** | `T-N01`–`T-N05` 明细表（§10.12）；§15 B4 条件 ⑧；§16 额外条件 #17（"5 个写方法各 50 次并发，0 次出现两条 ACTIVE、0 次未捕获 `SQLiteConstraintException`"）；U-15 明确"latch 不证明任意调度" |

### A-2｜说清 §14.3 的 5 个索引是哪 5 个

| 项 | 内容 |
|---|---|
| **原章节** | §14.3（体积超限处置） |
| **v2 的写法** | "核查新增的 **5 个**索引是否都被用到，删掉无人用的" |
| **v3 改成** | 19 个索引按"能不能删"分三类：**A 必用 5 个**（`idx_mi_slot` / `idx_mi_scope` / `idx_mi_observed` / `idx_mr_to` / `idx_mev_source`，逐个给出对应的高频查询）、**B 可评估 7 个**（`idx_mi_type_time` / `idx_mi_source` / `idx_ms_ref` / `idx_mr_from` / `idx_mrev_mem` / `idx_mrev_at` / `idx_me_entity`）、**C 完整性约束 7 个**（3 唯一索引 + 4 自动索引）；四步处置顺序；明确"确认无人用的不删 DDL，而是推迟到 C2 建" |
| **为什么** | v2 把"必用的 5 个"与"可评估的 7 个"**说反了**：它让人去核查那 5 个必用的，而真正能省的是另外 7 个。最坏情况是照 v2 删掉 `idx_mi_dedupe`——那不是省空间，是让重复数据重新变成可能 |
| **验收证据** | §14.3 的三分类表 + 四步处置；20k 预演库上的 `EXPLAIN QUERY PLAN` 输出（C1-B5 留存） |

### A-3｜修正 §14.4 冷启动的算术矛盾

| 项 | 内容 |
|---|---|
| **原章节** | §14.4（启动） |
| **v2 的写法** | "冷启动总增量 ≤ 300 ms = 上述之和的上界"，而表里 `200 + 80 + 2 + 30 = 312` |
| **v3 改成** | 按**三条真实路径**分别给门槛：**A 首次安装首启 ≤ 280ms**（`onCreate` 200 + 首次探测 80）、**B 升级后首启 ≤ 110ms（不含迁移）**（`ensureAll` 30 + 首次探测 80，因为 `DB_VERSION` 变了缓存必失效）、**C 平时冷启动 ≤ 32ms**（`ensureAll` 30 + 缓存命中 2）；另给单项指标表；解释"为什么缓存命中也要 2ms 不是 0.2ms" |
| **为什么** | 那四个指标**不在同一条路径上**：`onCreate` 与 `onOpen` 互斥（新装走前者，之后每次走后者），首次探测与缓存命中互斥。把它们相加当"总增量"本身就不成立，312 > 300 只是这个错误的显式症状。另外 v2 把 3–12s 的迁移混进"启动增量"，会让这个指标永远不达标，也会掩盖真正的启动开销 |
| **验收证据** | §14.4 的三条路径表 + 单项指标表；D2 设备实测（`MemoryMigrationTimingTest` 的 `ms` 与 `pss_kb`） |

### A-4｜迁移流程步骤数量改为实际数量

| 项 | 内容 |
|---|---|
| **原章节** | §5.2（迁移步骤） |
| **v2 的写法** | 标题写"十三个步骤"，表里有 14 行 |
| **v3 改成** | 标题改"**13 个动作 + 1 条已取消的动作**"，并写清口径：C0.2 §8.9 是 13 步（第 13 步为 `PRAGMA user_version=17`）；本方案**新增**第 0 步（前置断言）、**保留** 1–12 步、**取消**第 13 步（`X-6`：交给 helper）→ 表共 **14 行**，其中动作 13 个 |
| **为什么** | 两者"都不算错"但口径没写清：说"十三个步骤"时数的是动作，说"14 行"时数的是表格行。实现者按 13 数会漏掉第 0 步，按 14 数会以为有第 13 个动作。显式写"13 动作 + 1 撤销说明"后，两种读法都收敛到同一个答案 |
| **验收证据** | §5.2 标题 + 口径段 + 14 行表（第 13 行"**不做**"）；§19.1 的 `B2-1`–`B2-9` 与之对齐；§18.4 撤销记录 #2 |

### A-5｜§15 / §16 / §19 的编号、文件数、测试数、退出条件一致

| 项 | 内容 |
|---|---|
| **原章节** | §15（阶段完成条件）、§16（总退出条件）、§19.1（checklist） |
| **v2 的写法** | 三处各自都能读，但彼此对不上：§15 B1 说 7 个文件、§16 #2 说 396、§19.1 的 B5 说"≥396"、B4 没有 `T-S21`–`T-S24` 与 `T-N01`–`T-N05` |
| **v3 改成** | 三处全部重写并对齐：**文件数**统一 11 / 21（B1 / 全程）；**测试数**统一 XML ≥ 457；**完成条件** B2 加"账本闭合 + `failed=0` + `T-M46` + `oldVersion=14/15/16` 三路径"、B3 加"六道门逐条可验"、B4 加 ⑦⑧ 两项、B5 加"17 条命令"；§19.1 的 B2 扩到 9 步、B3 加 `B3-6`、B4 加 `B4-10` / `B4-11` |
| **为什么** | 这三个章节是**验收时真正被逐条念的东西**。任何一个数字对不上，执行的人就要停下来判断"以哪个为准"，而现场判断大概率会挑宽松的那个。这是"文档写了但没人验"的典型形态 |
| **验收证据** | §15 五行表、§16 的 9 + 9 条、§19.1 checklist 三处交叉核对；§19.1 的 `B5-1`（XML ≥457）与 §16 #2 与 §15 B5 ① 逐字一致 |

---

## 5. 自查追加修正（12 项）

> 这 12 项不在用户列的 20 条内，是本轮回读全文时发现的**旧口径残留**或**新引入的不一致**。不修就会重新变成"总数与明细对不上"。

| # | 原章节 | 问题 | v3 的处置 | 为什么必须修 |
|---|---|---|---|---|
| **S-1** | §10.6 | 标题写"E 组 9 条"，§10.11 只算 8 条 | 标题改"**9 条，C1 实现 8 条——`T-D32` 明列但归 C3**"，并加一段说明：列 9 条是为了 C0.2 编号链完整，`T-D32` 不计入 47、不占预算 | 9 与 8 都"对"，但读者无法判断哪个进分母。47 = 13+11+8+7+8 只有用 8 才成立 |
| **S-2** | §8.5.1 / §16 #4 | 文件总数写 19，而"7 生产新增 + 4 生产修改 + 8 测试新增 + 1 测试修改"相加等于 20；B4 的追加清单漏了 `MemoryStoreTest.kt` 与 `MemoryRepo.kt` | 改为 **21 = 7 + 4 + 9 + 1**；B4 追加 7 个（含 `MemoryRepo.kt` 与 `MemoryStoreTest.kt`）→ 20，B5 → 21；§8.5.2 #3 期望值改 10 | 等式本身不成立。逐阶段累加与分类汇总两条路必须得到同一个数，否则白名单形同虚设 |
| **S-3** | §15 B5 / §19.1 B5-5 | 写"§8.5.2 的 11 条 git 自检"，实际有 17 条命令 | 改为"**17 条命令（1 准备 + 16 自检）**"，并在 §8.5.2 开头写明"共 17 条" | 数错条数会让执行的人以为"跑完 11 条就完事"，漏掉 6 条结构断言 |
| **S-4** | §10.9.3 | v2 把 `T-X09` 归为纯函数，导致 7+4=11≠12；且"`tests` 不按参数化展开算"是错的 | `T-X09` 改判 **L2**（要用 `MemoryStore.conflictFacts()` 的 SQL）→ **8 条双跑 + 4 条单跑 = 20 次**；改用两个具名子类而非 `Parameterized`，并说明两种方式下计数一致 | K-7 的"两种模式行为一致"完全依赖这个分类。把 L2 当 L1，那条用例就只在 `TRIGGER` 下跑过 |
| **S-5** | §3.2 #26 | `vectorText` 写"只对 ACTIVE 返回"，与 §7.12.3 第 12 行"`CURRENT`"矛盾 | 统一为 **`CURRENT`**（与 #10/#11 一致） | 同一个投影函数两处判据不同，实现时必然二选一，选错就是"过期事实仍被向量检索命中" |
| **S-6** | §13.2 K-11 | 写 `MemoryTypes.kt`（**不动**），与 D-6 冲突 | 改"**D-6 授权修改**：只加 3 个枚举值与 `of()` 的兜底，不动 `Fact` 的 16 个字段" | 风险项的"预防措施"里写着"零改动"，而 D-6 明确要改——这会让 review 时把 D-6 的改动当成违规 |
| **S-7** | §13.2 K-7 | 验收写"§10 的 47 条 × 2 模式全绿" | 改"**双跑的 51 条 × 2 = 102 次 + 单跑 24 条**"；并说明 A 组 13 条与 D-7 6 条不碰 `ftsMode` | 47 × 2 = 94 ≠ 126。K-7 是"两条路径行为一致"的唯一保障，它的验收数字必须是可执行的 |
| **S-8** | §10.2 `T-M05` | 断言写"`skipped` 计数" | 改"**分别**计入 `skippedNoSubject` / `skippedNoValue`，且两者之和等于本表两行" | 与 P1-2 的六终态账本对齐。笼统的 `skipped` 无法定位"是空 subject 还是空 value" |
| **S-9** | §11.4 | 不变量 ① 仍是 `src == inserted + quarantined + skipped`；⑥ 没写前提；⑧ 只要求 `TRIGGER` 档 | ① 改 6 终态恒等式 + `merged <= inserted` + `failed == 0`；⑥ 补"9 张表逐张、不存在则跳过"；⑧ 改"`TRIGGER` 与 `APP_SYNC` 两档都要满足" | §5.9 改了而 §11.4 没改，等于**同一份文档里有两个恒等式**。预演脚本会照 §11.4 写，然后永远对不上 |
| **S-10** | §13 | 风险清单 14 项 | 加 **K-15**（FTS 缓存说 `TRIGGER`、实际触发器已丢），共 **15 项** | 这是 P0-4 六道门要防的那个具体故障。风险清单不列它，六道门就成了"没有对应风险的措施" |
| **S-11** | §5.2 | 第 0 步的源表断言表里写"由第 5 步的 `createFactTableV15` 补建"，而 §5.2 自己的第 5 步是"重算 `times_reinforced`" | 改为"由 helper 的 `upgradeSteps` 在 `oldVersion<15` 分支补建（**§5.1 时序表第 5 步，不是本节第 5 步**）" | 同一节内"第 5 步"指两个不同动作。实现者按本节理解会去 step 5 找建表代码，找不到 |
| **S-12** | §3.4 ⑥ / §4.8.2 / §8.2 / §13.2 K-14 | 行号写 `onCreate:317` / `onOpen:339` / `onUpgrade:787`，读起来像"函数声明在第 317 行"，实际声明在 **269 / 332 / 673**，317 / 339 / 787 是**调用点** | 全部改为"`onCreate` 内调用点 `:317`"等表述；K-14 补"声明在 `:269` / `:332` / `:673`，定义在 `:501`" | 行号指错会让 review 打开错误的代码位置，进而怀疑整份文档的"已核查"标记 |

---

## 6. v2 被撤销的两个结论（不是"改进"，是"技术上不成立"）

这两条必须单独列，因为 C1-B1 开工时会照着 v2 写代码。

### 撤销 1｜WAL 下 `checkpoint` 后继续

| 项 | 内容 |
|---|---|
| **v2 原文** | §5.12："检测到 `journal_mode=wal` 就执行 `wal_checkpoint(FULL)`，然后继续迁移" |
| **为什么不成立** | ① `backupBeforeMigration` 是 `onUpgrade` 的**第一行**，备份早已完成，checkpoint 在后，救不了已拷出的文件；② helper 的 T0 已 `BEGIN EXCLUSIVE`，当前连接持写锁，而 `wal_checkpoint` 需要写锁 → `SQLITE_BUSY`；③ 想真正修好要把备份前移到 `getWritableDatabase()` 之前，需要 `open class` + `override`，影响 30+ 构造点 |
| **v3 的结论** | `require(mode.lowercase() in setOf("delete","truncate","persist"))`，WAL 直接抛，异常消息写明"备份已在事务内完成，WAL 下该备份不完整；请先切回 delete 模式并重新打开库" |
| **残余风险** | U-13：将来若切 WAL，需单开一批做"前移备份 + `T-M47`（WAL 下备份可恢复）"。在那之前断言会显式拒绝，**不会静默生成坏备份** |

### 撤销 2｜`onUpgrade` 内的分批提交

| 项 | 内容 |
|---|---|
| **v2 原文** | §14.1 / §5.13："超过耗时门槛 → 在 `onUpgrade` 内分批提交" |
| **为什么不成立** | Android 的嵌套事务不发 `COMMIT`（`SQLiteSession` 只在 `mTransactionStack == null` 时 `BEGIN`，只有最外层 `endTransaction()` 才提交）。裸写 `COMMIT` 会提前结束 helper 的 T0，`db.setVersion(17)` 落到事务外——迁移失败时留下"表已建、版本还是 16"的脏状态 |
| **v3 的结论** | 只做"**分批处理、单事务提交**"：`batchSize` 只控制 Cursor 窗口，不产生任何中间提交。超门槛按四级优先处置（提门槛并记录 / 减小单批内存 / 设备分档 / 单开一批做进度表 + 幂等重入） |
| **残余风险** | 低端机 20k 行可能超过 12s（U-6）。当前方案是"让它慢但正确"，而不是"快但可能半迁移" |

---

## 7. 数字口径总账（防止后面再混）

| 数字 | 含义 | 出现在 |
|---|---|---|
| **57** | C0.2 §11.2 定义的架构用例总数（A13+B11+C8+D7+E9+F2+G7） | §10.11.1 的"全口径"列 |
| **54** | 其中 C1 需要实现的（57 − F 的 2 条 − `T-D32` 1 条） | §10.11.1 |
| **47** | C1-B **新增**的架构用例（54 − G 组 7 条已在 C1-A） | §10.1 / §15 B4 |
| **28** | D-6 的 12 + D-7 的 6 + v3 的 10 | §10.9 |
| **75** | C1-B 新增**逻辑**用例（47 + 28） | §0.1 / §1.5 / §10.11.1 / §15 |
| **24** | 单跑的执行次数（A13 + D-6 的 L1 4 + D-7 6 + `T-M46` 1） | §10.11.2 |
| **51** | 双跑的用例条数（B11 + C8 + D7 + E8 + D-6 的 L2 8 + `T-S` 4 + `T-N` 5） | §10.11.2 |
| **126** | 新增**执行**次数（24 + 51×2） | §1.5 / §10.11.2 / §15 |
| **331** | C1-A 结束时的 XML 基线 | §1.5 / §16 #2 |
| **457** | C1-B 目标的 XML `tests`（331 + 126） | §16 #2 / §19.1 B5-1 |
| **15 + 4 = 19** | 15 个显式索引（12 普通 + 3 唯一）+ 4 个自动索引 | §4.5 / §8.4 / §14.3 |
| **11 / 21** | B1 阶段 / 全过程的文件白名单 | §8.5.1 / §15 B1 / §16 #4 |
| **13 + 1** | 迁移的 13 个动作 + 1 条已取消的动作 | §5.2 |
| **9** | 迁移的 9 个时序步骤（§5.1） | §5.1 / §16 #18 |
| **2,600 / 2,760 / 2,800** | 新增测试代码（不含 fixture）/ 含 fixture / 上限 | §8.4 / §8.5 |
| **15** | 风险项 K-1–K-15 | §13 |
| **15** | 未解决事项 U-1–U-15 | §19.3 |

---

## 8. 修订要求的落实情况（用户原文四条）

| 要求 | 落实方式 | 可核对处 |
|---|---|---|
| **不得为修某个问题顺手扩大生产代码范围** | 修改的生产文件**仍是 4 个**；§16 额外条件 #12（生产新增 ≤1,500 行）；§5.1 明确"不把 `InboxDatabase` 改成 `open`"；§5.13.1 明确"不做真分批" | §8.2 / §8.5 / §16 #12 |
| **不得修改 `MemoryRepo` 的 27 个公开签名** | §3.2 逐行列出 27 个签名 + 新实现落点；§15 B4 条件 ①"27 个签名不变" | §3.2 / §9.4 / §15 |
| **不得把 C2/C3 工作提前塞入 C1-B** | §7.11"明确不做的事"；§17 的 12 条"不属于 C1-B"；§7.5.2 的 6 条禁止转换；F 组 2 条与 `T-D32` 明列但不实现 | §7.11 / §17 / §10.6 / §10.7 |
| **不得把未验证的设备行为写成已验证事实** | §0.4 的四种陈述类型（已核查 / 设计决定 / 待验证 / 未知）；U-1–U-15 全部标注"解决不了"与"谁来解"；§14.4 的耗时为估算并注明"未测" | §0.4 / §19.3 / §14.4 |
| **保留未解决的技术事实，不得靠删问题制造"全部解决"** | v3 没有删除任何一条 v1/v2 提出的问题；U-1–U-11 原样保留，新增 U-12–U-15；K-1–K-15 新增 K-15；§18.4 保留两条"撤销"的原文描述 | §19.3 / §13 / §18.4 |

---

## 9. 本轮没做的事（边界声明）

| 没做 | 原因 |
|---|---|
| 改任何生产代码 | v3 是 docs-only 修订；`DB_VERSION` 仍是 16，`app/src` 零改动 |
| 跑测试 | C1-B 尚未开工，测试文件不存在 |
| 改 C0.2 | C0.2 是冻结的架构依据。D-6 与 §10.3 的冲突按"后出优先 + 具体优先"处理，差异记在本文 §3 与修订记录 v2 |
| 改 D-6 / D-7 的结论 | v3 只修实现层面的错误，不推翻决策 |
| 清理 `Temp/` | 明令禁止（§11.6） |
| 补齐"§10 各表 T- 编号 vs C0.2 §11.2"的逐条映射 | 需要对照 C0.2 全文 57 条逐条核，属人工核对项（见 §10 A-1） |

---

## 10. 仍需人工核对的事项清单

> 分四类。**A 类没核完不要开工**——它们错了会导致返工，而不是"跑一次测试就发现"。

### A. 必须人工核对（错了会返工）

| # | 事项 | 怎么核 | 我为什么没核完 |
|---|---|---|---|
| **A-1** | §10 各表的 `T-` 编号与 C0.2 §11.2 的 57 条逐条对应（编号、名称、断言） | 打开 C0.2 §11.2，逐行比对 §10.2–§10.8 的用例名 | 需要通读 C0.2 全文的用例表；本轮只核了**数量**（57 = 13+11+8+7+9+2+7）与**分组归属** |
| **A-2** | 51 条双跑的判据是否成立——即 B/C/D/E 组每条是否**真的**调用 `MemoryStore` 的写方法 | 逐条看断言：凡"用 `insert` 造数据"或"走 `softDelete`"的才算；纯读取的不算 | 判据已写死在 §10.11.2，但逐条归类是我按用例名推的；若某条 B 组用例其实用 `insertRaw` 造数据，它就应是单跑，126 要减 1 |
| **A-3** | §3.2 的 27 个签名与 `MemoryRepo.kt` 实际的 public 成员一一对应 | `grep -n "fun \|val \|var " app/src/main/java/com/zhuli/memory/MemoryRepo.kt` 后逐个勾 | 27 这个数字来自 C0.2 与 v1，本轮只核了"D-6 授权的改动不影响签名" |
| **A-4** | §4.2 的 40 列与 C0.2 §2.2 逐列一致（列名 / 类型 / 默认值 / 是否可空） | 两表并排比对 | 40 这个数字核过，逐列的默认值没逐条比对 |
| **A-5** | §4.5 的 15 个索引名与 C0.2 §6 的 DDL 逐字一致，尤其 3 个唯一索引的 `WHERE` 子句 | 对照 C0.2 的 201 / 260 / 286 行 | 三个唯一索引**确实是** `CREATE UNIQUE INDEX`（已核）；名字与 `WHERE` 的具体写法未逐字比 |
| **A-6** | §5.3.1 的三类输入（`NULL` / 空串 / 其它）是否覆盖真实脏数据 | 对 4 个真库跑 `SELECT DISTINCT status FROM zhuli_fact` | 手上没有 4 个真库的副本（`Temp/c1b/` 尚未生成） |
| **A-7** | §5.7 的 9 张 `AUTOINCREMENT` 表清单是否完整 | `grep -rn "AUTOINCREMENT" app/src/main` 后去掉注释与字符串外的匹配 | 我 grep 到 **9 处**（276/403/434/455/505/519/565/641/659），但若某张表的 DDL 是动态拼的，grep 可能漏 |
| **A-8** | §8.4 的 2,760 行估算 | 实现后 `git diff --stat` | 只能实现后才知道；2,800 是上限不是目标 |
| **A-9** | §14.4 的 200 / 80 / 30 / 2 ms | 设备实测 | **全部是估算**，v3 只保证了它们**不在同一条路径上相加** |
| **A-10** | §5.13 的 3–12s 迁移耗时 | `MemoryMigrationTimingTest` 在模拟器/真机跑 1k/5k/20k | 当前数字来自桌面 Python 的 0.17–0.21 ms/行推算，**不是设备数据**（§11.5 已声明） |

### B. 需要设备才能核对

| # | 事项 | 现状 |
|---|---|---|
| **B-1** | U-1：`trusted_schema` 在 API 28/29 与 34+ 的默认值是否一致 | 无 API 28/29 设备；`detectFtsMode` 已能兜住 |
| **B-2** | U-4：API 28（SQLite 3.22）上的 DDL 兼容性 | 无设备；发布前用云真机补测 |
| **B-3** | U-6：20k 行在低端真机（非模拟器）的耗时 | 只有模拟器 |
| **B-4** | `PRAGMA journal_mode` 在设备上的返回值大小写（`DELETE` vs `delete`） | v3 已用 `.lowercase()` 兜住，但只证明了"逻辑上兜住"，没在设备上验过 |
| **B-5** | `T-M46` 的"新 id 从 1 开始"反例能否在 Robolectric 上复现 | Robolectric 的 SQLite 与设备不完全一致；若复现不出，这条用例就抓不到 P1-8 的缺陷 |
| **B-6** | F5（迁移中途 `kill -9` / `am force-stop`） | 无法在 Robolectric 模拟，只能手工（§11.3） |

### C. 已核对，但建议复核

| # | 事项 | 我的核对结论 |
|---|---|---|
| **C-1** | 21 个文件白名单与 §8.1–§8.4 的逐行对应 | 已逐条对过（7 + 4 + 9 + 1），且逐阶段累加也是 21 |
| **C-2** | 三个唯一约束的 DDL 类型 | 三个都是 `CREATE UNIQUE INDEX`（不是表内 `UNIQUE`），两个带 `WHERE` |
| **C-3** | `sqlite_sequence` 是否存在 | 9 处 `AUTOINCREMENT`，`inbox` 表 v1 就存在 → 真实库必然有 |
| **C-4** | `batch-a` 的 tag 类型 | `git cat-file -t batch-a` → `tag`（附注）；`object` 行 → `9555bab` |
| **C-5** | `v16-baseline-20261124` 的 tag 类型 | lightweight，`git rev-parse` 直接给 commit `394023c` |
| **C-6** | `createCoreTables` 的 3 个调用点 | `:317`（`onCreate` 内）/ `:339`（`onOpen` 内）/ `:787`（`onUpgrade` 的 `oldVersion<15` 分支）；声明在 `:269` / `:332` / `:673`，定义在 `:501` |
| **C-7** | §16 的 9 + 9 条编号连续、§19.3 的 U-1–U-15 连续、§13 的 K-1–K-15 连续 | 已用脚本核过（无缺号、无重号） |
| **C-8** | §5.1 的 9 步时序与 §16 #18 的"9 个步骤"一致 | 一致 |

### D. 无法核对（明确说"核不了"）

| # | 事项 | 为什么核不了 |
|---|---|---|
| **D-1** | C0.2 §11.2 的 57 条本身是否完整（有没有漏编号） | 那是 C0.2 的自我一致性，不属本轮范围；我只核了"C1-B 引用它的数量对不对" |
| **D-2** | Robolectric 与真机在 `sqlite_version()` 上是否一致 | 影响 `FtsModeCache` 的第 4 道门（`sqlite_version` 变则失效）。在模拟器与真机上各读一次即可，但需要设备 |
| **D-3** | `SensitiveGuard.redact` 的实际脱敏覆盖率 | U-14：正则能力边界，需专项测试；C1 只承诺"SECRET 整行不迁入" |
| **D-4** | 并发测试能否覆盖真实调度 | U-15：`CountDownLatch` 只缩小窗口，不能穷举 |

---

## 11. 一句话总结

v3 没有新增任何功能、没有扩大生产代码范围、没有推翻 D-6/D-7；它做的是**把 v2 里"技术上不成立的两条"删掉、把"数字口径混用的三处"拆清、把"写了但无法验收的十一条"补上可执行判据**。剩下的不确定项（A 类 10 条、B 类 6 条、D 类 4 条）如实列在 §10，**没有一条被"删掉问题"的方式掩盖**。

开工前请先过 A 类。A-1 与 A-2 若与实际情况不符，`75 / 126 / 457` 三个数字要一起改，不能只改一个。
