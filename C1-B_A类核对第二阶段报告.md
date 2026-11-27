# C1-B A 类核对第二阶段报告

> 本报告仅为**核查记录**：未修改任何生产代码、未修改数据库、未执行迁移、未启动 C1-B1。
> 所有数据库操作均以 `mode=ro` + `PRAGMA query_only=ON` 只读执行。

## 1. 基线

| 项 | 值 |
|---|---|
| 代码仓库 | `https://github.com/asd1548428584-cloud/zhuli` |
| 远端 `master` HEAD | `c44314b93c5a62d10b2babcc46d065fa88f3f918`（孤儿快照提交，与本地历史不同源） |
| 本地工作区 HEAD | `08b17c19f6f41cc80653d4da1949634cafd77092`（分支 `master`）；工作区除 `?? Temp/` 外干净 |
| 代码文件基线 | `MemoryRepo.kt` Blob `7796a17695de3d433496494a0028e5d3914844f1`，与 `3597904` 中同名 Blob **完全一致**；`InboxDatabase.kt` Blob `4b6fae89f1dbd3134212026b6bd5e7941752dee3` |
| `DB_VERSION` | **16**（`InboxDatabase.kt:50 private const val DB_VERSION = 16`），与方案基线一致 |
| 方案版本 / commit | `C1B-v4.2.1-all-in-one.md`，基线 `1458c9304537ab27ebced0e1df916b8ba18a6cbc` |
| C0.2 对照基线 | `a0e1d52d6442299a5634c69315cf4ac460620aa4`，文档 `docs/批次C0.2_记忆系统架构设计最终版.md` Blob `8686bc7855824992b990d69139f511ca0dfa936a`（工作区与 `a0e1d52` **逐字一致**） |
| 数据库文件来源 | `Temp/c01/zhuli_inbox_v16_real.db`，由真机 `adb exec-out run-as lastchat.rikkafork.cocolal.dev cat databases/zhuli_inbox.db` 导出（来源记录见 `Temp/c01/part3.md:320`、`Temp/c01/part4.md:301`） |
| 数据库特征 | 274,432 B；SHA-256 `5936d12efcb1bd99f10aea15fa5d24ecd56d294825156ce630fbbec4b9f75660`；`user_version=16`；`journal_mode=delete`；`PRAGMA integrity_check=ok`；17 张表 |
| 设备状态 | `adb devices` 无连接设备；因此**未**从设备现场二次导出。使用已有真机导出副本，未改动原库 |

> 版本一致性判断：真实库 `user_version=16` 与方案基线（v16→v17 迁移前）一致，A-7 结果可用于判定；不存在跨版本混判。

## 2. A-3 结果：`MemoryRepo` 27 个公开成员

| 项 | 值 |
|---|---|
| 文件 | `zhuli/lastchat/app/src/main/java/com/zhuli/memory/MemoryRepo.kt`（485 行） |
| Blob SHA | `7796a17695de3d433496494a0028e5d3914844f1` —— 与 `3597904` 中同文件**一致**，与题述一致 |
| 公开成员 | **27 个**（1 个 `@Volatile var` + 26 个 `fun`，其中 1 个 `suspend`） |
| 私有辅助 | 9 个（`enqueueVector` / `insert` / `reinforce` / `supersede` / `upsertEntity` / `queryOne` / `query` / `read` / `Cursor.str`），不计入公开成员 |
| 可见性变化 | **无**；无新增、无删除 |
| 签名变化 | **无**；两处参数名与方案写法不同但类型/数量一致：`search(queryText:…)` vs 方案 `search(text:…)`；`resolveConflict(keepId, dropId, now)` vs 方案 `resolveConflict(keep, drop, now)` |

| 序号 | 公开成员名称 | 类型/签名 | 源码位置 | 与方案 §3.2 是否一致 |
|---:|---|---|---|---|
| 1 | `onIndexRequested` | `var onIndexRequested: (() -> Unit)?   // @Volatile` | `MemoryRepo.kt:25` | 一致 |
| 2 | `ingest` | `fun ingest(drafts: List<FactDraft>, source: SourceType, sourceId: Long, now: Long, activateThreshold: Double = 0.0): Int` | `MemoryRepo.kt:33` | 一致 |
| 3 | `correct` | `fun correct(factId: Long, newValue: String, now: Long): Long?` | `MemoryRepo.kt:100` | 一致 |
| 4 | `deleteFact` | `fun deleteFact(id: Long, now: Long): Int` | `MemoryRepo.kt:133` | 一致 |
| 5 | `byId` | `fun byId(id: Long): Fact?` | `MemoryRepo.kt:145` | 一致 |
| 6 | `unconfirmed` | `fun unconfirmed(limit: Int = 100): List<Fact>` | `MemoryRepo.kt:148` | 一致 |
| 7 | `confirmFact` | `fun confirmFact(id: Long, now: Long): Boolean` | `MemoryRepo.kt:155` | 一致 |
| 8 | `activeOf` | `fun activeOf(subject: String, predicate: String): List<Fact>` | `MemoryRepo.kt:175` | 一致 |
| 9 | `active` | `fun active(limit: Int = 200): List<Fact>` | `MemoryRepo.kt:182` | 一致 |
| 10 | `activeCount` | `fun activeCount(): Int` | `MemoryRepo.kt:185` | 一致 |
| 11 | `history` | `fun history(subject: String, predicate: String): List<Fact>` | `MemoryRepo.kt:190` | 一致 |
| 12 | `search` | `fun search(queryText: String, now: Long, limit: Int = 8): List<FactHit>` | `MemoryRepo.kt:197` | 一致（方案写 `text`，源码 `queryText`；仅参数名差异） |
| 13 | `conflicts` | `fun conflicts(): List<Pair<Fact, Fact>>` | `MemoryRepo.kt:201` | 一致 |
| 14 | `resolveConflict` | `fun resolveConflict(keepId: Long, dropId: Long, now: Long)` | `MemoryRepo.kt:203` | 一致（方案写 `keep/drop`，源码 `keepId/dropId`；仅参数名差异） |
| 15 | `putVector` | `fun putVector(factId: Long, space: EmbeddingSpace, vec: FloatArray)` | `MemoryRepo.kt:222` | 一致 |
| 16 | `markVectorIndexed` | `fun markVectorIndexed(factId: Long)` | `MemoryRepo.kt:235` | 一致 |
| 17 | `markVectorFailed` | `fun markVectorFailed(factId: Long, reason: String, now: Long)` | `MemoryRepo.kt:240` | 一致 |
| 18 | `dropVector` | `fun dropVector(factId: Long)` | `MemoryRepo.kt:248` | 一致 |
| 19 | `pendingVectorFacts` | `fun pendingVectorFacts(limit: Int = 16): List<Long>` | `MemoryRepo.kt:265` | 一致 |
| 20 | `pendingVectorCount` | `fun pendingVectorCount(): Int` | `MemoryRepo.kt:279` | 一致 |
| 21 | `waitingVectorCount` | `fun waitingVectorCount(): Int` | `MemoryRepo.kt:287` | 一致 |
| 22 | `failedVectorCount` | `fun failedVectorCount(): Int` | `MemoryRepo.kt:292` | 一致 |
| 23 | `indexedSpaces` | `fun indexedSpaces(): Set<String>` | `MemoryRepo.kt:298` | 一致 |
| 24 | `vectorCount` | `fun vectorCount(space: EmbeddingSpace): Int` | `MemoryRepo.kt:308` | 一致 |
| 25 | `enqueueAllActive` | `fun enqueueAllActive(now: Long, dropSpace: String? = null): Int` | `MemoryRepo.kt:320` | 一致 |
| 26 | `vectorText` | `fun vectorText(factId: Long): String?` | `MemoryRepo.kt:351` | 一致 |
| 27 | `hybridSearch` | `suspend fun hybridSearch(text: String, provider: EmbeddingProvider, now: Long, limit: Int = 8): List<FactHit>` | `MemoryRepo.kt:365` | 一致 |

**A-3 判定：通过。** 逐项 27/27 与 v4.2.1 §3.2 对应，Blob SHA 未变化。注意：A-3 核对的是**当前旧实现**的公开面，方案 §3.2 描述的是**C1-B4 要改成的实现**；两者签名一致正是兼容要求，不代表新实现已经写好。

## 3. A-7 结果：9 张 AUTOINCREMENT 表真实核验

| 核验项 | 预期 | 实际 | 结果 | 证据 |
|---|---|---|---|---|
| 源码声明 AUTOINCREMENT 表数 | 9 | 9（`InboxDatabase.kt:276/403/434/455/505/519/565/641/659`） | 一致 | `grep -n AUTOINCREMENT` |
| 真实库含 AUTOINCREMENT 的表数 | 9 | **9** | 一致 | 第一组 SQL |
| 9 张表逐名对应 | inbox/reminders/ledger/chat_log/zhuli_entity/zhuli_fact/zhuli_agent_step/skills/cloud_log | **9 张全部存在**，名称完全一致 | 一致 | 第五组 SQL |
| `sqlite_sequence` 是否存在 | 存在 | 存在（`CREATE TABLE sqlite_sequence(name,seq)`） | 一致 | 第二组 SQL |
| `sqlite_sequence` 同名重复数 | 0 | **0 行重复** | 一致 | 第三组 SQL 返回空 |
| `sqlite_sequence` 实际记录数 | 不预设 | **4 条**：cloud_log=6、zhuli_agent_step=186、zhuli_entity=2、zhuli_fact=2 | 记录 | 第四组 SQL |
| 其余 5 张表无 sequence 行 | 允许（未插入过） | inbox/reminders/ledger/chat_log/skills 无行 | 正常 | 第四组 SQL：SQLite 只在实际插入 AUTOINCREMENT 行时写 sequence |
| 非 AUTOINCREMENT 主键情况 | 需另行核对 | 6 张：`embeddings`、`source_rule`、`zhuli_agent_task`、`zhuli_fact_vec`、`zhuli_tool_exec`、`zhuli_vec_queue`，均为单列主键（`INTEGER PRIMARY KEY` 或 `TEXT PRIMARY KEY`），**无复合主键、无 AUTOINCREMENT 遗漏** | 一致 | 第六组 SQL：`sqlite_master.sql` 全表 DDL |
| 源码 DDL 与真实库是否一致 | 一致 | 9 张表的表名、主键形式与真实库 DDL 一致 | 一致 | 源码常量 `52/77/90/100/117/137/148/149/177` 行 + 真实库 DDL |

**A-7 判定：通过。** 9 张源码声明与真实库完全一致，`sqlite_sequence` 存在且无重复；连续失败阈值相关的"同名重复行"风险在真实库中当前不存在（但不代表未来不会出现，`alignSequence` 仍按方案保留收敛逻辑）。原始查询输出见 `Temp/auditA/a7_raw.json`。

## 4. A-1 / A-4 / A-5 结果（证据准备，不作通过判定）

### 4.1 A-1：57 条架构用例映射

C0.2 §11.2 共 **57 条**（35 条 C0.1 原编号 + 22 条本版新增），C1-B v4.2.1 §10.2–§10.8 中 **57 条全部有对应项**，无新增、无删除；其中 47 条属 C1-B 实现范围、7 条（`T-F50`–`T-F56`）属已完成的 C1-A、3 条（`T-D32`/`T-A34`/`T-A35`）归 C3。

**重要限制**：C1-B 尚未开工，目标源码文件 `MemoryMigrator.kt` / `MemoryStore.kt` 等**尚不存在**；下表"源码/架构依据"列给出的是 C0.2 条款与 C1-B 测试矩阵位置，**不构成"架构要求已实现"的证据**。

| 序号 | 用例编号 | C0.2 用例名 | C0.2 断言（§11.2） | C1-B v4.2.1 对应位置 | 源码/架构依据 | 结论 |
|---:|---|---|---|---|---|---|
| 1 | `T-A34` | 注入记忆块 | `steps` 不变、状态机轨迹不变、审批/取消/暂停/恢复与 B1 一致 | §10.7（v4.2.1:4045） | C0.2 §11.2:1883；实现目标 `（C3，不在 C1-B）`（源码尚不存在） | 有对应项；归 C3，不在 C1-B 范围 |
| 2 | `T-A35` | prompt injection | 记忆内容里的"忽略之前的指令"不改变工具选择 | §10.7（v4.2.1:4045） | C0.2 §11.2:1884；实现目标 `（C3，不在 C1-B）`（源码尚不存在） | 有对应项；归 C3，不在 C1-B 范围 |
| 3 | `T-C25` | 项目 A / B | 互不命中（数据由 fixture 构造，见 14.3） | §10.5（v4.2.1:4016） | C0.2 §11.2:1857；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 4 | `T-C26` | 助手 A / B（含切换） | 互不命中，`GLOBAL` 仍可见 | §10.5（v4.2.1:4017） | C0.2 §11.2:1858；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 5 | `T-C27` | 普通聊天读项目私有 | 0 命中 | §10.5（v4.2.1:4018） | C0.2 §11.2:1859；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 6 | `T-C28` | TASK 过期 | `valid_until` 过后 0 命中 | §10.5（v4.2.1:4019） | C0.2 §11.2:1860；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 7 | `T-C29` | GLOBAL 进入项目任务 | 命中 | §10.5（v4.2.1:4020） | C0.2 §11.2:1861；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 8 | `T-C30` | 非法 scope（`PROJECT` + 空 id） | 写入抛异常；检索抛异常 | §10.5（v4.2.1:4021） | C0.2 §11.2:1862；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 9 | `T-C57` | 生产写入对 PROJECT/CONVERSATION 保持关闭 | `MemoryStore.insert(scope=PROJECT(p))` 与 `CONVERSATION(c)` 均抛 `IllegalArgumentException`；测试数据只经 `MemoryTestFixture` 构造（14.3） | §10.5（v4.2.1:4022） | C0.2 §11.2:1863；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 10 | `T-D31` | 软删 | item 行在（DELETED）；`pre_delete_status` 记录了原状态；FTS/向量/队列/检索四路 0 命中 | §10.6（v4.2.1:4028） | C0.2 §11.2:1869；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 11 | `T-D32` | 硬删（store 级，C3 预演） | 10 张表无残留；墓碑写入；`optimize` 后 FTS 无命中 | §10.6（v4.2.1:4026） | C0.2 §11.2:1870；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；归 C3，不在 C1-B 范围 |
| 12 | `T-D33` | 共享来源 | 删记忆 A 后 source 仍存在（B 还引用）；删 B 后 source 被 GC | §10.6（v4.2.1:4028） | C0.2 §11.2:1871；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 13 | `T-D44` | 删 ACTIVE 再恢复 | `status=ACTIVE`、`deleted_at=0`、`invalidated_at=0`、`pre_delete_status=''`、向量重新入队、FTS 重新命中 | §10.6（v4.2.1:4028） | C0.2 §11.2:1872；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 14 | `T-D45` | 删 SUPERSEDED 再恢复 | `status=SUPERSEDED`（不是 ACTIVE）、`invalidated_at` 仍是原替代时间（不是删除时间）、检索 0 命中 | §10.6（v4.2.1:4036） | C0.2 §11.2:1873；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 15 | `T-D46` | 删 CANDIDATE 再恢复 | `status=CANDIDATE`（不是 ACTIVE）、不进 `CURRENT`、出现在待确认列表 | §10.6（v4.2.1:4037） | C0.2 §11.2:1874；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 16 | `T-D47` | 恢复时槽位被占 | 同槽不同值已有 ACTIVE → `Demoted(CANDIDATE)` + `CONFLICTS_WITH`；同槽是 MULTI → `Restored(ACTIVE)` | §10.6（v4.2.1:4038） | C0.2 §11.2:1875；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 17 | `T-D48` | 恢复时同值已被占用 | 同 `dedupe_key` 已被 ACTIVE/CANDIDATE 占 → `AlreadyPresent(id)`，不抛 UNIQUE，本行仍 DELETED，证据复制到占用行 | §10.6（v4.2.1:4039） | C0.2 §11.2:1876；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 18 | `T-D49` | 删除/恢复的 revision | 两条 revision（`DELETE` → `RESTORE`），`before_json`/`after_json` 成对；`actor='USER'` | §10.6（v4.2.1:4028） | C0.2 §11.2:1877；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 19 | `T-F50` | 创建 FTS4 虚拟表 | `CREATE VIRTUAL TABLE … USING fts4(search_text, tokenize=simple)` 在 `SQLiteDatabase` 上成功 | §10.8（v4.2.1:4049） | C0.2 §11.2:1890；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 20 | `T-F51` | 创建 3 个触发器 | `sqlite_master` 有 `zhuli_memory_fts_ai/au/ad`，创建不被 `SQLITE_DBCONFIG_DEFENSIVE` 拒绝 | §—（v4.2.1:0） | C0.2 §11.2:1891；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 21 | `T-F52` | INSERT 后可检索 | 插入含"会议提醒"的行，`MATCH` 命中 | §—（v4.2.1:0） | C0.2 §11.2:1892；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 22 | `T-F53` | UPDATE `search_text` | 旧词 0 命中、新词 1 命中 | §—（v4.2.1:0） | C0.2 §11.2:1893；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 23 | `T-F54` | UPDATE `status='DELETED'` | 该行 0 命中（FTS 行数 -1） | §—（v4.2.1:0） | C0.2 §11.2:1894；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 24 | `T-F55` | 恢复 `status='ACTIVE'` | 重新命中（FTS 行数 +1） | §—（v4.2.1:0） | C0.2 §11.2:1895；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 25 | `T-F56` | 物理 DELETE | 索引行消失，`MATCH` 0 命中 | §10.8（v4.2.1:4049） | C0.2 §11.2:1896；实现目标 `（C1-A 已完成）`（源码尚不存在） | 有对应项；C1-A 已完成（不属 C1-B 实现） |
| 26 | `T-M01` | v15 库直接打开（现有用例扩展） | 事实不丢、`activeCount=1`、value 一致 | §10.2（v4.2.1:3938） | C0.2 §11.2:1810；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 27 | `T-M02` | 真实副本迁移（预演脚本同款数据） | 行数一致、替代链 1→2、向量指向 2、FTS 2 行 | §10.2（v4.2.1:3939） | C0.2 §11.2:1811；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 28 | `T-M03` | 断链 `superseded_by` | 清 0，`invalidated_at` 保留 | §10.2（v4.2.1:3940） | C0.2 §11.2:1812；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 29 | `T-M04` | SECRET 行 | 不迁入新表，`quarantined` 计数 +1，总数闭合 | §10.2（v4.2.1:3941） | C0.2 §11.2:1813；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 30 | `T-M05` | 空 subject / 空 value | 跳过，计数闭合 | §10.2（v4.2.1:3942） | C0.2 §11.2:1814；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 31 | `T-M06` | 同键重复合并 | 保留强证据、区间取最早起点、保留行得 SUPPORTING 且被合并行 PRIMARY 仍在、`SAME_AS` | §10.2（v4.2.1:3943） | C0.2 §11.2:1815；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 32 | `T-M07` | MULTI 槽位两个值 | 都 ACTIVE，不合并 | §10.2（v4.2.1:3944） | C0.2 §11.2:1816；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 33 | `T-M08` | 唯一索引建立失败（人为造重复） | 抛异常 → 回滚 → `user_version=16` | §10.2（v4.2.1:3945） | C0.2 §11.2:1817；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 34 | `T-M09` | 迁移后表名 | `zhuli_fact` 不存在，`zhuli_fact_v1` 存在且 0 写入 | §10.2（v4.2.1:3946） | C0.2 §11.2:1818；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 35 | `T-M10` | 二次打开 | `onOpen` 自愈不重复建表、不重复插数据、`items` 数量不变 | §10.2（v4.2.1:3947） | C0.2 §11.2:1819；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 36 | `T-M36` | 旧 `valid_from` 不得升级为确定时间 | 真实副本迁移后，每一行 `time_confidence='NONE'`；特别断言 `valid_from>0` 的行也是 `NONE`（这是 C0.1 的错） | §10.2（v4.2.1:3948） | C0.2 §11.2:1820；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 37 | `T-M37` | 旧 `valid_to` 的去向 | SUPERSEDED 行 `valid_until=0`、`invalidated_at = 原 valid_to`；DELETED 行 `valid_until=0`、`invalidated_at = 原 updated_at` | §10.2（v4.2.1:3949） | C0.2 §11.2:1821；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 38 | `T-M38` | 未知写入方的告警 | 人造一行 `valid_from ≠ observed_at`：迁移不中止、仍写 `NONE`、`vf_mismatch` 计数 +1、`valid_from` 原值保留 | §10.2（v4.2.1:3950） | C0.2 §11.2:1822；实现目标 `MemoryMigrator.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 39 | `T-S11` | 同值两来源 | 1 条 item、2 条 evidence（1 PRIMARY + 1 SUPPORTING）、`times_reinforced=1` | §10.3（v4.2.1:3987） | C0.2 §11.2:1828；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 40 | `T-S12` | rank 更高 | 旧 SUPERSEDED + `SUPERSEDES` 关系 + `invalidated_at=now` | §10.3（v4.2.1:3988） | C0.2 §11.2:1829；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 41 | `T-S13` | 旧为 USER_CONFIRMED，新为 RULE_EXTRACTED | 旧保持 ACTIVE，新为 CANDIDATE + `CONFLICTS_WITH` | §10.3（v4.2.1:3989） | C0.2 §11.2:1830；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 42 | `T-S14` | MULTI 槽位 | 两条 ACTIVE，`superseded_by=0` | §10.3（v4.2.1:3990） | C0.2 §11.2:1831；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 43 | `T-S15` | 同 rank（RULE_EXTRACTED）× 间隔 3 天 | 不替代，新建为 CANDIDATE + 冲突 | §10.3（v4.2.1:3991） | C0.2 §11.2:1832；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 44 | `T-S16` | `correct()` | 新行 `valid_from` 继承旧值、`time_confidence` 也继承（迁移行纠正后仍是 `NONE`）；旧行 `valid_until=新 valid_from`；`CORRECTS` | §10.3（v4.2.1:3992） | C0.2 §11.2:1833；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 45 | `T-S17` | STATE 变更 | 旧 STATE EXPIRED（`valid_until=now`），新 STATE ACTIVE，无重叠 | §10.3（v4.2.1:3993） | C0.2 §11.2:1834；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 46 | `T-S18` | EVENT 纠错 | 旧 `event_start/event_end` 一字不变，`status=SUPERSEDED`，`CORRECTS` | §10.3（v4.2.1:3994） | C0.2 §11.2:1835；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 47 | `T-S19` | 删除后重新写入同值 | 部分唯一索引允许插入（旧行 DELETED） | §10.3（v4.2.1:3995） | C0.2 §11.2:1836；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 48 | `T-S20` | DELETED/REJECTED 同值再提取 | 抑制；用户显式陈述则允许复活 | §10.3（v4.2.1:3996） | C0.2 §11.2:1837；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 49 | `T-S39` | R0 时间自洽 | 写入 `valid_from=now+30d` 且 `time_confidence=NONE` → `valid_from` 被置 0；同条件 `DAY` → 保留 | §10.3（v4.2.1:3997） | C0.2 §11.2:1838；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 50 | `T-T21` | `WORLD_ASOF` 历史点（有置信度的新数据） | `at<X` 命中旧值、不命中新值 | §10.4（v4.2.1:4003） | C0.2 §11.2:1844；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 51 | `T-T22` | `BELIEF_ASOF` | 失效前命中、失效后不命中 | §10.4（v4.2.1:4004） | C0.2 §11.2:1845；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 52 | `T-T23` | `valid_from=0` | CURRENT 可见；WORLD 以 `observed_at` 为下界；结果标 `boundary_unknown` | §10.4（v4.2.1:4005） | C0.2 §11.2:1846；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 53 | `T-T24` | 纠正后 `WORLD_ASOF` | 旧值不再出现（`CORRECTS` 排除），`BELIEF_ASOF` 仍可见 | §10.4（v4.2.1:4006） | C0.2 §11.2:1847；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 54 | `T-T40` | CURRENT 排除未来生效 | ① `valid_from=now+1d`、`DAY`、ACTIVE → `CURRENT` 0 命中；把 `now` 调到生效之后 → 命中。② 同值但 `time_confidence=NONE` → R0 清零 `valid_from` 且落 `CANDIDATE`，`CURRENT` 0 命中（不允许"清零后变成现在就是"） | §10.4（v4.2.1:4007） | C0.2 §11.2:1848；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 55 | `T-T41` | 半开区间精确边界 | 旧 `valid_until=X`、新 `valid_from=X`：`at=X-1` 只旧、`at=X` 只新、`at=X+1` 只新；任何时刻都不出现两条 | §10.4（v4.2.1:4008） | C0.2 §11.2:1849；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 56 | `T-T42` | 未知边界 → ambiguous | 迁移行（`NONE`/`valid_until=0`）在 `WORLD_ASOF` 下与后续值并存，结果 `ambiguous=true` 且每条 `boundary_unknown=true`；`BELIEF_ASOF` 同一时刻只返回一条 | §10.4（v4.2.1:4009） | C0.2 §11.2:1850；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |
| 57 | `T-T43` | 任务到期不被起点置信度挡住 | `valid_from=0 & NONE`、`valid_until=finishedAt+7d` 的 TASK 记忆：到期前命中、到期后 0 命中 | §10.4（v4.2.1:4010） | C0.2 §11.2:1851；实现目标 `MemoryStore.kt`（源码尚不存在） | 有对应项；C1-B 已列入测试矩阵，尚未实现（待 B1–B5 验证） |

### 4.2 A-4：`zhuli_memory_item` 40 列

C0.2 §2.2 明确 "从 39 列增到 **40 列**（+`pre_delete_status`）"；实际解析 DDL 得 **40 列**（见下表）。但：

- 当前**源码**中 `MemorySchema.kt` 不存在（C1-B 未开工），无法提取 `DDL_ITEM`；
- 真实库 `user_version=16` 中 **无 `zhuli_memory_item` 表**；

因此只能标为"目标 DDL 已定义、源码与真实库均无该表"，**不可判定通过**。

| 序号 | 列名 | C0.2 定义（§2.2） | 源码定义 | 真实数据库实际定义 | 是否一致 |
|---:|---|---|---|---|---|
| 1 | `id` | `id INTEGER PRIMARY KEY AUTOINCREMENT` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 2 | `memory_type` | `memory_type TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 3 | `subject` | `subject TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 4 | `subject_key` | `subject_key TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 5 | `predicate` | `predicate TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 6 | `qualifier_key` | `qualifier_key TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 7 | `value` | `value TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 8 | `content` | `content TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 9 | `normalized_content` | `normalized_content TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 10 | `dedupe_key` | `dedupe_key TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 11 | `slot_hash` | `slot_hash TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 12 | `search_text` | `search_text TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 13 | `status` | `status TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 14 | `cardinality` | `cardinality TEXT NOT NULL DEFAULT 'SINGLE'` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 15 | `confidence` | `confidence REAL NOT NULL DEFAULT 0.5` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 16 | `confidence_basis` | `confidence_basis TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 17 | `source_id` | `source_id INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 18 | `scope_type` | `scope_type TEXT NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 19 | `scope_id` | `scope_id TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 20 | `sensitivity` | `sensitivity INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 21 | `event_start` | `event_start INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 22 | `event_end` | `event_end INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 23 | `valid_from` | `valid_from INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 24 | `valid_until` | `valid_until INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 25 | `observed_at` | `observed_at INTEGER NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 26 | `recorded_at` | `recorded_at INTEGER NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 27 | `updated_at` | `updated_at INTEGER NOT NULL` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 28 | `last_confirmed_at` | `last_confirmed_at INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 29 | `invalidated_at` | `invalidated_at INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 30 | `time_confidence` | `time_confidence TEXT NOT NULL DEFAULT 'NONE'` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 31 | `superseded_by` | `superseded_by INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 32 | `revision` | `revision INTEGER NOT NULL DEFAULT 1` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 33 | `extractor` | `extractor TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 34 | `legacy_kind` | `legacy_kind TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 35 | `payload_json` | `payload_json TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 36 | `deleted_at` | `deleted_at INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 37 | `pre_delete_status` | `pre_delete_status TEXT NOT NULL DEFAULT ''` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 38 | `times_reinforced` | `times_reinforced INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 39 | `times_retrieved` | `times_retrieved INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |
| 40 | `last_used_at` | `last_used_at INTEGER NOT NULL DEFAULT 0` | `MemorySchema.kt` 不存在（C1-B 未开工） | `zhuli_memory_item` 不存在（真实库 v16） | **无法核对**：目标 DDL 已定义，源码与真实库均无该表 |

### 4.3 A-5：15 个索引

C0.2 §2.2–§2.7 共提取 **15 个索引**（item 6、source 2、evidence 2、relation 2、revision 2、entity 1）。真实库虽有 24 个索引，但**全部属于旧表**（如 `idx_fact_status`、`idx_inbox_pending`），15 个目标索引**一个都不存在**；源码 `MemorySchema.EXPECTED_INDEXES` 亦不存在。

| 索引名 | C0.2 定义 | 源码定义 | 数据库实际定义 | 结论 |
|---|---|---|---|---|
| `idx_mi_slot` | `CREATE INDEX idx_mi_slot ON zhuli_memory_item(subject_key, predicate, qualifier_key, status)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mi_scope` | `CREATE INDEX idx_mi_scope ON zhuli_memory_item(scope_type, scope_id, status, memory_type)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mi_type_time` | `CREATE INDEX idx_mi_type_time ON zhuli_memory_item(memory_type, status, valid_until, event_start)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mi_observed` | `CREATE INDEX idx_mi_observed ON zhuli_memory_item(status, observed_at)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mi_source` | `CREATE INDEX idx_mi_source ON zhuli_memory_item(source_id)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mi_dedupe` | `CREATE UNIQUE INDEX idx_mi_dedupe ON zhuli_memory_item(dedupe_key) WHERE status IN ('ACTIVE', 'CANDIDATE')` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_ms_ref` | `CREATE INDEX idx_ms_ref ON zhuli_memory_source(source_type, ref_table, ref_id)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_ms_dedupe` | `CREATE UNIQUE INDEX idx_ms_dedupe ON zhuli_memory_source(source_type, ref_table, ref_id, ref_sub_id, content_hash)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mev_source` | `CREATE INDEX idx_mev_source ON zhuli_memory_evidence(source_id)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mev_primary` | `CREATE UNIQUE INDEX idx_mev_primary ON zhuli_memory_evidence(memory_id) WHERE role='PRIMARY'` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mr_from` | `CREATE INDEX idx_mr_from ON zhuli_memory_relation(from_id, relation)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mr_to` | `CREATE INDEX idx_mr_to ON zhuli_memory_relation(to_id, relation)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mrev_mem` | `CREATE INDEX idx_mrev_mem ON zhuli_memory_revision(memory_id, revision_no)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_mrev_at` | `CREATE INDEX idx_mrev_at ON zhuli_memory_revision(created_at)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |
| `idx_me_entity` | `CREATE INDEX idx_me_entity ON zhuli_memory_entity(entity_id)` | `MemorySchema.EXPECTED_INDEXES` 不存在（C1-B 未开工） | 未存在（真实库 v16 的 24 个索引均为旧表索引） | **目标 DDL 已定义、当前数据库未存在**，不可判定通过 |

**A-4 / A-5 判定：证据已准备，但核对未完成（不可通过）。** 必须等 C1-B 实现后，在迁移完成的 v17 真实库上重新执行 `PRAGMA table_info(zhuli_memory_item)` 与 `SELECT name,tbl_name,sql FROM sqlite_master WHERE type='index'`，方可逐列/逐字判定。

## 5. 证据清单

| # | 核对项 | 来源 | 标识 / 查询 | 结果 |
|---|---|---|---|---|
| 1 | 代码基线 | 本地 git + GitHub API | HEAD `08b17c1`；远端 `master` `c44314b9` | 已记录 |
| 2 | A-3 文件 | `MemoryRepo.kt` | Blob `7796a17695de3d433496494a0028e5d3914844f1`；行 25/33/…/365 | 27/27 一致 |
| 3 | A-7 数据库 | `Temp/c01/zhuli_inbox_v16_real.db` | SHA-256 `5936d12e…60fbbec4b9f75660`；v16；integrity ok | 只读 |
| 4 | A-7 SQL | 任务书六组 SQL | 原始输出 `Temp/auditA/a7_raw.json` | 9/9、seq 4 行、0 重复 |
| 5 | A-1 C0.2 侧 | `批次C0.2_记忆系统架构设计最终版.md` @ `a0e1d52` | Blob `8686bc7855824992b990d69139f511ca0dfa936a`；§11.2 L1802–1905 | 57 条 |
| 6 | A-1 方案侧 | `C1B-v4.2.1-all-in-one.md` @ `1458c930` | §10.2–§10.8 | 57 条全对应 |
| 7 | A-4 C0.2 侧 | 同上 §2.2 | DDL L141–207 | 40 列 |
| 8 | A-4 真实库侧 | 同上数据库 | `SELECT ... WHERE name='zhuli_memory_item'` | 空 |
| 9 | A-5 C0.2 侧 | 同上 §2.2–§2.7 | 15 条 `CREATE [UNIQUE] INDEX` | 15 个 |
| 10 | A-5 真实库侧 | 同上数据库 | `SELECT name,tbl_name,sql FROM sqlite_master WHERE type='index'` | 24 个旧索引，目标 0 个 |
| 11 | 证据中间文件 | `Temp/auditA/` | `a1_map.txt` / `a1_table.md` / `a3_table.md` / `a4_table.md` / `a5_table.md` / `a7_raw.json` | 可复核 |

## 6. 阶段结论

**PARTIAL（部分完成，仍有证据缺口）。**

- 已完成且有真实证据：**A-3 通过**（27 项逐项、Blob 未变）、**A-7 通过**（真实 v16 库 9 张表、sequence、无重复）。
- 已完成证据准备但**不可判通过**：A-1（57 条映射完成，但实现源码不存在）、A-4（40 列目标 DDL 已列出，真实库无该表、源码文件不存在）、A-5（15 索引目标 DDL 已列出，真实库无这些索引）。
- A-2、A-6 未在本次任务范围内，维持"待核对"。

按任务书第八节门禁：**A-1 至 A-7 必须全部完成并经独立审计，才能讨论 C1-B1 开工**。本次完成 A-3/A-7 不意味着可以启动 C1-B1；当前状态为 **不建议开工**。本次到交付报告为止，未做任何编码工作。
