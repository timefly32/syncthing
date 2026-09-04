# 事件总线深入

## 事件数据结构

### DeviceConnected

```go
map[string]interface{}{
    "id":        deviceID.String(),
    "addr":      addr,
    "name":      deviceName,
    "inbps":     inBytesPerSec,
    "outbps":    outBytesPerSec,
    "paused":    paused,
    "type":      connType,
    "crypto":    crypto,
    "indexID":   indexID,
}
```

### FolderCompletion

```go
map[string]interface{}{
    "device":      deviceID.String(),
    "folder":      folderID,
    "completion":  completionPct,
    "globalBytes": globalBytes,
    "needBytes":   needBytes,
    "needItems":   needItems,
}
```

### ItemFinished

```go
map[string]interface{}{
    "folder":  folderID,
    "item":    itemName,
    "error":   errString,
    "action":  "update"|"delete"|"metadata",
    "type":    "file"|"dir"|"symlink",
    "modified": modifiedTime,
}
```

## 事件订阅掩码

订阅者可指定感兴趣的事件类型掩码：

```go
mask := events.FolderScanProgress | events.ItemFinished
sub := evLogger.Subscribe(mask, 1000)
```

掩码过滤在 `BufferedSubscription` 中完成，减少不必要的事件处理。

## 事件顺序保证

- 单个订阅者内事件按发布顺序
- 多订阅者间无顺序保证
- 事件时间戳为发布时间

## 缓冲溢出处理

当订阅者缓冲满时：

1. 丢弃最旧的事件
2. 记录 `events.EventBufferOverflow` 警告
3. 发布者不被阻塞

## API 事件流

`/rest/events` 端点：

```
GET /rest/events?since=123&limit=100
```

- `since`：返回此 ID 之后的事件
- `limit`：最大返回数量
- 长轮询：阻塞直到有事件或超时

## 磁盘事件流

`/rest/events/disk` 端点读取磁盘日志文件，用于崩溃后恢复。
