# 符号索引

## 核心类型

### Model

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `model` | `lib/model/model.go` | 核心同步引擎 |
| `model.Start()` | `lib/model/model.go` | 启动 Model |
| `model.AddFolder()` | `lib/model/model.go` | 添加文件夹 |
| `model.Serve()` | `lib/model/model.go` | 主循环 |
| `folderRunner` | `lib/model/folder.go` | 文件夹运行器 |
| `sendReceiveFolder` | `lib/model/sender.go` | SendReceive 文件夹 |
| `roFolder` | `lib/model/rofolder.go` | SendOnly 文件夹 |
| `recvonlyFolder` | `lib/model/recvonly.go` | ReceiveOnly 文件夹 |

### Protocol

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Connection` | `lib/protocol/protocol.go` | 协议连接 |
| `rawConnection` | `lib/protocol/protocol.go` | 原始连接 |
| `NewConnection()` | `lib/protocol/protocol.go:222` | 创建连接 |
| `DeviceID` | `lib/protocol/deviceid.go` | 设备 ID |
| `ShortID` | `lib/protocol/deviceid.go` | 短设备 ID |
| `Vector` | `lib/protocol/vector.go` | 版本向量 |
| `Counter` | `lib/protocol/vector.go` | 版本计数器 |
| `FileInfo` | `lib/protocol/bep_fileinfo.go` | 文件信息 |
| `BlockInfo` | `lib/protocol/bep_fileinfo.go` | 块信息 |
| `ClusterConfig` | `lib/protocol/bep_clusterconfig.go` | 集群配置 |
| `Hello` | `lib/protocol/bep_hello.go` | 握手消息 |

### Database

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `DB` | `internal/db/interface.go` | 数据库接口 |
| `FolderDB` | `internal/db/interface.go` | 文件夹数据库接口 |
| `Typed` | `internal/db/typed.go` | 类型安全 KV |
| `baseDB` | `internal/db/sqlite/basedb.go` | 基础数据库 |
| `folderDB` | `internal/db/sqlite/folderdb_open.go` | SQLite 文件夹数据库 |

### Connections

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Service` | `lib/connections/service.go` | 连接服务 |
| `dialerFactory` | `lib/connections/structs.go` | 拨号器工厂接口 |
| `listenerFactory` | `lib/connections/structs.go` | 监听器工厂接口 |
| `limiter` | `lib/connections/limiter.go` | 限速器 |
| `dialQueue` | `lib/connections/dialqueue.go` | 拨号队列 |

### Config

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Configuration` | `lib/config/config.go` | 配置结构体 |
| `Wrapper` | `lib/config/wrapper.go` | 线程安全配置包装 |
| `FolderConfiguration` | `lib/config/folderconfig.go` | 文件夹配置 |
| `DeviceConfiguration` | `lib/config/deviceconfig.go` | 设备配置 |
| `OptionsConfiguration` | `lib/config/optionsconfig.go` | 选项配置 |
| `GUIConfiguration` | `lib/config/guiconfig.go` | GUI 配置 |
| `Subscriber` | `lib/config/wrapper.go` | 配置订阅者接口 |

### Scanner

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Config` | `lib/scanner/walk.go` | 扫描配置 |
| `Walk()` | `lib/scanner/walk.go` | 文件系统遍历 |
| `Blocks()` | `lib/scanner/blocks.go` | 块哈希计算 |
| `HashFile()` | `lib/scanner/blockqueue.go` | 并行哈希 |

### API

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `httpServer` | `lib/api/httpserver.go` | HTTP 服务器 |
| `service` | `lib/api/api.go` | API 服务 |

### Events

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Event` | `lib/events/events.go` | 事件结构体 |
| `EventType` | `lib/events/events.go` | 事件类型 |
| `Logger` | `lib/events/events.go` | 事件日志接口 |
| `BufferedSubscription` | `lib/events/buffered.go` | 缓冲订阅 |

### Filesystem

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Filesystem` | `lib/fs/fs.go` | 文件系统接口 |
| `FilesystemType` | `lib/fs/fs.go` | 文件系统类型 |
| `NewFilesystem()` | `lib/fs/mfs.go` | 创建文件系统 |

### Ignore

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Matcher` | `lib/ignore/ignore.go` | 忽略匹配器 |
| `MatchResult` | `lib/ignore/ignore.go` | 匹配结果 |

### Versioner

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `Versioner` | `lib/versioner/versioner.go` | 版本控制接口 |

## 关键常量

### Protocol

| 常量 | 位置 | 值 |
| --- | --- | --- |
| `MaxMessageLen` | `lib/protocol/protocol.go` | 500 MB |
| `MinBlockSize` | `lib/protocol/bep_fileinfo.go` | 128 KiB |
| `MaxBlockSize` | `lib/protocol/bep_fileinfo.go` | 16 MiB |
| `MaxRequestSize` | `lib/protocol/protocol.go` | 32 MiB |
| `PingSendInterval` | `lib/protocol/protocol.go` | 90 秒 |
| `ReceiveTimeout` | `lib/protocol/protocol.go` | 300 秒 |
| `HelloMessageMagic` | `lib/protocol/bep_hello.go` | `0x2EA7D90B` |
| `DeviceIDLength` | `lib/protocol/deviceid.go` | 32 |

### Connections

| 常量 | 位置 | 值 |
| --- | --- | --- |
| `tlsHandshakeTimeout` | `lib/connections/service.go` | 10 秒 |
| `stdConnectionLoopSleep` | `lib/connections/service.go` | 60 秒 |
| `dialMaxParallel` | `lib/connections/service.go` | 64 |
| `maxNumConnections` | `lib/connections/service.go` | 128 |

### Database

| 常量 | 位置 | 值 |
| --- | --- | --- |
| `currentSchemaVersion` | `internal/db/sqlite/` | 6 |
| `maxOpenConns` | `internal/db/sqlite/db_open.go` | 8 |
| `maxIdleConns` | `internal/db/sqlite/db_open.go` | 4 |

## 关键接口

### Model 回调

```go
type Model interface {
    ClusterConfig(deviceID DeviceID, config ClusterConfig)
    Index(deviceID DeviceID, folder string, files []FileInfo)
    IndexUpdate(deviceID DeviceID, folder string, files []FileInfo)
    Request(deviceID DeviceID, folder string, name string, ...) (ResponseMessage, error)
    DownloadProgress(deviceID DeviceID, folder string, updates []FileDownloadProgressUpdate)
    Closed(err error)
}
```

### Filesystem

```go
type Filesystem interface {
    Chmod(name string, mode FileMode) error
    Create(name string) (File, error)
    Open(name string) (File, error)
    Stat(name string) (FileInfo, error)
    Walk(name string, walkFn WalkFunc) error
    ...
}
```

### Versioner

```go
type Versioner interface {
    Archive(filePath string) error
    GetVersions(filePath string) ([]FileVersion, error)
    Restore(filePath string, version time.Time) error
}
```
