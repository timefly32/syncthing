# 数据库事务深入

## 事务模型

`internal/db/sqlite/` 使用 SQLite 事务：

### 读事务

```go
func (db *DB) Read(fn func(tx ReadTransaction) error) error {
    db.readMu.Lock()
    defer db.readMu.Unlock()
    
    tx, err := db.readDb.Beginx()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    return fn(&readTransaction{tx})
}
```

### 写事务

```go
func (db *DB) Update(fn func(tx Transaction) error) error {
    db.writeMu.Lock()
    defer db.writeMu.Unlock()
    
    tx, err := db.writeDb.Beginx()
    if err != nil {
        return err
    }
    
    if err := fn(&writeTransaction{tx}); err != nil {
        tx.Rollback()
        return err
    }
    
    return tx.Commit()
}
```

## 读写分离

SQLite WAL 模式支持读写并发：

- **读连接**：从 WAL 读取，不阻塞写
- **写连接**：串行写入，更新 WAL

```go
type DB struct {
    readDb  *sqlx.DB  // 读连接池
    writeDb *sqlx.DB  // 写连接（单连接）
    ...
}
```

## 预处理语句缓存

```go
type txPreparedStmts struct {
    *sqlx.Tx
    stmts map[string]*sqlx.Stmt
}

func (t *txPreparedStmts) Prepare(query string) (*sqlx.Stmt, error) {
    if stmt, ok := t.stmts[query]; ok {
        return stmt, nil
    }
    stmt, err := t.Tx.Preparex(query)
    if err != nil {
        return nil, err
    }
    t.stmts[query] = stmt
    return stmt, nil
}

func (t *txPreparedStmts) Commit() error {
    defer t.closeStmts()
    return t.Tx.Commit()
}

func (t *txPreparedStmts) closeStmts() {
    for _, stmt := range t.stmts {
        stmt.Close()
    }
}
```

## 批量更新

`folderDB.Update` 支持批量文件更新：

```go
func (db *folderDB) Update(fn func(tx Transaction) error) error {
    db.writeMu.Lock()
    defer db.writeMu.Unlock()
    
    tx, _ := db.db.Beginx()
    prepared := &txPreparedStmts{Tx: tx, stmts: make(map[string]*sqlx.Stmt)}
    
    if err := fn(&writeTransaction{prepared}); err != nil {
        prepared.Rollback()
        return err
    }
    
    if err := prepared.Commit(); err != nil {
        return err
    }
    
    // 检查 WAL checkpoint
    if db.shouldCheckpoint() {
        db.checkpoint()
    }
    
    return nil
}
```

## WAL Checkpoint

```go
func (db *folderDB) shouldCheckpoint() bool {
    // 检查 WAL 文件大小
    info, _ := os.Stat(db.dbPath + "-wal")
    return info.Size() > walCheckpointThreshold
}

func (db *folderDB) checkpoint() {
    db.db.Exec("PRAGMA wal_checkpoint(PASSIVE)")
}
```

### Checkpoint 模式

| 模式 | 说明 |
| --- | --- |
| `PASSIVE` | 不阻塞读写，尽可能 checkpoint |
| `FULL` | 等待读完成，checkpoint 所有帧 |
| `RESTART` | 像 FULL，但重启 WAL |

## 事务隔离

SQLite 默认隔离级别：

- **读**：快照隔离（WAL 模式）
- **写**：串行化

## 死锁防护

- 单写连接，无写-写死锁
- 读不阻塞写，写不阻塞读（WAL）
- 应用层锁保护跨 folder 操作

## 性能优化

### 批量插入

```go
func (tx *writeTransaction) UpdateRemoteFiles(deviceID protocol.DeviceID, files []protocol.FileInfo) error {
    stmt := tx.Prepare("INSERT OR REPLACE INTO files ...")
    for _, file := range files {
        stmt.Exec(file...)
    }
    return nil
}
```

### 事务大小

- 大事务占用写锁时间长
- 小事务增加提交开销
- 平衡：每 1000 个文件提交一次

### 索引维护

- `WITHOUT ROWID` 表主键即聚簇索引
- 触发器自动维护计数
- 预处理语句减少编译开销

## 错误处理

### 事务回滚

```go
if err := fn(&writeTransaction{prepared}); err != nil {
    prepared.Rollback()
    return err
}
```

### 连接断开

- 连接池自动重连
- 事务自动回滚
- 应用层重试

## 测试

`internal/db/sqlite/db_test.go`：

- 事务提交/回滚测试
- 并发读写测试
- 批量更新测试
