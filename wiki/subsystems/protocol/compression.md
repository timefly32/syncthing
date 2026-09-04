# Protocol 压缩深入

## 压缩策略

`lib/protocol/protocol.go`：

```go
type Compression int

const (
    CompressionNever    Compression = 0
    CompressionMetadata Compression = 1
    CompressionAlways   Compression = 2
)
```

## shouldCompressMessage

```go
func shouldCompressMessage(compression Compression, t proto.MessageType, n int) bool {
    switch compression {
    case CompressionNever:
        return false
    case CompressionAlways:
        return n > compressionThreshold
    case CompressionMetadata:
        // 元数据压缩，但 Response 不压缩（数据通常已不可压缩）
        if t == proto.MessageTypeResponse {
            return false
        }
        return n > compressionThreshold
    }
    return false
}
```

## 压缩阈值

```go
const compressionThreshold = 128  // 字节
```

小于 128 字节的消息不压缩（压缩开销大于收益）。

## 压缩收益门槛

```go
func writeCompressedMessage(...) {
    // 1. 序列化消息
    uncompressed := marshal(msg)
    // 2. 压缩
    compressed := lz4Compress(uncompressed)
    // 3. 检查收益
    if len(compressed) >= len(uncompressed)-len(uncompressed)/32 {
        // 压缩后大小 >= 原始大小的 31/32（节省 < 3.125%）
        // 放弃压缩
        writeUncompressed(uncompressed)
        return
    }
    // 4. 写入压缩数据
    writeCompressed(compressed, len(uncompressed))
}
```

## LZ4 格式

压缩的消息格式：

```
[4字节原始大小][LZ4 压缩数据]
```

前 4 字节存原始大小，用于解压缓冲分配。

## 解压

```go
func readMessage(...) {
    // 1. 读取 Header
    header := readHeader()
    // 2. 读取消息
    data := readBytes(msgLen)
    // 3. 检查压缩标志
    if header.Compressed {
        // 4. 读取原始大小
        origSize := binary.BigEndian.Uint32(data[:4])
        // 5. 解压
        decompressed := lz4Decompress(data[4:], origSize)
        data = decompressed
    }
    // 6. 反序列化
    msg := unmarshal(data, header.Type)
}
```

## 压缩性能

### 优点

- 减少网络传输
- 索引消息通常可压缩（文件名重复）
- LZ4 压缩/解压速度快

### 缺点

- 增加 CPU 开销
- Response 数据通常已不可压缩（加密文件、媒体文件）
- 小消息压缩无收益

## 配置

设备级配置：

```xml
<device id="xxx" compression="metadata">
```

| 值 | 说明 |
| --- | --- |
| `never` | 不压缩 |
| `metadata` | 仅压缩元数据（默认） |
| `always` | 总是压缩 |

## 协商

压缩策略在 ClusterConfig 中交换：

```protobuf
message FolderDevice {
    ...
    optional Compression compression = 6;
}
```

双方使用较保守的策略（取较小值）。

## 测试

`lib/protocol/protocol_test.go`：

- `TestWriteCompressed` — 压缩写入
- `TestLZ4Compression` — LZ4 往返
- `TestLZ4CompressionUpdate` — 增量索引压缩
