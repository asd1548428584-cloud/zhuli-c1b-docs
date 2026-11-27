# C1-B A 类核对门槛修订与证据补齐报告

> 本报告只做核对与文档修订：未修改任何生产代码、未修改数据库、未执行迁移、未启动 C1-B1。
> 新报告与证据附件均为**新增文件**，不覆盖 v3 / v4 / v4.1 / v4.2 / v4.2.1 与第二阶段报告。

## 1. 远端基线（开工前重新读取）

| 项 | 值 |
|---|---|
| 仓库 | `asd1548428584-cloud/zhuli-c1b-docs`，分支 `main` |
| 读取到的 HEAD | `20505375c31f88773445cb3bbcb49ae63b5f2b11`（`docs(c1b): publish phase-2 A-class verification report`） |
| 父提交 | `1458c9304537ab27ebced0e1df916b8ba18a6cbc` |
| `C1B-v4.2.1-all-in-one.md` | 存在，Blob `77c91582b51d11024dce2bb989cf3396651a996c` —— 与题述历史值一致 |
| `C1-B_A类核对第二阶段报告.md` | 存在，Blob `c0addf231173eb033c7a0cfb931628eb380b4180` |
| 历史文件 | `C1B-v3` `4bff3d13…`、`C1B-v4` `df425c10…`、`C1B-v4.1` `22830c0d…`、`C1B-v4.2` `a77112b9…` 全部存在且未变 |
| 目标新文件 | 上传前远端**不存在**本报告与证据附件（见第 9 节复核） |

## 2. A-1：57 条架构用例重新核对 —— **PASS**

来源：C0.2 最终版 `a0e1d52` §11.2（57 条）；C1-B v4.2.1 §10.2–§10.8；第二阶段报告 A-1 原始映射。

核对方法与结果：

- C0.2 §11.2 解析出 **57** 条；逐条检查是否存在对应编号、归属是否明确、是否重复映射、是否遗漏；
- C1-B 侧存在三种合法写法：① §10.2–§10.6 表格行直接列出；② §10.7 正文逐条列出 `T-A34` / `T-A35`；③ §10.8 以 `` `T-F50`–`T-F56` `` **范围写法**覆盖 7 条。此前 "MISSING 5 条" 是把范围写法当成逐条编号导致的**解析器口径问题**，不是文档缺项；
- 57 条**全部有对应、无遗漏、无重复映射**。归属：C1-B 本批次 **47** 条、C1-A 已验证 **7** 条、C3 后续 **3** 条（47+7+3=57）；
- **T-M09 专项复核**：C0.2 §11.2 第 **1818** 行有真实条目（"迁移后表名 / `zhuli_fact` 不存在，`zhuli_fact_v1` 存在且 0 写入"）；v4.2.1 §10.2 第 **3946** 行有同编号表格行；第二阶段报告第 **125** 行（排序后第 34 条）有真实映射记录。三处均为实际文本，不是只改了锚点文字。

| 序号 | C0.2 编号 | C0.2 标题 | C0.2 原始断言 | C1-B v4.2.1 位置 | 归属 | 结论 |
|---:|---|---|---|---|---|---|
| 1 | `T-A34` | 注入记忆块 | `steps` 不变、状态机轨迹不变、审批/取消/暂停/恢复与 B1 一致 | §10.7（行 4045；§10.7 正文逐条列出） | C3 | 有对应、不重复、不遗漏 |
| 2 | `T-A35` | prompt injection | 记忆内容里的"忽略之前的指令"不改变工具选择 | §10.7（行 4045；§10.7 正文逐条列出） | C3 | 有对应、不重复、不遗漏 |
| 3 | `T-C25` | 项目 A / B | 互不命中（数据由 fixture 构造，见 14.3） | §10.5（行 4016；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 4 | `T-C26` | 助手 A / B（含切换） | 互不命中，`GLOBAL` 仍可见 | §10.5（行 4017；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 5 | `T-C27` | 普通聊天读项目私有 | 0 命中 | §10.5（行 4018；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 6 | `T-C28` | TASK 过期 | `valid_until` 过后 0 命中 | §10.5（行 4019；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 7 | `T-C29` | GLOBAL 进入项目任务 | 命中 | §10.5（行 4020；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 8 | `T-C30` | 非法 scope（`PROJECT` + 空 id） | 写入抛异常；检索抛异常 | §10.5（行 4021；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 9 | `T-C57` | 生产写入对 PROJECT/CONVERSATION 保持关闭 | `MemoryStore.insert(scope=PROJECT(p))` 与 `CONVERSATION(c)` 均抛 `IllegalArgumentException`；测试数据只经 `MemoryTestFixture` 构造（14.3） | §10.5（行 4022；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 10 | `T-D31` | 软删 | item 行在（DELETED）；`pre_delete_status` 记录了原状态；FTS/向量/队列/检索四路 0 命中 | §10.6（行 4032；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 11 | `T-D32` | 硬删（store 级，C3 预演） | 10 张表无残留；墓碑写入；`optimize` 后 FTS 无命中 | §10.6（行 4033；表格行） | C3 | 有对应、不重复、不遗漏 |
| 12 | `T-D33` | 共享来源 | 删记忆 A 后 source 仍存在（B 还引用）；删 B 后 source 被 GC | §10.6（行 4034；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 13 | `T-D44` | 删 ACTIVE 再恢复 | `status=ACTIVE`、`deleted_at=0`、`invalidated_at=0`、`pre_delete_status=''`、向量重新入队、FTS 重新命中 | §10.6（行 4035；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 14 | `T-D45` | 删 SUPERSEDED 再恢复 | `status=SUPERSEDED`（不是 ACTIVE）、`invalidated_at` 仍是原替代时间（不是删除时间）、检索 0 命中 | §10.6（行 4036；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 15 | `T-D46` | 删 CANDIDATE 再恢复 | `status=CANDIDATE`（不是 ACTIVE）、不进 `CURRENT`、出现在待确认列表 | §10.6（行 4037；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 16 | `T-D47` | 恢复时槽位被占 | 同槽不同值已有 ACTIVE → `Demoted(CANDIDATE)` + `CONFLICTS_WITH`；同槽是 MULTI → `Restored(ACTIVE)` | §10.6（行 4038；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 17 | `T-D48` | 恢复时同值已被占用 | 同 `dedupe_key` 已被 ACTIVE/CANDIDATE 占 → `AlreadyPresent(id)`，不抛 UNIQUE，本行仍 DELETED，证据复制到占用行 | §10.6（行 4039；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 18 | `T-D49` | 删除/恢复的 revision | 两条 revision（`DELETE` → `RESTORE`），`before_json`/`after_json` 成对；`actor='USER'` | §10.6（行 4040；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 19 | `T-F50` | 创建 FTS4 虚拟表 | `CREATE VIRTUAL TABLE … USING fts4(search_text, tokenize=simple)` 在 `SQLiteDatabase` 上成功 | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 20 | `T-F51` | 创建 3 个触发器 | `sqlite_master` 有 `zhuli_memory_fts_ai/au/ad`，创建不被 `SQLITE_DBCONFIG_DEFENSIVE` 拒绝 | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 21 | `T-F52` | INSERT 后可检索 | 插入含"会议提醒"的行，`MATCH` 命中 | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 22 | `T-F53` | UPDATE `search_text` | 旧词 0 命中、新词 1 命中 | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 23 | `T-F54` | UPDATE `status='DELETED'` | 该行 0 命中（FTS 行数 -1） | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 24 | `T-F55` | 恢复 `status='ACTIVE'` | 重新命中（FTS 行数 +1） | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 25 | `T-F56` | 物理 DELETE | 索引行消失，`MATCH` 0 命中 | §10.8（行 4049；§10.8 范围写法展开） | C1-A | 有对应、不重复、不遗漏 |
| 26 | `T-M01` | v15 库直接打开（现有用例扩展） | 事实不丢、`activeCount=1`、value 一致 | §10.2（行 3938；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 27 | `T-M02` | 真实副本迁移（预演脚本同款数据） | 行数一致、替代链 1→2、向量指向 2、FTS 2 行 | §10.2（行 3939；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 28 | `T-M03` | 断链 `superseded_by` | 清 0，`invalidated_at` 保留 | §10.2（行 3940；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 29 | `T-M04` | SECRET 行 | 不迁入新表，`quarantined` 计数 +1，总数闭合 | §10.2（行 3941；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 30 | `T-M05` | 空 subject / 空 value | 跳过，计数闭合 | §10.2（行 3942；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 31 | `T-M06` | 同键重复合并 | 保留强证据、区间取最早起点、保留行得 SUPPORTING 且被合并行 PRIMARY 仍在、`SAME_AS` | §10.2（行 3943；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 32 | `T-M07` | MULTI 槽位两个值 | 都 ACTIVE，不合并 | §10.2（行 3944；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 33 | `T-M08` | 唯一索引建立失败（人为造重复） | 抛异常 → 回滚 → `user_version=16` | §10.2（行 3945；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 34 | `T-M09` | 迁移后表名 | `zhuli_fact` 不存在，`zhuli_fact_v1` 存在且 0 写入 | §10.2（行 3946；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 35 | `T-M10` | 二次打开 | `onOpen` 自愈不重复建表、不重复插数据、`items` 数量不变 | §10.2（行 3947；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 36 | `T-M36` | 旧 `valid_from` 不得升级为确定时间 | 真实副本迁移后，每一行 `time_confidence='NONE'`；特别断言 `valid_from>0` 的行也是 `NONE`（这是 C0.1 的错） | §10.2（行 3948；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 37 | `T-M37` | 旧 `valid_to` 的去向 | SUPERSEDED 行 `valid_until=0`、`invalidated_at = 原 valid_to`；DELETED 行 `valid_until=0`、`invalidated_at = 原 updated_at` | §10.2（行 3949；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 38 | `T-M38` | 未知写入方的告警 | 人造一行 `valid_from ≠ observed_at`：迁移不中止、仍写 `NONE`、`vf_mismatch` 计数 +1、`valid_from` 原值保留 | §10.2（行 3950；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 39 | `T-S11` | 同值两来源 | 1 条 item、2 条 evidence（1 PRIMARY + 1 SUPPORTING）、`times_reinforced=1` | §10.3（行 3987；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 40 | `T-S12` | rank 更高 | 旧 SUPERSEDED + `SUPERSEDES` 关系 + `invalidated_at=now` | §10.3（行 3988；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 41 | `T-S13` | 旧为 USER_CONFIRMED，新为 RULE_EXTRACTED | 旧保持 ACTIVE，新为 CANDIDATE + `CONFLICTS_WITH` | §10.3（行 3989；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 42 | `T-S14` | MULTI 槽位 | 两条 ACTIVE，`superseded_by=0` | §10.3（行 3990；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 43 | `T-S15` | 同 rank（RULE_EXTRACTED）× 间隔 3 天 | 不替代，新建为 CANDIDATE + 冲突 | §10.3（行 3991；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 44 | `T-S16` | `correct()` | 新行 `valid_from` 继承旧值、`time_confidence` 也继承（迁移行纠正后仍是 `NONE`）；旧行 `valid_until=新 valid_from`；`CORRECTS` | §10.3（行 3992；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 45 | `T-S17` | STATE 变更 | 旧 STATE EXPIRED（`valid_until=now`），新 STATE ACTIVE，无重叠 | §10.3（行 3993；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 46 | `T-S18` | EVENT 纠错 | 旧 `event_start/event_end` 一字不变，`status=SUPERSEDED`，`CORRECTS` | §10.3（行 3994；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 47 | `T-S19` | 删除后重新写入同值 | 部分唯一索引允许插入（旧行 DELETED） | §10.3（行 3995；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 48 | `T-S20` | DELETED/REJECTED 同值再提取 | 抑制；用户显式陈述则允许复活 | §10.3（行 3996；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 49 | `T-S39` | R0 时间自洽 | 写入 `valid_from=now+30d` 且 `time_confidence=NONE` → `valid_from` 被置 0；同条件 `DAY` → 保留 | §10.3（行 3997；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 50 | `T-T21` | `WORLD_ASOF` 历史点（有置信度的新数据） | `at<X` 命中旧值、不命中新值 | §10.4（行 4003；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 51 | `T-T22` | `BELIEF_ASOF` | 失效前命中、失效后不命中 | §10.4（行 4004；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 52 | `T-T23` | `valid_from=0` | CURRENT 可见；WORLD 以 `observed_at` 为下界；结果标 `boundary_unknown` | §10.4（行 4005；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 53 | `T-T24` | 纠正后 `WORLD_ASOF` | 旧值不再出现（`CORRECTS` 排除），`BELIEF_ASOF` 仍可见 | §10.4（行 4006；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 54 | `T-T40` | CURRENT 排除未来生效 | ① `valid_from=now+1d`、`DAY`、ACTIVE → `CURRENT` 0 命中；把 `now` 调到生效之后 → 命中。② 同值但 `time_confidence=NONE` → R0 清零 `valid_from` 且落 `CANDIDATE`，`CURRENT` 0 命中（不允许"清零后变成现在就是"） | §10.4（行 4007；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 55 | `T-T41` | 半开区间精确边界 | 旧 `valid_until=X`、新 `valid_from=X`：`at=X-1` 只旧、`at=X` 只新、`at=X+1` 只新；任何时刻都不出现两条 | §10.4（行 4008；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 56 | `T-T42` | 未知边界 → ambiguous | 迁移行（`NONE`/`valid_until=0`）在 `WORLD_ASOF` 下与后续值并存，结果 `ambiguous=true` 且每条 `boundary_unknown=true`；`BELIEF_ASOF` 同一时刻只返回一条 | §10.4（行 4009；表格行） | C1-B | 有对应、不重复、不遗漏 |
| 57 | `T-T43` | 任务到期不被起点置信度挡住 | `valid_from=0 & NONE`、`valid_until=finishedAt+7d` 的 TASK 记忆：到期前命中、到期后 0 命中 | §10.4（行 4010；表格行） | C1-B | 有对应、不重复、不遗漏 |

## 3. A-4 拆分：A-4A 开工前设计核对 / A-4B 实现后核对

**循环依赖的成因**：旧口径把 `MemorySchema.DDL_ITEM` 列为开工前核对对象，但该文件是 C1-B1 的产物，开工前不存在。

**A-4A（开工前，只核设计）**：对照 C0.2 §2.2 的 40 列与 v4.2.1 §4.2 / §4.3 的目标定义。方案明确 "11 个同名列 + 29 个新增列 = 40"，且 §4.3 声明 C0.2 §2.2–§2.6 是唯一 DDL 真相、方案不复制第二份。解析结果：C0.2 40 列与方案 11+29 的列集合**完全相等**（无缺失、无多余），逐列的类型/NOT NULL/默认值以 C0.2 为准。**A-4A 判定：PASS（设计层）。**

| 序号 | 列名 | C0.2 §2.2 定义 | 方案 v4.2.1 目标 | 结论 |
|---:|---|---|---|---|
| 1 | `id` | `id INTEGER PRIMARY KEY AUTOINCREMENT` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 2 | `memory_type` | `memory_type TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 3 | `subject` | `subject TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 4 | `subject_key` | `subject_key TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 5 | `predicate` | `predicate TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 6 | `qualifier_key` | `qualifier_key TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 7 | `value` | `value TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 8 | `content` | `content TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 9 | `normalized_content` | `normalized_content TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 10 | `dedupe_key` | `dedupe_key TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 11 | `slot_hash` | `slot_hash TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 12 | `search_text` | `search_text TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 13 | `status` | `status TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 14 | `cardinality` | `cardinality TEXT NOT NULL DEFAULT 'SINGLE'` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 15 | `confidence` | `confidence REAL NOT NULL DEFAULT 0.5` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 16 | `confidence_basis` | `confidence_basis TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 17 | `source_id` | `source_id INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 18 | `scope_type` | `scope_type TEXT NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 19 | `scope_id` | `scope_id TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 20 | `sensitivity` | `sensitivity INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 21 | `event_start` | `event_start INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 22 | `event_end` | `event_end INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 23 | `valid_from` | `valid_from INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 24 | `valid_until` | `valid_until INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 25 | `observed_at` | `observed_at INTEGER NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 26 | `recorded_at` | `recorded_at INTEGER NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 27 | `updated_at` | `updated_at INTEGER NOT NULL` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 28 | `last_confirmed_at` | `last_confirmed_at INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 29 | `invalidated_at` | `invalidated_at INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 30 | `time_confidence` | `time_confidence TEXT NOT NULL DEFAULT 'NONE'` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 31 | `superseded_by` | `superseded_by INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 32 | `revision` | `revision INTEGER NOT NULL DEFAULT 1` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 33 | `extractor` | `extractor TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 34 | `legacy_kind` | `legacy_kind TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 35 | `payload_json` | `payload_json TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 36 | `deleted_at` | `deleted_at INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 37 | `pre_delete_status` | `pre_delete_status TEXT NOT NULL DEFAULT ''` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 38 | `times_reinforced` | `times_reinforced INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 39 | `times_retrieved` | `times_retrieved INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |
| 40 | `last_used_at` | `last_used_at INTEGER NOT NULL DEFAULT 0` | §4.2（11 同名 + 29 新增 = 40 列）；§4.3 声明 C0.2 §2.2 为唯一 DDL 真相 | **一致**（设计层） |

**A-4B（B1 完成后验收，当前不得标 PASS）**：核对四方一致——C0.2 最终架构、实际 `MemorySchema.DDL_ITEM`、迁移代码建表 SQL、迁移后 v17 库的 `PRAGMA table_info` / 外键 / 约束。当前状态：**未开始**（B1 未开工，源码与 v17 库均不存在）。

## 4. A-5 拆分：A-5A 开工前设计核对 / A-5B 实现后核对

**A-5A（开工前，只核设计）**：C0.2 §2.2–§2.7 提取 15 个显式索引；v4.2.1 §4.5.1 也列 15 个（12 普通 + 3 唯一）。按"表别名归一化（item→zhuli_memory_item 等）+ 去括号/空白/反引号"比较，**15/15 的名称、目标表、唯一性、列顺序、WHERE 条件完全一致**；三个唯一索引的原始 DDL 行号（C0.2 201/260/286）也已在方案 §4.5.1 逐一引用。**A-5A 判定：PASS（设计层）。**

| 索引名 | C0.2 §2.2–§2.7 | 方案 §4.5.1 | 结论 |
|---|---|---|---|
| `idx_mi_slot` | `subject_key, predicate, qualifier_key, status ` | 第 5 项：表 `item`，普通，`(subject_key, predicate, qualifier_key, status)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mi_scope` | `scope_type, scope_id, status, memory_type ` | 第 6 项：表 `item`，普通，`(scope_type, scope_id, status, memory_type)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mi_type_time` | `memory_type, status, valid_until, event_start ` | 第 7 项：表 `item`，普通，`(memory_type, status, valid_until, event_start)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mi_observed` | `status, observed_at ` | 第 8 项：表 `item`，普通，`(status, observed_at)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mi_source` | `source_id ` | 第 9 项：表 `item`，普通，`(source_id)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mi_dedupe` | `dedupe_key WHERE status IN ('ACTIVE', 'CANDIDATE')` | 第 10 项：表 `item`，唯一（部分），`(dedupe_key) WHERE status IN ('ACTIVE','CANDIDATE')` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_ms_ref` | `source_type, ref_table, ref_id ` | 第 11 项：表 `source`，普通，`(source_type, ref_table, ref_id)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_ms_dedupe` | `source_type, ref_table, ref_id, ref_sub_id, content_hash ` | 第 12 项：表 `source`，唯一，`(source_type, ref_table, ref_id, ref_sub_id, content_hash)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mev_source` | `source_id ` | 第 13 项：表 `evidence`，普通，`(source_id)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mev_primary` | `memory_id WHERE role='PRIMARY'` | 第 14 项：表 `evidence`，唯一（部分），`(memory_id) WHERE role='PRIMARY'` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mr_from` | `from_id, relation ` | 第 15 项：表 `relation`，普通，`(from_id, relation)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mr_to` | `to_id, relation ` | 第 16 项：表 `relation`，普通，`(to_id, relation)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mrev_mem` | `memory_id, revision_no ` | 第 17 项：表 `revision`，普通，`(memory_id, revision_no)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_mrev_at` | `created_at ` | 第 18 项：表 `revision`，普通，`(created_at)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |
| `idx_me_entity` | `entity_id ` | 第 19 项：表 `entity`，普通，`(entity_id)` | **一致**（唯一性/表/列顺序/条件经归一化后逐字相同） |

**A-5B（B1 完成后验收，当前不得标 PASS）**：核对实际 `EXPECTED_INDEXES`、实际建索引 SQL、迁移后 v17 库 `sqlite_master`，要求 C0.2 / 方案 / 源码 / 真实库四方一致。当前状态：**未开始**。

## 5. A-6：真实旧数据库状态分布 —— **PASS（有真实证据）**

| 项 | 值 |
|---|---|
| 数据库 | `Temp/c01/zhuli_inbox_v16_real.db`，真机 `adb exec-out run-as lastchat.rikkafork.cocolal.dev cat databases/zhuli_inbox.db` 导出 |
| 字节数 / SHA-256 | 274,432 / `5936d12efcb1bd99f10aea15fa5d24ecd56d294825156ce630fbbec4b9f75660` |
| `user_version` | **16**（与方案基线一致） |
| 打开方式 | `mode=ro` + `PRAGMA query_only=ON`，核验前后 SHA-256 相同 |

**原始查询结果（`SELECT status, COUNT(*) AS cnt FROM zhuli_fact GROUP BY status ORDER BY status`）：**

```
ACTIVE      | 1
SUPERSEDED  | 1
```

分类统计：

| 类别 | 结果 |
|---|---|
| 总行数 | 2 |
| NULL | 0 |
| 空字符串 | 0 |
| 已知状态（ACTIVE / SUPERSEDED） | 2（各 1） |
| 未知状态（§5.3.1 三条件） | **0** |
| 大小写异常 / 首尾空格 | 0 / 0 |
| `kind` 分布 | `OTHER` × 2 |

**对照结论**：真实库只有 `ACTIVE` 与 `SUPERSEDED` 两个合法值，与 §5.3.1 映射表四状态分支完全吻合；真实库未知状态数为 **0**。T-X08 的 `unknown_status=1` 是**人工写入 `status='garbage'` 的构造用例期望**，不是真实库现状，二者不矛盾；其"未知 → CANDIDATE、不落 ACTIVE"的处理规则在方案 §5.3.1 与 §7.12 中已明确，待 B1 用合成数据验证。**A-6 判定：PASS。**

## 6. A-7：真实数据库原始证据 —— **PASS（附件已整理）**

原始查询文件 `Temp/auditA/a7_raw.json` **存在**，其数据库 SHA-256 与报告一致（`5936d12e…60fbbec4b9f75660`），`user_version=16`。本次按任务书五组 SQL 重新执行，并把**SQL 原文 + 原始输出**整理为独立附件 `C1-B_A6_A7原始数据库证据.md`（与本报告一同上传）。

| 区分项 | 数量 | 证据 |
|---|---:|---|
| 源码声明的 AUTOINCREMENT 表 | **9** | `InboxDatabase.kt:276/403/434/455/505/519/565/641/659` |
| 真实库含 AUTOINCREMENT 的表 | **9** | 附件 Q1，表名与源码逐名一致 |
| 9 个预期表实际存在 | **9** | 附件 Q6 |
| `sqlite_sequence` 存在 | 是 | 附件 Q2 |
| `sqlite_sequence` 记录行数 | **4** | 附件 Q4：cloud_log=6、zhuli_agent_step=186、zhuli_entity=2、zhuli_fact=2 |
| 同名重复行数 | **0** | 附件 Q3 空结果 |
| 真实库全部表 | 17 | 附件 Q5 |

**A-7 判定：PASS。** 证据完整、可复核，且与第二阶段报告结论一致。

## 7. 新版 A 类门槛

### 7.1 B1 开工前门槛（七项全部通过才可申请批准）

| 项 | 内容 | 当前状态 | 依据 |
|---|---|---|---|
| A-1 | 57 条架构用例映射 | ✅ PASS | 本报告 §2 |
| A-2（预核对） | 测试计划分类与统计：单跑 66 + 双跑 48×2 + 三档 1×3 = **165**；逻辑 115；XML 496；按文件与按参数化两条对账路径一致 | ✅ PASS（仅计划算术，非实际执行） | v4.2.1 §10.11 |
| A-3 | `MemoryRepo` 27 个公开成员 | ✅ PASS | 第二阶段报告，Blob `7796a176…` |
| A-4A | 40 列目标设计 | ✅ PASS | 本报告 §3 |
| A-5A | 15 索引目标设计 | ✅ PASS | 本报告 §4 |
| A-6 | 真实 v16 库状态分布 | ✅ PASS | 本报告 §5 + 证据附件 |
| A-7 | 真实 v16 库 AUTOINCREMENT | ✅ PASS | 本报告 §6 + 证据附件 |

### 7.2 B1 完成后验收门槛（保留，不得提前标 PASS）

| 项 | 内容 | 当前状态 |
|---|---|---|
| A-2（执行复核） | 实际测试执行与 Gradle XML 统计复核 | ⏳ 未开始 |
| A-4B | 源码 `DDL_ITEM` + 迁移建表 SQL + v17 库 `PRAGMA table_info` + C0.2 四方一致 | ⏳ 未开始 |
| A-5B | 源码 `EXPECTED_INDEXES` + 建索引 SQL + v17 库 `sqlite_master` + C0.2 四方一致 | ⏳ 未开始 |

## 8. 未完成项与阻塞原因

| 项 | 状态 | 原因 |
|---|---|---|
| A-2 执行复核 | 未完成 | C1-B1 未开工，测试未实现 |
| A-4B / A-5B | 未完成 | `MemorySchema.kt`、v17 数据库均不存在；这是 B1 产物，不应作为开工阻塞项 |
| A-6 未知状态运行时验证（T-X08） | 未完成 | 真实库未知状态为 0；需 B1 用合成数据验证处理规则 |
| C1-B1 开工授权 | 未获得 | 必须项目负责人审阅并明确批准；本报告不自动授权 |

## 9. 同步与验证（见最终远端回执）

- 新增：`C1-B_A类核对门槛修订与证据补齐报告.md`、`C1-B_A6_A7原始数据库证据.md`；
- 上传前确认远端 HEAD `20505375`、目标文件不存在、五个历史文件与报告 blob 未变；
- 上传后复核远端 SHA-256、父子提交关系、本次 commit 文件清单。

## 10. 最终建议：**PASS（阶段核对）**

1. **A-1 是否通过？** 是，57/57 有对应，无遗漏无重复；T-M09 三处真实条目已复核。
2. **A-3 是否通过？** 是，27 个公开成员逐项一致，Blob 未变。
3. **A-4A 是否通过？** 是，40 列目标设计一致。
4. **A-5A 是否通过？** 是，15 个索引目标设计一致。
5. **A-6 是否有真实数据库证据？** 有，v16 真机导出副本，SHA 与 user_version 已记录，原始查询结果见 §5 与附件。
6. **A-7 是否有完整原始证据？** 有，五组 SQL 原文与输出见附件 `C1-B_A6_A7原始数据库证据.md`。
7. **B1 开工前还有哪些阻塞项？** 无 A 类阻塞项；剩余 A-2 执行复核、A-4B、A-5B 均为 **B1 完成后**验收项，不构成开工阻塞。
8. **是否具备向项目负责人申请 C1-B1 开工批准的条件？** **具备申请条件**（七项开工前门槛均有证据且通过），但**本任务完成不等于开工授权**，须等待项目负责人审阅并明确批准；批准前不得开始任何 C1-B1 编码。
