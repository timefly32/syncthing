# Block Exchange Protocol (BEP) 设计文档

## 1. 概述

### 1.1 协议定位

Block Exchange Protocol（BEP）是 Syncthing 设备间交换文件元数据和数据块的核心协议。BEP 运行在 TLS 之上，定义了从连接握手、文件夹协商、索引交换到数据传输的完整消息体系。

### 1.2 设计目标

- **去中心化**：设备间点对点通信，无中心服务器
- **增量高效**：仅交换变更的文件元数据和需要的数据块
- **冲突感知**：基于版本向量检测并发修改
- **加密友好**：传输层 TLS + 可选端到端加密（针对不可信设备）
- **跨平台**：线格式统一（NFC 文件名、正斜杠路径）

### 1.3 版本

当前协议版本：BEP 1.0（ALPN 标识 `bep/1.0`，定义于 `lib/connections/service.go`）。

---

## 2. 协议层次

```
+-------------------------------------------+
|          BEP 消息层（本协议）              |
|  Hello / ClusterConfig / Index / Request  |
+-------------------------------------------+
|          压缩层（LZ4，可选）               |
+-------------------------------------------+
|          TLS 1.3 加密层                    |
|  (Ed25519/ECDSA 证书, ALPN bep/1.0)       |
+-------------------------------------------+
|          传输层（TCP / QUIC / Relay）      |
+-------------------------------------------+
```

BEP 本身不关心传输层细节，通过统一的 `Connection` 接口（`lib/protocol/protocol.go`）抽象 TCP、QUIC、Relay 三种连接方式。

---

## 3. 消息类型

### 3.1 消息类型枚举

定义于 `proto/bep/bep.proto:22-31`：

| 类型 ID | 名称 | 用途 |
| ---: | --- | --- |
| 0 | `CLUSTER_CONFIG` | 文件夹和设备信息交换 |
| 1 | `INDEX` | 完整文件索引（首次发送） |
| 2 | `INDEX_UPDATE` | 增量索引更新 |
| 3 | `REQUEST` | 请求文件数据块 |
| 4 | `RESPONSE` | 返回文件数据块 |
| 5 | `DOWNLOAD_PROGRESS` | 通知已下载的块 |
| 6 | `PING` | Keepalive 探测 |
| 7 | `CLOSE` | 优雅关闭连接 |

Go 侧类型映射在 `lib/protocol/protocol.go:889-933`（`typeOf` 与 `newMessage` 双向 switch）。

### 3.2 消息 Header

每条消息（Hello 除外）前缀一个 Header（`proto/bep/bep.proto:17-20`）：

```protobuf
message Header {
  MessageType type = 1;
  MessageCompression compression = 2;
}
```

压缩枚举（`bep.proto:33-36`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `COMPRESSION_NONE` | 不压缩 |
| 1 | `COMPRESSION_LZ4` | LZ4 压缩 |

### 3.3 线格式

普通消息线格式（`lib/protocol/protocol.go:577-606`、`protocol.go:787-839`）：

```
+-------------------+-------------------+-------------------+-------------------+
| 2 bytes           | hdrLen bytes      | 4 bytes           | msgLen bytes       |
| Header length     | Header protobuf   | Message length    | Message protobuf   |
| (big-endian u16)  | (type, compress)  | (big-endian u32)  | (可能 LZ4 压缩)    |
+-------------------+-------------------+-------------------+-------------------+
```

- Header 长度：2 字节 uint16，最大 65535
- Message 长度：4 字节 uint32，校验 `> MaxMessageLen`（500 MB）则拒绝
- 若压缩：Message 前 4 字节为原始长度，后接 LZ4 压缩流

---

## 4. 握手流程

### 4.1 Hello 消息

Hello 是预认证消息，**独立于 Header 体系**，使用专用魔数标识。

Protobuf 定义（`proto/bep/bep.proto:7-13`）：

```protobuf
message Hello {
  string device_name = 1;
  string client_name = 2;
  string client_version = 3;
  int32 num_connections = 4;
  int64 timestamp = 5;
}
```

线格式（`lib/protocol/bep_hello.go:119-135`）：

```
+-------------------+-------------------+-------------------+
| 4 bytes           | 2 bytes           | msgLen bytes      |
| magic             | message length    | Hello protobuf    |
| 0x2EA7D90B        | (big-endian u16)  |                   |
+-------------------+-------------------+-------------------+
```

魔数常量（`bep_hello.go:19-22`）：

| 魔数 | 含义 |
| --- | --- |
| `0x2EA7D90B` | 当前版本 Hello |
| `0x9F79BC40` | v1.3 旧版 Hello（返回 `ErrTooOldVersion`） |
| `0x00010001` / `0x00010000` | 旧版 ClusterConfig 头（返回 `ErrTooOldVersion`） |

### 4.2 Hello 交换

`ExchangeHello`（`bep_hello.go:65-73`）采用**先写后读**策略：发起方先发送自己的 Hello，再读取对端 Hello。

```
A (拨号端)                                B (监听端)
  |                                         |
  |---- TCP/TLS Connect ------------------->|
  |                                         |
  |---- Hello (magic + protobuf) ---------->|  ExchangeHello
  |<--- Hello (magic + protobuf) -----------|
  |                                         |
  |  connectionID = f(A.timestamp, B.timestamp)
```

Hello 构造（`lib/connections/service.go:304-319`）：

| 字段 | 值 |
| --- | --- |
| `ClientName` | `"syncthing"` |
| `ClientVersion` | `build.Version` |
| `Timestamp` | `time.Now().UnixNano()` |
| `DeviceName` | 配置的设备名称 |
| `NumConnections` | 当前连接数 |

### 4.3 设备 ID 验证

设备 ID = SHA-256(证书 Raw bytes)（`lib/protocol/deviceid.go:46-48`）：

```go
func NewDeviceID(rawCert []byte) DeviceID {
    return DeviceID(sha256.Sum256(rawCert))
}
```

**监听端验证**（`connections/service.go:264-280`）：
1. TLS 握手后取 `PeerCertificates`，要求恰好 1 张证书
2. `remoteID := NewDeviceID(remoteCert.Raw)`
3. 拒绝连接到自己（`remoteID == myID`）
4. 早期检查：忽略设备、连接数上限、设备暂停、网络白名单

**拨号端验证**（`connections/service.go:1167-1198` `validateIdentity`）：
1. 同样要求 1 张证书
2. `remoteID == myID` 拒绝
3. `remoteID != expectedID` 拒绝（严格校验）

### 4.4 证书名称验证

`connections/service.go:404-422`：先比较 `CommonName == certName`（默认 `syncthing`），否则用 `VerifyHostname(certName)`。

### 4.5 超时

| 阶段 | 超时 | 定义位置 |
| --- | --- | --- |
| TLS 握手 | 10 秒 | `connections/service.go:80` |
| Hello 交换 | 20 秒 | `connections/service.go:288` |

---

## 5. ClusterConfig 消息

### 5.1 结构

`proto/bep/bep.proto:42-68`：

```protobuf
message ClusterConfig {
  repeated Folder folders = 1;
  bool secondary = 2;
}

message Folder {
  string id = 1;
  string label = 2;
  FolderType type = 3;
  FolderStopReason stop_reason = 7;
  repeated Device devices = 16;
}

message Device {
  bytes id = 1;
  string name = 2;
  repeated string addresses = 3;
  Compression compression = 4;
  string cert_name = 5;
  int64 max_sequence = 6;
  bool introducer = 7;
  uint64 index_id = 8;
  bool skip_introduction_removals = 9;
  bytes encryption_password_token = 10;
}
```

Go 封装（`lib/protocol/bep_clusterconfig.go:40-171`）：

```go
type ClusterConfig struct {
    Folders   []Folder
    Secondary bool
}

type Folder struct {
    ID          string
    Label       string
    Type        FolderType
    StopReason  FolderStopReason
    Devices     []Device
}

type Device struct {
    ID                       DeviceID
    Name                     string
    Addresses                []string
    Compression              Compression
    CertName                 string
    MaxSequence              int64
    Introducer               bool
    IndexID                  IndexID
    SkipIntroductionRemovals bool
    EncryptionPasswordToken  []byte
}
```

### 5.2 枚举

**FolderType**（`bep.proto:70-75`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `SEND_RECEIVE` | 发送和接收（默认） |
| 1 | `SEND_ONLY` | 仅发送 |
| 2 | `RECEIVE_ONLY` | 仅接收 |
| 3 | `RECEIVE_ENCRYPTED` | 接收加密 |

**Compression**（`bep.proto:77-81`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `METADATA` | 仅压缩元数据（默认） |
| 1 | `NEVER` | 不压缩 |
| 2 | `ALWAYS` | 总是压缩 |

**FolderStopReason**（`bep.proto:83-86`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `RUNNING` | 正常运行 |
| 1 | `PAUSED` | 已暂停 |

### 5.3 状态机角色

`lib/protocol/protocol.go:453-464`：连接初始状态为 `stateInitial`，收到 `ClusterConfig` 后切换到 `stateReady`。**在 `stateInitial` 状态下，除 `ClusterConfig` 和 `Close` 外的任何消息都会触发协议错误并断开连接**。

```
stateInitial
    |
    | 收到 ClusterConfig
    v
stateReady
    |
    | 允许: Index, IndexUpdate, Request, Response,
    |       DownloadProgress, Ping, ClusterConfig, Close
    |
    | 收到 Close 或错误 → 关闭连接
    | 300s 无消息 → ErrTimeout 关闭
```

### 5.4 发送优先级

`protocol.go:732-746` `writerLoop`：`clusterConfigBox` 通道与 `outbox`/`closeBox` 并存，但 ClusterConfig 在循环开始时**优先单独处理**，确保握手后立即发送。

### 5.5 加密文件夹密码令牌

`Device.EncryptionPasswordToken`（`bep_clusterconfig.go:140`）通过 `PasswordToken()`（`lib/protocol/encryption.go:599-601`）生成：用 folder key 对 `knownBytes(folderID)` 做 AES-SIV 加密。`encryptedConnection.ClusterConfig`（`encryption.go:256-259`）在发送前调用 `setPasswords` 注册 folder keys。

---

## 6. 索引交换

### 6.1 Index 与 IndexUpdate

`proto/bep/bep.proto:90-101`：

```protobuf
message Index {
  string folder = 1;
  repeated FileInfo files = 2;
  int64 last_sequence = 3;
}

message IndexUpdate {
  string folder = 1;
  repeated FileInfo files = 2;
  int64 last_sequence = 3;
  int64 prev_sequence = 4;
}
```

Go 封装（`lib/protocol/bep_index_updates.go:11-78`）：

- `Index`：首次发送完整索引，携带 `last_sequence`（本批最大序列号）
- `IndexUpdate`：增量更新，多了 `prev_sequence`（上批最大序列号），形成序列号链，便于接收方检测缺失

### 6.2 FileInfo 结构

`proto/bep/bep.proto:103-167`：

```protobuf
message FileInfo {
  string name = 1;
  FileInfoType type = 2;
  int64 size = 3;
  uint32 permissions = 4;
  int64 modified_s = 5;
  Vector version = 9;
  int64 sequence = 10;
  int32 modified_ns = 11;
  uint64 modified_by = 12;
  int32 block_size = 13;
  PlatformData platform = 14;
  repeated BlockInfo blocks = 16;
  bytes symlink_target = 17;
  bytes blocks_hash = 18;
  bytes encrypted = 19;
  bytes previous_blocks_hash = 20;
}
```

**FileInfoType**（`bep.proto:169-173`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `FILE` | 常规文件 |
| 1 | `DIRECTORY` | 目录 |
| 2 | `SYMLINK` | 符号链接 |

**关键字段说明**：

| 字段 | 说明 |
| --- | --- |
| `name` | folder-relative 路径，正斜杠分隔，NFC 归一化 |
| `version` | 版本向量，记录各设备的修改计数 |
| `sequence` | 单调递增序列号，用于增量同步 |
| `blocks` | 文件块列表（仅 FILE 类型） |
| `blocks_hash` | 所有块 hash 的 SHA-256，用于快速比对 |
| `previous_blocks_hash` | 上一版本内容的 hash，用于冲突判定 |
| `encrypted` | 加密文件夹中真实 FileInfo 的密文 |
| `modified_by` | 最后修改者的 ShortID |

### 6.3 BlockInfo 结构

`proto/bep/bep.proto:175-180`：

```protobuf
message BlockInfo {
  bytes hash = 1;
  int64 offset = 2;
  int32 size = 3;
}
```

Go 封装（`lib/protocol/bep_fileinfo.go:610-614`）：

```go
type BlockInfo struct {
    Hash   []byte
    Offset int64
    Size   int
}
```

### 6.4 块大小

常量（`lib/protocol/protocol.go:46-57`）：

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `MinBlockSize` | 128 KiB | 最小块大小 |
| `MaxBlockSize` | 16 MiB | 最大块大小 |
| `MaxRequestSize` | 32 MiB | 最大请求大小（2 × MaxBlockSize） |
| `DesiredPerFileBlocks` | 2000 | 每文件期望块数 |

`BlockSizes`（`bep_fileinfo.go:93`）在 `init()` 中按 2 倍递增填充（128K → 256K → ... → 16M）。`BlockSize(fileSize)`（`bep_fileinfo.go:403-412`）选择使 `fileSize < DesiredPerFileBlocks × blockSize` 的最小块大小。

`BlockInfo.IsEmpty()`（`bep_fileinfo.go:649-654`）通过比对预计算的空块哈希判断块是否全零（稀疏文件优化）。

### 6.5 版本向量

`proto/bep/bep.proto:182-187`：

```protobuf
message Vector {
  repeated Counter counters = 1;
}

message Counter {
  uint64 id = 1;
  uint64 value = 2;
}
```

Go 封装（`lib/protocol/vector.go:24-26`）：

```go
type Vector struct { Counters []Counter }
type Counter struct { ID ShortID; Value uint64 }
```

`Counter.ID` 是 `ShortID`（`deviceid.go:28`，`uint64`，取 DeviceID 前 8 字节）。

**关键操作**：

| 操作 | 位置 | 说明 |
| --- | --- | --- |
| `Update(id)` | `vector.go:122-149` | 递增计数器，`Value = max(当前+1, now)`（防时钟回拨） |
| `Merge(b)` | `vector.go:154-184` | 取两侧各计数器的最大值 |
| `Compare(b)` | `vector.go:263-328` | 返回 `Equal/Greater/Lesser/ConcurrentLesser/ConcurrentGreater` |

**比较结果**（`vector.go:248-254`）：

| 结果 | 含义 |
| --- | --- |
| `Equal` | 两侧版本相同 |
| `Greater` | 本侧严格更新 |
| `Lesser` | 对端严格更新 |
| `ConcurrentGreater` | 并发修改，本侧优先 |
| `ConcurrentLesser` | 并发修改，对端优先 |

### 6.6 冲突解决

`bep_fileinfo.go:190-208` `InConflictWith`、`bep_fileinfo.go:212-229` `WinsConflict`：

1. 先比较 invalid 标志（invalid 者输）
2. 再比较修改时间（新者赢）
3. 最后用版本向量的 `ConcurrentGreater` 作 tie-breaker

`InConflictWith`（`bep_fileinfo.go:200-207`）：若 `PreviousBlocksHash == 对端 BlocksHash`，则非冲突（基于已知内容修改）。

### 6.7 一致性校验

`checkIndexConsistency`（`protocol.go:620-627`）对每个 FileInfo 调用 `checkFileInfoConsistency`（`protocol.go:630-657`）：

| 规则 | 错误 |
| --- | --- |
| 已删除文件不能有 blocks | `errDeletedHasBlocks` |
| 非文件类型不能有 blocks | `errNonFileHasBlocks` |
| 目录大小必须为 0 或 128 | — |
| 符号链接大小必须为 0 | — |
| 非删除、非无效的文件必须至少有 1 个 block | `errFileHasNoBlocks` |

文件名校验 `checkFilename`（`protocol.go:662-686`）：
- `path.Clean` 后不变
- 非空/非`.`/非`..`
- 非绝对路径
- 不以`../`开头

**违反任一规则都会断开连接**。

### 6.8 LocalFlags 不上线

`FileInfo.ToWire(false)`（`bep_fileinfo.go:183-186`）显式将 `LocalFlags` 置零。`LocalFlags` 仅在本地 DB 持久化时保留，不通过 BEP 传输。

---

## 7. 数据请求与响应

### 7.1 Request 消息

`proto/bep/bep.proto:189-196`：

```protobuf
message Request {
  int32 id = 1;
  string folder = 2;
  string name = 3;
  int64 offset = 4;
  int32 size = 5;
  bytes hash = 6;
  bool from_temp = 7;
}
```

### 7.2 Response 消息

`proto/bep/bep.proto:198-204`：

```protobuf
message Response {
  int32 id = 1;
  bytes data = 2;
  ErrorCode code = 3;
}
```

**ErrorCode**（`bep.proto:206-214`）：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `NO_ERROR` | 成功 |
| 1 | `NO_SUCH_FILE` | 文件不存在 |
| 2 | `INVALID_FILE` | 文件无效 |
| 3 | `TIMEOUT` | 超时 |
| 4 | `GENERIC` | 通用错误 |

Go 侧错误映射在 `lib/protocol/errors.go:17-41`。

### 7.3 请求-响应匹配

请求发起（`protocol.go:350-386`）：

```go
rc := make(chan asyncResult, 1)
c.awaitingMut.Lock()
id := c.nextID          // 自增请求 ID
c.nextID++
c.awaiting[id] = rc     // 注册等待 channel
c.awaitingMut.Unlock()
req.ID = id
c.send(ctx, req.toWire(), nil)

select {
case res := <-rc:       // 等待响应
    return res, res.err
case <-ctx.Done():      // 超时
    return nil, ctx.Err()
case <-c.closed:        // 连接关闭
    return nil, ErrClosed
}
```

响应处理（`protocol.go:709-717`）：从 `awaiting` 取出对应 channel，发送 `asyncResult{Data, codeToError(Code)}`，关闭 channel。

### 7.4 请求校验

`protocol.go:484-500` 收到 Request 后校验：

| 校验 | 拒绝条件 |
| --- | --- |
| 文件名 | `checkFilename` 失败 |
| 请求大小 | `Size < 0` |
| 请求大小 | `Size > MaxRequestSize`（32 MiB） |
| 哈希 | `len(Hash) == 0`（v1.28.1 前旧版除外） |

校验通过后**异步**处理：`go c.handleRequest(requestFromWire(msg))`（`protocol.go:500`）。

### 7.5 DownloadProgress

`proto/bep/bep.proto:236-252`：

```protobuf
message DownloadProgress {
  string folder = 1;
  repeated FileDownloadProgressUpdate updates = 2;
}

message FileDownloadProgressUpdate {
  FileDownloadProgressUpdateType update_type = 1;
  string name = 2;
  Vector version = 3;
  repeated int32 block_indexes = 4;
  int32 block_size = 5;
}
```

**UpdateType**：

| 值 | 名称 | 说明 |
| ---: | --- | --- |
| 0 | `APPEND` | 追加已下载的块索引 |
| 1 | `FORGET` | 忘记该文件的下载进度 |

用于告知对端本设备已下载了哪些块，便于协调拉取。加密连接中 `DownloadProgress` 被忽略（`encryption.go:145-154`、`encryption.go:248-254`）。

---

## 8. Ping 与 Close

### 8.1 Ping

`proto/bep/bep.proto:256`：

```protobuf
message Ping {}
```

空消息，仅用于 keepalive。

### 8.2 Close

`proto/bep/bep.proto:258-262`：

```protobuf
message Close {
  string reason = 1;
}
```

携带关闭原因字符串。

---

## 9. 连接生命周期

### 9.1 连接建立完整流程

```
A (拨号端)                                B (监听端)
  |                                         |
  |---- TCP Connect ----------------------->|
  |                                         |
  |==== TLS Handshake (ALPN bep/1.0) ======|
  |     (Ed25519/ECDSA cert)                |
  |                                         |
  |  validateIdentity (A侧)                 |  ConnectionState (B侧)
  |  NewDeviceID(peerCert.Raw)              |  NewDeviceID(peerCert.Raw)
  |  检查 == expectedID                     |  检查 != myID, connectionCheckEarly
  |                                         |
  |---- Hello (magic 0x2EA7D90B + proto) -->|  ExchangeHello
  |<--- Hello (magic 0x2EA7D90B + proto) ---|
  |                                         |
  |  connectionID = f(A.timestamp, B.timestamp)
  |                                         |
  |---- ClusterConfig --------------------->|  stateInitial → stateReady
  |<--- ClusterConfig ----------------------|
  |                                         |
  |---- Index ----------------------------->|  索引交换
  |<--- Index ------------------------------|
  |                                         |
  |---- IndexUpdate (增量) ---------------->|  持续同步
  |<--- IndexUpdate (增量) -----------------|
  |                                         |
  |---- Request (块数据) ------------------>|  数据传输
  |<--- Response (块数据) ------------------|
  |                                         |
  |---- Ping (keepalive) ------------------>|  每 45-90 秒
  |<--- Ping (keepalive) -------------------|
  |                                         |
  |---- Close (reason) -------------------->|  优雅关闭
  |                                         |
```

### 9.2 Goroutine 架构

每条连接启动 5 个 goroutine（`protocol.go:288-309`）：

| Goroutine | 位置 | 职责 |
| --- | --- | --- |
| `readerLoop` | `protocol.go:409-427` | 读消息 → `inbox` channel |
| `dispatcherLoop` | `protocol.go:429-512` | 从 `inbox` 取消息分发给 model |
| `writerLoop` | `protocol.go:732-785` | 从 `outbox`/`clusterConfigBox`/`closeBox` 取消息写出 |
| `pingSender` | `protocol.go:1019-1039` | 定期发送 Ping |
| `pingReceiver` | `protocol.go:1044-1063` | 检测读取超时 |

### 9.3 关闭流程

`Close(err)`（`protocol.go:957-978`）：

1. `sendCloseOnce` 保证只发一次 Close 消息
2. 通过 `closeBox` 发送 `&bep.Close{Reason: err.Error()}`
3. 等待 `done` 或 `CloseTimeout`（10 秒）或 `closed`
4. `go c.internalClose(err)`（异步避免 dispatcherLoop 死锁）

`internalClose`（`protocol.go:981-1012`）：

1. `closeOnce` 保证只执行一次
2. 关闭底层 `closer`
3. `close(c.closed)` 通知所有 goroutine 退出
4. 关闭所有 `awaiting` 中的 channel，使挂起的 Request 返回 `ErrClosed`
5. 等待 `dispatcherLoopStopped`
6. 调用 `c.model.Closed(err)`

收到对端 Close（`protocol.go:458-459`）：`return fmt.Errorf("closed by remote: %v", msg.Reason)`，触发 dispatcherLoop 退出 → `c.Close(err)`。

---

## 10. 加密

### 10.1 TLS 传输加密

TLS 配置（`lib/syncthing.go:263-268`）：

```go
tlsCfg := tlsutil.SecureDefaultTLS13()        // 强制 TLS 1.3
tlsCfg.Certificates = []tls.Certificate{a.cert}
tlsCfg.NextProtos = []string{"bep/1.0"}       // ALPN
tlsCfg.ClientAuth = tls.RequestClientCert
tlsCfg.SessionTicketsDisabled = true
tlsCfg.InsecureSkipVerify = true              // 自行验证 DeviceID
```

`InsecureSkipVerify=true` 是因为 Syncthing 用 DeviceID（SHA-256 of cert）而非 PKIX 验证身份。

**证书算法**：
- 同步连接：Ed25519（`tlsutil.go:112-113`）
- 浏览器兼容：ECDSA-P256

**TLS 1.2 备选**（`tlsutil.go:73-91`）：密码套件优先 ChaCha20-Poly1305。

### 10.2 端到端加密（加密文件夹）

针对不可信设备（untrusted device），Syncthing 在 TLS 之上对文件夹元数据和数据进行额外加密。

常量（`lib/protocol/encryption.go:31-42`）：

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `nonceSize` | 24 | ChaCha20-Poly1305-X nonce 大小 |
| `tagSize` | 16 | 认证标签大小 |
| `keySize` | 32 | 密钥大小 |
| `minPaddedSize` | 1024 | 最小块大小（防流量分析） |
| `blockOverhead` | 40 | tagSize + nonceSize |
| `encryptedDirExtension` | `.syncthing-enc` | 加密目录扩展名 |

**密钥派生**：

| 密钥 | 算法 | 参数 | 位置 |
| --- | --- | --- | --- |
| Folder key | scrypt | N=32768, r=8, p=1, salt=`"syncthing"+folderID` | `encryption.go:555-573` |
| File key | HKDF-SHA256 | input=folderKey+filename, salt=`"syncthing"` | `encryption.go:582-597` |

LRU 缓存：folder keys 1000 条、file keys 5000 条（`encryption.go:40-41`）。

**两种加密模式**：

| 模式 | 算法 | 用途 |
| --- | --- | --- |
| 随机 nonce | ChaCha20-Poly1305-X | FileInfo 整体加密、Response 数据加密 |
| 确定性加密 | AES-SIV | 文件名加密、block hash 加密 |

确定性加密用于文件名和 block hash，附加 offset 作为 associated data，使相同内容在不同 offset 产生不同密文，防止关联分析。

**FileInfo 加密**（`encryptFileInfo`，`encryption.go:281-369`）：

1. 整个真实 FileInfo 用随机 nonce 加密 → 存入 `Encrypted` 字段
2. 文件名用 AES-SIV 加密 + base32hex + slashify（每 200 字符插入 `/`，首段加 `.syncthing-enc`）
3. 构造**伪造的 block list**：每个 block 加 `blockOverhead`（40 字节），小于 `minPaddedSize` 的块填充到 1024
4. 伪造 version vector：`{ID:1, Value:sum(所有真实 counters)}`，使不可信设备无需冲突解决即可按序接收
5. 符号链接伪装为目录

**Request 加密**（`encryptedConnection.Request`，`encryption.go:205-246`）：

| 字段 | 加密方式 |
| --- | --- |
| `Size` | `encSize = max(req.Size, minPaddedSize) + blockOverhead` |
| `Offset` | `encOffset = req.Offset + BlockNo × blockOverhead` |
| `Name` | `encryptName(req.Name, folderKey)` |
| `Hash` | `encryptBlockHash(req.Hash, req.Offset, fileKey)` |

收到响应后 `DecryptBytes`，截取 `bs[:req.Size]`。

**加密文件夹消息流**：

```
可信设备 A                  不可信设备 U                可信设备 B
  |                            |                          |
  |== ClusterConfig ==========>| (含 EncryptionPasswordToken)
  |                            |== ClusterConfig ========>|
  |                            |                          |
  |  encryptFileInfo:          |                          |
  |   - name → AES-SIV+base32  |                          |
  |   - 整体 FileInfo → ChaCha20|                         |
  |   - blocks → 伪造(加密hash, |                          |
  |     size+40, padding 1024) |                          |
  |                            |                          |
  |==== Index (加密) =========>|==== Index (加密) ========>|
  |                            |  (U 无法解读，仅按        |
  |                            |    伪造 version 排序存储) |
  |                            |                          |
  |==== Request (加密) =======>|==== Request ============>|
  |                            |                          |  解密 name/offset/size/hash
  |                            |<=== Response ===========|
  |<=== Response (加密 data) =|  (U 加密返回，padding 1024)
  |  DecryptBytes → 截取原size |                          |
```

---

## 11. 压缩

### 11.1 压缩策略

`shouldCompressMessage`（`protocol.go:935-952`）：

| 策略 | 行为 |
| --- | --- |
| `CompressionNever` | 不压缩 |
| `CompressionAlways` | `proto.Size(msg) >= 128` 时压缩 |
| `CompressionMetadata`（默认） | 非 Response 且 `proto.Size(msg) >= 128` 时压缩 |

- `compressionThreshold = 128`（`protocol.go:63`）：小于 128 字节不压缩
- `CompressionMetadata`：**Response 永不压缩**（数据块通常是已压缩/加密的高熵数据）

压缩设置在 ClusterConfig 的 `Device.Compression` 中协商。

### 11.2 LZ4 实现

`lz4Compress`（`protocol.go:1081-1093`）：
- 用 `lz4.CompressBlock`
- 压缩结果前加 4 字节原始长度（big-endian uint32）
- 若 `n == 0`（不可压缩）返回 `errNotCompressible`

`lz4Decompress`（`protocol.go:1095-1112`）：
- 读前 4 字节原始长度，校验 `> MaxMessageLen`
- `lz4.UncompressBlock`

### 11.3 压缩收益门槛

`writeCompressedMessage`（`protocol.go:845-887`）：

```go
maxCompressed = cOverhead + len(marshaled) - len(marshaled)/32
```

**若无法节省 3.125% 带宽则放弃压缩**，回退到未压缩写入。

---

## 12. 限流与超时

### 12.1 消息大小限制

| 限制 | 值 | 定义位置 |
| --- | --- | --- |
| `MaxMessageLen` | 500 MB | `protocol.go:44` |
| `MinBlockSize` | 128 KiB | `protocol.go:46` |
| `MaxBlockSize` | 16 MiB | `protocol.go:50` |
| `MaxRequestSize` | 32 MiB | `protocol.go:54` |
| `DesiredPerFileBlocks` | 2000 | `protocol.go:57` |
| `compressionThreshold` | 128 字节 | `protocol.go:63` |
| Header 长度 | 65535（u16） | `protocol.go:583` |
| Hello 长度 | 32767（u16） | `bep_hello.go:94` |

### 12.2 超时

| 超时 | 值 | 定义位置 |
| --- | --- | --- |
| TLS 握手 | 10 秒 | `connections/service.go:80` |
| Hello 交换 | 20 秒 | `connections/service.go:288` |
| Ping 发送间隔 | 45-90 秒 | `protocol.go:208` `PingSendInterval=90s` |
| 读取超时 | 300 秒 | `protocol.go:209` `ReceiveTimeout=300s` |
| 关闭超时 | 10 秒 | `protocol.go:220` `CloseTimeout=10s` |

### 12.3 Ping/Keepalive 机制

`pingSender`（`protocol.go:1019-1039`）：
- 每 `PingSendInterval/2`（45s）检查
- 若距上次写入 < 45s，跳过
- 否则发送 Ping（空消息）
- 实际 ping 间隔在 45s ~ 90s 之间

`pingReceiver`（`protocol.go:1044-1063`）：
- 每 `ReceiveTimeout/2`（150s）检查
- 若距上次读取 > 300s，`internalClose(ErrTimeout)`

`countingReader.Last()` / `countingWriter.Last()`（`counting.go:39-41`、`counting.go:62-64`）记录最后一次 IO 的纳秒时间戳。

### 12.4 连接数限制

| 限制 | 值 | 定义位置 |
| --- | --- | --- |
| 全局最大连接数 | 128 | `connections/service.go:88` |
| 每设备并行拨号数 | 8 | `connections/service.go:87` |

### 12.5 缓冲池

`lib/protocol/bufferpool.go:18-101`：按 `BlockSizes` 分桶的 `sync.Pool`，避免大消息频繁分配。

- `Get(size)`：超过 `MaxBlockSize` 不池化
- `Put`：cap 超范围跳过

### 12.6 指标

`lib/protocol/metrics.go:14-52` 定义 6 个 Prometheus counter：

| 指标 | 说明 |
| --- | --- |
| `sent_bytes` | 发送字节数 |
| `recv_bytes` | 接收字节数 |
| `sent_messages` | 发送消息数 |
| `recv_messages` | 接收消息数 |
| `sent_uncompressed_bytes` | 发送未压缩字节数 |
| `recv_decompressed_bytes` | 接收解压后字节数 |

按 device ID 标签。`counting.go` 在每次 Read/Write 时更新。

---

## 13. 线格式归一化

`lib/protocol/wireformat.go` 实现线格式归一化：

| 规则 | 说明 |
| --- | --- |
| NFC 归一化 | 文件名转为 NFC（Normalization Form Canonical Composition） |
| 正斜杠 | 本地路径分隔符转为正斜杠 |
| folder-relative | 确保路径是 folder 相对的 |

**设计权衡**：

| 选择 | 优势 | 劣势 |
| --- | --- | --- |
| NFC（vs NFD） | 大多数系统默认形式 | macOS 使用 NFD，需要转换 |
| 正斜杠（vs 本地分隔符） | 跨平台一致 | Windows 需要转换 |
| folder-relative（vs 绝对路径） | 防止路径遍历，简化处理 | 需要确保路径不以 `/` 开头 |

---

## 14. 错误处理

### 14.1 错误类型

| 错误 | 触发条件 | 处理方式 |
| --- | --- | --- |
| `ErrClosed` | 连接已关闭 | 返回给调用方 |
| `ErrTimeout` | 300s 无读取 | `internalClose` |
| `ErrProtocolError` | 协议错误（如非法消息） | `internalClose` |
| `ErrTooOldVersion` | 旧版 Hello 魔数 | 拒绝连接 |
| `ErrUnknownMagic` | 未知 Hello 魔数 | 拒绝连接 |
| `ErrFolderMissing` | 请求的文件夹不存在 | 发送错误响应 |
| `ErrNoSuchFile` | 请求的文件不存在 | 发送错误响应 |
| `ErrInvalidFile` | 文件无效 | 发送错误响应 |

### 14.2 错误传播

```
错误发生
  |
  ├── 协议错误 → internalClose → model.Closed(err) → events.DeviceDisconnected
  ├── 请求错误 → 发送错误响应（不断开连接）
  ├── 超时     → internalClose
  └── 关闭     → internalClose
```

### 14.3 请求错误码映射

`errorToCode`（`errors.go`）：

| 错误 | ErrorCode |
| --- | --- |
| `nil` | `NO_ERROR` |
| `ErrNoSuchFile` | `NO_SUCH_FILE` |
| `ErrInvalidFile` | `INVALID_FILE` |
| `ErrTimeout` | `TIMEOUT` |
| 其他 | `GENERIC` |

---

## 15. 设计要点总结

1. **Hello 独立于 Header 体系**：使用专用魔数 `0x2EA7D90B`，便于版本协商和旧版检测。

2. **设备 ID = SHA-256(cert)**：不依赖 PKIX，`InsecureSkipVerify=true` + 自行验证。

3. **ClusterConfig 是状态机分水岭**：必须先收到 ClusterConfig 才能进入 ready 状态。

4. **LocalFlags 不上线**：`FileInfo.ToWire(false)` 显式置零，仅 DB 持久化时保留。

5. **版本向量防时钟回拨**：`Update` 用 `max(counter+1, now)`。

6. **压缩策略**：Metadata 模式下 Response 不压缩，阈值 128 字节，无法节省 3.125% 则放弃。

7. **加密层双模式**：随机 nonce（ChaCha20-Poly1305-X）用于数据，确定性（AES-SIV）用于 name/hash。

8. **5 goroutine 架构**：reader/dispatcher/writer/pingSender/pingReceiver，通过 channel 解耦。

9. **异步 Request**：`awaiting map[int]chan asyncResult` 匹配请求-响应。

10. **优雅关闭**：先发 Close 消息（10s 超时），再 internalClose 关闭底层连接并通知所有等待者。

---

## 16. 参考文件索引

| 文件 | 用途 |
| --- | --- |
| `proto/bep/bep.proto` | Protobuf 消息定义（权威源） |
| `lib/protocol/protocol.go` | 核心连接、消息循环、读写、压缩、ping |
| `lib/protocol/bep_hello.go` | Hello 握手消息 |
| `lib/protocol/bep_clusterconfig.go` | ClusterConfig / Folder / Device 结构 |
| `lib/protocol/bep_fileinfo.go` | FileInfo / BlockInfo / PlatformData 结构 |
| `lib/protocol/bep_index_updates.go` | Index / IndexUpdate 结构 |
| `lib/protocol/bep_request_response.go` | Request / Response / ErrorCode |
| `lib/protocol/bep_download_progress.go` | DownloadProgress 消息 |
| `lib/protocol/wireformat.go` | 线格式归一化（NFC、正斜杠） |
| `lib/protocol/encryption.go` | 加密文件夹的额外加密层 |
| `lib/protocol/vector.go` | 版本向量（Vector / Counter） |
| `lib/protocol/deviceid.go` | 设备 ID（SHA-256 of cert） |
| `lib/protocol/indexid.go` | IndexID 标识符 |
| `lib/protocol/bufferpool.go` | 缓冲池 |
| `lib/protocol/counting.go` | 字节计数 / keepalive 时戳 |
| `lib/protocol/errors.go` | ErrorCode ↔ error 映射 |
| `lib/protocol/metrics.go` | Prometheus 指标 |
| `lib/connections/service.go` | 连接服务（TLS 握手、Hello 交换、设备验证） |
| `lib/tlsutil/tlsutil.go` | TLS 配置工具 |
