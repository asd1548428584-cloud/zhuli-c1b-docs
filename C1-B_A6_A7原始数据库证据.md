# C1-B A-6 / A-7 原始数据库证据（只读）

- 数据库：`Temp/c01/zhuli_inbox_v16_real.db`（真机 `adb exec-out run-as` 导出副本）
- 文件字节数：274432
- SHA-256：`5936d12efcb1bd99f10aea15fa5d24ecd56d294825156ce630fbbec4b9f75660`
- `PRAGMA user_version`：**16**
- `PRAGMA journal_mode`：`delete`
- `PRAGMA integrity_check`：`ok`
- 打开方式：`mode=ro` + `PRAGMA query_only=ON`，核验前后 SHA-256 相同。

## Q1 AUTOINCREMENT 表

```sql
SELECT name, sql FROM sqlite_master WHERE type = 'table' AND sql LIKE '%AUTOINCREMENT%' ORDER BY name
```

原始输出：

```
chat_log | CREATE TABLE chat_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                role TEXT NOT NULL,
                content TEXT NOT NULL,
                created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')*1000)
            )
cloud_log | CREATE TABLE cloud_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                at INTEGER NOT NULL,
                task TEXT NOT NULL DEFAULT '',
                host TEXT NOT NULL DEFAULT '',
                model TEXT NOT NULL DEFAULT '',
                request_redacted TEXT NOT NULL DEFAULT '',
                ok INTEGER NOT NULL DEFAULT 1,
                note TEXT NOT NULL DEFAULT ''
            )
inbox | CREATE TABLE inbox (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                source TEXT NOT NULL,
                package_name TEXT NOT NULL,
                app_label TEXT NOT NULL DEFAULT '',
                title TEXT NOT NULL DEFAULT '',
                text TEXT NOT NULL DEFAULT '',
                received_at INTEGER NOT NULL DEFAULT (strftime('%s','now')*1000),
                pending_extract INTEGER NOT NULL DEFAULT 1,
                url TEXT NOT NULL DEFAULT '',
                summary TEXT NOT NULL DEFAULT '',
                category TEXT NOT NULL DEFAULT '',
                deadline_at INTEGER NOT NULL DEFAULT 0,
                amount REAL NOT NULL DEFAULT 0,
                note TEXT NOT NULL DEFAULT '',
                extracted_at INTEGER NOT NULL DEFAULT 0,
                media_path TEXT NOT NULL DEFAULT '',
                repeat_rule TEXT NOT NULL DEFAULT ''
            )
ledger | CREATE TABLE ledger (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                inbox_id INTEGER NOT NULL,
                dir TEXT NOT NULL DEFAULT '支',
                category TEXT NOT NULL DEFAULT '其他',
                amount REAL NOT NULL DEFAULT 0,
                occurred_at INTEGER NOT NULL,
                note TEXT NOT NULL DEFAULT ''
            )
reminders | CREATE TABLE reminders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                inbox_id INTEGER NOT NULL,
                due_at INTEGER NOT NULL,
                title TEXT NOT NULL DEFAULT '',
                body TEXT NOT NULL DEFAULT '',
                status TEXT NOT NULL DEFAULT 'scheduled',
                fired_at INTEGER NOT NULL DEFAULT 0,
                calendar_event_id INTEGER NOT NULL DEFAULT 0,
                notified INTEGER NOT NULL DEFAULT 0,
                repeat_rule TEXT NOT NULL DEFAULT '',
                snooze_count INTEGER NOT NULL DEFAULT 0,
                confidence TEXT NOT NULL DEFAULT 'high',
                basis TEXT NOT NULL DEFAULT ''
            )
skills | CREATE TABLE skills (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                kind TEXT NOT NULL,
                pattern TEXT NOT NULL,
                arg TEXT NOT NULL DEFAULT '',
                enabled INTEGER NOT NULL DEFAULT 1,
                hits INTEGER NOT NULL DEFAULT 0,
                created_at INTEGER NOT NULL
            )
zhuli_agent_step | CREATE TABLE zhuli_agent_step (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                task_id TEXT NOT NULL,
                seq INTEGER NOT NULL,
                kind TEXT NOT NULL,
                content TEXT NOT NULL DEFAULT '',
                tool_name TEXT NOT NULL DEFAULT '',
                tool_args TEXT NOT NULL DEFAULT '',
                tool_call_id TEXT NOT NULL DEFAULT '',
                calls_json TEXT NOT NULL DEFAULT '',
                ok INTEGER NOT NULL DEFAULT 1,
                at INTEGER NOT NULL,
                latency_ms INTEGER NOT NULL DEFAULT 0,
                provider_id TEXT NOT NULL DEFAULT '',
                UNIQUE(task_id, seq)
            )
zhuli_entity | CREATE TABLE zhuli_entity (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                kind TEXT NOT NULL,
                name TEXT NOT NULL,
                norm TEXT NOT NULL,
                aliases TEXT NOT NULL DEFAULT '',
                created_at INTEGER NOT NULL,
                updated_at INTEGER NOT NULL,
                UNIQUE(kind, norm)
            )
zhuli_fact | CREATE TABLE zhuli_fact (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                subject TEXT NOT NULL,
                predicate TEXT NOT NULL,
                value TEXT NOT NULL,
                kind TEXT NOT NULL DEFAULT 'OTHER',
                confidence REAL NOT NULL DEFAULT 0.6,
                source_type TEXT NOT NULL DEFAULT 'INBOX',
                source_id INTEGER NOT NULL DEFAULT 0,
                source_text TEXT NOT NULL DEFAULT '',
                observed_at INTEGER NOT NULL DEFAULT 0,
                valid_from INTEGER NOT NULL DEFAULT 0,
                valid_to INTEGER NOT NULL DEFAULT 0,
                status TEXT NOT NULL DEFAULT 'ACTIVE',
                superseded_by INTEGER NOT NULL DEFAULT 0,
                created_at INTEGER NOT NULL,
                updated_at INTEGER NOT NULL
            )
```

## Q2 sqlite_sequence 存在性

```sql
SELECT name, sql FROM sqlite_master WHERE type = 'table' AND name = 'sqlite_sequence'
```

原始输出：

```
sqlite_sequence | CREATE TABLE sqlite_sequence(name,seq)
```

## Q3 sequence 同名重复

```sql
SELECT name, COUNT(*) AS cnt FROM sqlite_sequence GROUP BY name HAVING COUNT(*) > 1
```

原始输出：

```
```

## Q4 sequence 全量

```sql
SELECT name, seq FROM sqlite_sequence ORDER BY name
```

原始输出：

```
cloud_log | 6
zhuli_agent_step | 186
zhuli_entity | 2
zhuli_fact | 2
```

## Q5 全部表

```sql
SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name
```

原始输出：

```
android_metadata
chat_log
cloud_log
embeddings
inbox
ledger
reminders
skills
source_rule
sqlite_sequence
zhuli_agent_step
zhuli_agent_task
zhuli_entity
zhuli_fact
zhuli_fact_vec
zhuli_tool_exec
zhuli_vec_queue
```

## Q6 9预期表存在性

```sql
SELECT name FROM sqlite_master WHERE type='table' AND name IN ('inbox','reminders','ledger','chat_log','zhuli_entity','zhuli_fact','zhuli_agent_step','skills','cloud_log') ORDER BY name
```

原始输出：

```
chat_log
cloud_log
inbox
ledger
reminders
skills
zhuli_agent_step
zhuli_entity
zhuli_fact
```

## A-6 状态分布原始输出

```sql
SELECT status, COUNT(*) AS cnt FROM zhuli_fact GROUP BY status ORDER BY status;
```

```
ACTIVE | 1
SUPERSEDED | 1
```

- NULL 行数：0
- 空字符串行数：0
- 总行数：2

未知状态（§5.3.1 三条件 `IS NULL / TRIM NOT IN / TRIM=''`）：
```
(0 行)
```

