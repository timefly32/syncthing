# 术语表

## A

### API Key
Web UI REST API 的认证密钥，配置在 `GUI.APIKey`。

### Auto-Normalize
文件夹路径自动归一化功能，确保路径格式一致。

## B

### BEP (Block Exchange Protocol)
Syncthing 的核心协议，定义设备间如何交换文件元数据和数据块。见 `lib/protocol/`。

### Block
文件被分割的固定大小块（默认 128KB），用于增量同步和哈希校验。

### BlockHash
文件块的 SHA-256 哈希，用于标识块内容和去重。

### BlockSize
文件块的大小，默认 128KB，可配置为动态（大块模式）。

## C

### ClusterConfig
BEP 协议消息，交换文件夹和设备配置信息，必须是连接后的第一条消息。

### Completion
文件夹同步完成度，表示已同步的文件比例。

### Concurrent
版本向量比较结果之一，表示两个版本并发修改，需要冲突解决。

### Connection ID
每个连接的唯一标识，基于双方时间戳之和 + 随机数生成。

### Counter
版本向量中的元素，包含设备 ShortID 和修改计数。

## D

### Device ID
设备的唯一标识，等于设备 TLS 证书的 SHA-256，32 字节，显示为 56 字符的 base32 编码（含 Luhn 校验位）。

### DeviceConnectionTracker
跟踪每设备连接数和期望连接数的组件。

### Dialer
拨号器，负责主动连接其他设备。

### Discovery
设备发现机制，包括本地发现（UDP 广播）和全局发现（Discovery Server）。

### DownloadProgress
BEP 协议消息，通知对端正在下载的块。

## F

### FileInfo
文件的元数据，包含名称、大小、修改时间、版本向量、块列表等。

### Folder
同步文件夹，由唯一 ID 标识。

### FolderType
文件夹类型：SendReceive、SendOnly、ReceiveOnly、ReceiveEncrypted。

### FSWatcher
文件系统监视器，实时检测文件变更，减少全量扫描需求。

## G

### GlobalDeviceID
全局聚合设备 ID（`0xf8` 重复 32 次），用于表示全局状态。

### Global Version
文件的全局版本，合并所有设备的版本向量。

## H

### Hello
BEP 协议握手消息，交换协议版本和设备信息。

## I

### Index
BEP 协议消息，发送全量文件索引。

### IndexUpdate
BEP 协议消息，发送增量文件索引更新。

### IndexID
每个设备对每个文件夹的索引标识，用于增量同步。

### InConflictWith
文件冲突判定，基于版本向量和块哈希。

## L

### LocalDeviceID
本地设备 ID（`0xff` 重复 32 次），表示本地虚拟设备。

### LocalFlags
文件元数据的内部标志位，不上 wire 传输。

### Luhn-32
Syncthing 自定义的 Luhn 校验变体，用于设备 ID 的校验位。

## M

### Model
核心同步引擎，管理文件夹状态、协调扫描和同步。见 `lib/model/`。

### Mtime
文件修改时间（modification time）。

## N

### NAT
网络地址转换，Syncthing 通过 UPnP/PMP/PCP 进行 NAT 穿透。

### Need
文件需要同步的标志，存储在 `local_flags` 中。

## O

### Outbox
协议连接的出站消息 channel，无缓冲，形成背压。

## P

### Ping
BEP 协议心跳消息，用于检测连接活性。

### Protocol
Syncthing 的协议层，实现 BEP。见 `lib/protocol/`。

## Q

### QUIC
基于 UDP 的多路复用传输协议，Syncthing 支持作为 TCP 的替代。

## R

### ReceiveEncrypted
文件夹类型，接收加密数据，用于不可信设备。

### Relay
中继服务器，在设备无法直接连接时转发流量。

### Request
BEP 协议消息，请求文件块数据。

### Response
BEP 协议消息，响应文件块数据。

## S

### Scanner
文件系统扫描器，检测文件变更并计算块哈希。见 `lib/scanner/`。

### Sequence
文件元数据的单调递增序列号，用于增量同步。

### ShortID
设备 ID 的前 8 字节，用于版本向量中的 Counter.ID。

### STTRACE
环境变量，启用特定子系统的调试日志。

### Stignore
`.stignore` 文件，定义忽略规则。

### Synthetic Directory
虚拟目录条目，size 为 128（已废弃，新版本为 0）。

## T

### TLS
传输层安全，Syncthing 使用 TLS 加密所有连接。

### Typed
类型安全的 KV 访问包装器，见 `internal/db/typed.go`。

## U

### UPnP
通用即插即用，NAT 穿透协议之一。

## V

### Vector
版本向量，用于解决并发修改冲突。见 `lib/protocol/vector.go`。

### Versioner
版本控制组件，在文件被覆盖前保存旧版本。见 `lib/versioner/`。

## W

### Wire Format
协议线格式，文件名归一化为 NFC + 正斜杠 + folder-relative。

### WinsConflict
冲突解决仲裁函数，基于版本向量和修改时间决定胜者。
