# 统计模块

## 概述

`lib/stats/` 实现 Syncthing 的统计信息收集，约 400 行 Go 代码。记录设备连接、文件夹同步等统计信息。

## 职责

- **设备统计**：连接时间、数据传输量
- **文件夹统计**：同步状态、文件数量
- **持久化**：统计信息存储在数据库

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `stats.go` | ~200 | 统计接口和实现 |
| `device.go` | ~100 | 设备统计 |
| `folder.go` | ~100 | 文件夹统计 |

## 设备统计

```go
type DeviceStatistics struct {
    LastSeen time.Time
    LastConnectionDuration time.Duration
    TotalBytesSent int64
    TotalBytesReceived int64
}
```

### 记录连接

```go
func (s *Statistics) RegisterDeviceConnection(deviceID protocol.DeviceID, duration time.Duration) {
    // 更新 LastSeen
    // 更新 LastConnectionDuration
}
```

### 记录传输

```go
func (s *Statistics) RegisterDeviceTransfer(deviceID protocol.DeviceID, sent, received int64) {
    // 累加 TotalBytesSent/Received
}
```

## 文件夹统计

```go
type FolderStatistics struct {
    LastScan time.Time
    LastSync time.Time
    LastFileCount int
    LastFileSize int64
}
```

### 记录扫描

```go
func (s *Statistics) RegisterFolderScan(folder string, fileCount int, fileSize int64) {
    // 更新 LastScan
    // 更新 LastFileCount/Size
}
```

### 记录同步

```go
func (s *Statistics) RegisterFolderSync(folder string) {
    // 更新 LastSync
}
```

## 持久化

统计信息存储在数据库 KV 中：

- 键：`stats.device.<deviceID>`、`stats.folder.<folderID>`
- 值：JSON 序列化的统计结构

## API

`/rest/stats/device`、`/rest/stats/folder`：

```json
{
    "DEVICE_ID": {
        "lastSeen": "2024-01-01T00:00:00Z",
        "totalBytesSent": 123456789,
        "totalBytesReceived": 987654321
    }
}
```

## 测试

`stats_test.go`：

- 统计记录测试
- 持久化测试
