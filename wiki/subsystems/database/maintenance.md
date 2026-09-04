# 数据库维护深入

## GC（垃圾回收）

`internal/db/sqlite/db_service.go`：

### 文件夹 GC

```go
func (db *DB) Compact() error {
    // 1. 遍历所有文件夹
    for _, folder := range db.ListFolders() {
        folderDB := db.GetFolderDB(folder)
        // 2. 清理已删除设备的文件
        folderDB.Compact()
    }
    return nil
}
```

### folderDB.Compact

```go
func (db *folderDB) Compact() error {
    return db.Update(func(tx Transaction) error {
        // 1. 删除已删除设备的文件
        tx.Exec("DELETE FROM files WHERE device_idx NOT IN (SELECT idx FROM devices)")
        // 2. 清理孤立的 blocklists
        tx.Exec("DELETE FROM blocklists WHERE blocklist_hash NOT IN (SELECT DISTINCT blocklist_hash FROM files)")
        // 3. 清理孤立的 blocks
        tx.Exec("DELETE FROM blocks WHERE blocklist_hash NOT IN (SELECT blocklist_hash FROM blocklists)")
        // 4. 清理孤立的 file_names
        tx.Exec("DELETE FROM file_names WHERE idx NOT IN (SELECT DISTINCT name_idx FROM files)")
        // 5. 清理孤立的 file_versions
        tx.Exec("DELETE FROM file_versions WHERE idx NOT IN (SELECT DISTINCT version_idx FROM files)")
        // 6. 清理孤立的 mtimes
        tx.Exec("DELETE FROM mtimes WHERE name NOT IN (SELECT name FROM file_names)")
        return nil
    })
}
```

## 设备删除

```go
func (db *folderDB) DropDevice(deviceID protocol.DeviceID) error {
    return db.Update(func(tx Transaction) error {
        // 1. 查找设备 idx
        var deviceIdx int
        tx.Get(&deviceIdx, "SELECT idx FROM devices WHERE device_id = ?", deviceID.String())
        // 2. 删除设备的文件（触发器自动更新 counts）
        tx.Exec("DELETE FROM files WHERE device_idx = ?", deviceIdx)
        // 3. 删除设备的 indexid
        tx.Exec("DELETE FROM indexids WHERE device_idx = ?", deviceIdx)
        // 4. 删除设备记录
        tx.Exec("DELETE FROM devices WHERE idx = ?", deviceIdx)
        return nil
    })
}
```

## 文件夹删除

```go
func (db *DB) DropFolder(folder string) error {
    // 1. 获取 folder DB 路径
    folderDB := db.GetFolderDB(folder)
    folderDB.Close()
    // 2. 删除 SQLite 文件
    os.Remove(folderDB.Path())
    os.Remove(folderDB.Path() + "-wal")
    os.Remove(folderDB.Path() + "-shm")
    // 3. 从主数据库删除 folder 记录
    db.Update(func(tx Transaction) error {
        tx.Exec("DELETE FROM folders WHERE folder_id = ?", folder)
        return nil
    })
    return nil
}
```

## ObservedDB

`internal/db/observed.go`：

### 待处理设备

```go
type ObservedDevice struct {
    ID    protocol.DeviceID
    Name  string
    Time  time.Time
}

func (db *ObservedDB) AddPendingDevice(deviceID protocol.DeviceID, name string) error {
    return db.db.PutKV(fmt.Sprintf("pendingDevice:%s", deviceID.String()), ...)
}

func (db *ObservedDB) RemovePendingDevice(deviceID protocol.DeviceID) error {
    return db.db.DeleteKV(fmt.Sprintf("pendingDevice:%s", deviceID.String()))
}
```

### 待处理文件夹

```go
type ObservedFolder struct {
    ID       string
    DeviceID protocol.DeviceID
    Time     time.Time
}

func (db *ObservedDB) AddPendingFolder(folder string, deviceID protocol.DeviceID) error {
    return db.db.PutKV(fmt.Sprintf("pendingFolder:%s:%s", deviceID.String(), folder), ...)
}
```

## 指标

`internal/db/metrics.go`：

```go
var (
    metricTotalQueries = promauto.NewCounterVec(...)
    metricQueryDuration = promauto.NewHistogramVec(...)
)
```

### 查询计数

```go
func (db *DB) withMetrics(op string, fn func() error) error {
    start := time.Now()
    err := fn()
    metricTotalQueries.WithLabelValues(op).Inc()
    metricQueryDuration.WithLabelValues(op).Observe(time.Since(start).Seconds())
    return err
}
```

## 维护任务

### 定期维护

```go
func (db *DBService) Start() {
    go func() {
        ticker := time.NewTicker(maintenanceInterval)
        for {
            select {
            case <-ticker.C:
                db.maintain()
            case <-db.ctx.Done():
                return
            }
        }
    }()
}

func (db *DBService) maintain() {
    // 1. 检查数据库大小
    // 2. 触发 GC 如需
    // 3. 检查 WAL 大小
    // 4. 触发 checkpoint 如需
}
```

## 备份

### 在线备份

```bash
# SQLite 在线备份
sqlite3 db.sqlite ".backup '/backup/db.sqlite'"
```

### 离线备份

```bash
# 停止 Syncthing
systemctl stop syncthing
# 复制数据库
cp -r index-v0.15.0/ /backup/
# 启动 Syncthing
systemctl start syncthing
```

## 恢复

### 从备份恢复

```bash
# 停止 Syncthing
systemctl stop syncthing
# 恢复数据库
cp -r /backup/index-v0.15.0/ ~/.local/state/syncthing/
# 启动 Syncthing
systemctl start syncthing
```

### 重建数据库

```bash
# 停止 Syncthing
systemctl stop syncthing
# 删除数据库
rm -rf ~/.local/state/syncthing/index-v0.15.0/
# 启动 Syncthing（会重新扫描）
systemctl start syncthing
```

## 测试

`internal/db/sqlite/db_service_test.go`：

- GC 测试
- 设备删除测试
- 文件夹删除测试
- 维护任务测试
