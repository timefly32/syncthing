# 数据库查询深入

## 文件查询

### 按名称查询

```sql
SELECT f.*, fn.name, v.version
FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
JOIN file_versions v ON f.version_idx = v.idx
WHERE f.device_idx = ? AND fn.name = ?
```

### 按序列号范围（增量同步）

```sql
SELECT f.*, fn.name, v.version
FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
JOIN file_versions v ON f.version_idx = v.idx
WHERE f.device_idx = ? AND f.sequence > ?
ORDER BY f.sequence
LIMIT ?
```

### Need 列表

```sql
SELECT f.*, fn.name, v.version
FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
JOIN file_versions v ON f.version_idx = v.idx
WHERE f.device_idx = ? AND (f.local_flags & ?) != 0
ORDER BY f.sequence
```

`local_flags` 中的 `FlagLocalNeeded` 位标识需要同步的文件。

## 全局版本查询

### 全局文件

```sql
-- 获取每个文件的最新全局版本
SELECT fn.name, f.*, v.version
FROM files f
JOIN file_names fn ON f.name_idx = fn.idx
JOIN file_versions v ON f.version_idx = v.idx
WHERE f.device_idx = ?  -- GlobalDeviceID
```

### 全局版本合并

```sql
-- 获取某文件所有设备的版本
SELECT f.*, d.device_id, v.version
FROM files f
JOIN devices d ON f.device_idx = d.idx
JOIN file_names fn ON f.name_idx = fn.idx
JOIN file_versions v ON f.version_idx = v.idx
WHERE fn.name = ?
```

## 计数查询

### 文件夹统计

```sql
SELECT
    type,
    deleted,
    SUM(count) as count,
    SUM(size) as size
FROM counts
GROUP BY type, deleted
```

### 设备统计

```sql
SELECT
    d.device_id,
    c.type,
    c.deleted,
    c.count,
    c.size
FROM counts c
JOIN devices d ON c.device_idx = d.idx
WHERE d.device_id = ?
```

## Block 查询

### 按 hash 查找文件

```sql
SELECT DISTINCT fn.name
FROM blocks b
JOIN blocklists bl ON b.blocklist_hash = bl.blocklist_hash
JOIN files f ON f.blocklist_hash = bl.blocklist_hash
JOIN file_names fn ON f.name_idx = fn.idx
WHERE b.hash = ?
```

### 获取文件的块列表

```sql
SELECT bl.blprotobuf
FROM blocklists bl
WHERE bl.blocklist_hash = ?
```

## Mtime 查询

```sql
SELECT ondisk, virtual
FROM mtimes
WHERE name = ?
```

## IndexID 查询

```sql
SELECT index_id, sequence
FROM indexids
WHERE device_idx = ?
```

## KV 查询

```sql
SELECT value FROM kv WHERE key = ?
```

## 事务模式

### 读事务

```go
db.Read(func(tx ReadTransaction) error {
    // 只读操作
    return nil
})
```

### 写事务

```go
db.Update(func(tx Transaction) error {
    // 读写操作
    return nil
})
```

### 批量更新

`folderDB.Update` 支持批量文件更新：

1. 开始事务
2. 批量插入/更新文件
3. 触发器自动更新计数
4. 提交事务
5. 检查 WAL checkpoint

## 性能优化

### 预处理语句

```go
stmt := tx.Prepare("SELECT ...")
defer stmt.Close()
```

`txPreparedStmts` 缓存预处理语句，避免重复编译。

### 索引

关键索引（在 schema 文件中定义）：

- `files(device_idx, sequence)` — 增量同步
- `files(device_idx, name_idx)` — 按名称查询
- `blocks(hash)` — 按 hash 查找
- `blocklists(blocklist_hash)` — 块列表查找

### WITHOUT ROWID

`counts`、`indexids`、`mtimes`、`blocklists`、`blocks` 表使用 `WITHOUT ROWID`：

- 减少存储空间
- 提高查询性能
- 主键即聚簇索引

## WAL Checkpoint

`folderdb_update.go`：

```go
if shouldCheckpoint {
    db.Exec("PRAGMA wal_checkpoint(PASSIVE)")
}
```

- `PASSIVE`：不阻塞读写
- `FULL`：等待读完成
- `RESTART`：重启 WAL

触发条件：WAL 文件大小超过阈值。
