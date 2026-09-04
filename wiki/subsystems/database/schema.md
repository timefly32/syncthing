# 数据库 Schema

## 主数据库 Schema

### 迁移记录表（`schema/common/10-schema.sql`）

```sql
CREATE TABLE IF NOT EXISTS schemamigrations (
    schema_version INTEGER NOT NULL PRIMARY KEY,
    applied_at INTEGER NOT NULL,
    syncthing_version TEXT NOT NULL COLLATE BINARY
) STRICT;
```

### 全局 KV（`schema/common/70-kv.sql`）

```sql
CREATE TABLE IF NOT EXISTS kv (
    key TEXT NOT NULL PRIMARY KEY COLLATE BINARY,
    value BLOB NOT NULL
) STRICT, WITHOUT ROWID;
```

### Folder 索引（`schema/main/00-folders.sql`）

```sql
CREATE TABLE IF NOT EXISTS folders (
    idx INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    folder_id TEXT NOT NULL UNIQUE COLLATE BINARY,
    database_name TEXT COLLATE BINARY
) STRICT;
```

## Per-folder Schema

### 设备索引（`schema/folder/00-devices.sql`）

```sql
CREATE TABLE IF NOT EXISTS devices (
    idx INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    device_id TEXT NOT NULL UNIQUE COLLATE BINARY
) STRICT;
```

### 文件表（`schema/folder/20-files.sql`）

```sql
CREATE TABLE IF NOT EXISTS files (
    device_idx INTEGER NOT NULL,
    sequence INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    remote_sequence INTEGER,
    name_idx INTEGER NOT NULL,
    type INTEGER NOT NULL,
    modified INTEGER NOT NULL,
    size INTEGER NOT NULL,
    version_idx INTEGER NOT NULL,
    deleted INTEGER NOT NULL,
    local_flags INTEGER NOT NULL,
    blocklist_hash BLOB,
    FOREIGN KEY(device_idx) REFERENCES devices(idx) ON DELETE CASCADE,
    FOREIGN KEY(name_idx) REFERENCES file_names(idx),
    FOREIGN KEY(version_idx) REFERENCES file_versions(idx)
) STRICT;
```

**关键设计**：
1. **名称和版本规范化**（migration v5）：`name` 和 `version` 提取到独立表，减少存储空间
2. **`sequence` 作为主键**：自增序列号，用于高效范围查询和增量同步
3. **`Global`/`Need` 标志位**：存储在 `local_flags` 中
4. **`blocklist_hash` 间接引用**：block list 存储在独立表，相同 block list 只存储一次

### IndexID 和序列号（`schema/folder/30-indexids.sql`）

```sql
CREATE TABLE IF NOT EXISTS indexids (
    device_idx INTEGER NOT NULL PRIMARY KEY,
    index_id TEXT NOT NULL COLLATE BINARY,
    sequence INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY(device_idx) REFERENCES devices(idx) ON DELETE CASCADE
) STRICT, WITHOUT ROWID;

CREATE TRIGGER IF NOT EXISTS indexids_seq AFTER INSERT ON files
BEGIN
    INSERT INTO indexids (device_idx, index_id, sequence)
        VALUES (NEW.device_idx, "", COALESCE(NEW.remote_sequence, NEW.sequence))
        ON CONFLICT DO UPDATE SET sequence = COALESCE(NEW.remote_sequence, NEW.sequence);
END;
```

### 计数表（`schema/folder/40-counts.sql`）

```sql
CREATE TABLE IF NOT EXISTS counts (
    device_idx INTEGER NOT NULL,
    type INTEGER NOT NULL,
    local_flags INTEGER NOT NULL,
    deleted INTEGER NOT NULL,
    count INTEGER NOT NULL,
    size INTEGER NOT NULL,
    PRIMARY KEY(device_idx, type, local_flags, deleted),
    FOREIGN KEY(device_idx) REFERENCES devices(idx) ON DELETE CASCADE
) STRICT, WITHOUT ROWID;
```

三个触发器（`counts_insert`/`counts_delete`/`counts_update`）自动维护计数。

### Block 存储（`schema/folder/50-blocks.sql`）

```sql
CREATE TABLE IF NOT EXISTS blocklists (
    blocklist_hash BLOB NOT NULL PRIMARY KEY,
    blprotobuf BLOB NOT NULL
) STRICT, WITHOUT ROWID;

CREATE TABLE IF NOT EXISTS blocks (
    hash BLOB NOT NULL,
    blocklist_hash BLOB NOT NULL,
    idx INTEGER NOT NULL,
    offset INTEGER NOT NULL,
    size INTEGER NOT NULL,
    PRIMARY KEY(hash, blocklist_hash, idx)
) STRICT, WITHOUT ROWID;
```

### Mtime 存储（`schema/folder/50-mtimes.sql`）

```sql
CREATE TABLE IF NOT EXISTS mtimes (
    name TEXT NOT NULL PRIMARY KEY,
    ondisk INTEGER NOT NULL,
    virtual INTEGER NOT NULL
) STRICT, WITHOUT ROWID;
```

## Schema 迁移

`currentSchemaVersion = 6`，6 个迁移文件：

| 版本 | 文件 | 内容 |
| --- | --- | --- |
| v2 | `02-remove-invalid.sql` | 移除 `invalid` 列，改用 `local_flags` 中的 `RemoteInvalid` |
| v3 | `03-drop-bad-invalid.sql` | 删除损坏的文件条目（无 blocks 的非删除文件） |
| v4 | `04-alter-blocks-tables.sql` | 重建 blocks/blocklists 表以减少索引 |
| v5 | `05-normalize-files.sql` | 名称和版本规范化（提取到独立表） |
| v6 | `06-zero-size-dirs.sql` | 目录的 size 设为 0（之前为 128） |

迁移使用 Go 模板语法（如 `{{.FlagLocalRemoteInvalid}}`）注入运行时常量。

## 关键查询

### 获取 Need 列表

```sql
SELECT f.* FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
WHERE f.device_idx = ? AND f.local_flags & ? != 0
ORDER BY f.sequence
```

### 增量索引发送

```sql
SELECT f.* FROM files f
WHERE f.device_idx = ? AND f.sequence > ?
ORDER BY f.sequence
LIMIT ?
```

### 全局版本查询

```sql
SELECT f.* FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
WHERE fn.name = ? AND f.local_flags & ? = 0
```

## 设计决策

### sequence 作为主键

**选择**：`sequence` 自增作为 `files` 表主键。
**优势**：高效范围查询（增量同步）、单调有序。
**劣势**：不能直接按 name 查询（需 JOIN file_names）。

### blocklist_hash 间接引用

**选择**：block list 通过 hash 间接引用，相同 block list 只存储一次。
**优势**：减少存储空间（大量文件可能有相同块列表）。
**劣势**：查询时需要额外 JOIN。

### 触发器维护计数

**选择**：三个触发器自动维护 counts 表。
**优势**：计数总是准确，无需手动更新。
**劣势**：每次写入触发额外 SQL，影响性能（migration v4 优化了 blocks 表索引以缓解）。
