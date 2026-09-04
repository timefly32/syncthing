# 协议消息详细格式

## Header 格式

每个消息前有 Header：

```protobuf
message Header {
    required MessageType type = 1;
    optional bool compressed = 2;
}
```

线格式：`[2字节 Header 长度][Header proto]`

## 消息长度

`[4字节 消息长度][Message proto]`

- 长度为 `int32`（有符号）
- 负值或超过 `MaxMessageLen=500MB` 报错

## 压缩格式

压缩的消息：

```
[4字节原始大小][LZ4 压缩数据]
```

前 4 字节存原始大小，用于解压缓冲分配。

## ClusterConfig 消息

```protobuf
message ClusterConfig {
    repeated Folder folders = 1;
}

message Folder {
    required string id = 1;
    required string label = 2;
    required bool read_only = 3;
    required bool ignore_permissions = 4;
    required bool ignore_delete = 5;
    required bool disable_temp_indexes = 6;
    repeated FolderDevice devices = 7;
}

message FolderDevice {
    required bytes id = 1;
    optional bytes encryption_password = 2;
    optional int64 index_id = 3;
    optional int64 max_sequence = 4;
    optional IndexID index_id_v4 = 5;
}
```

## Index 消息

```protobuf
message Index {
    required string folder = 1;
    repeated FileInfo files = 2;
}

message IndexUpdate {
    required string folder = 1;
    repeated FileInfo files = 2;
}
```

## FileInfo 消息

```protobuf
message FileInfo {
    required string name = 1;
    optional int64 modified_s = 3;
    optional uint32 modified_by = 12;
    optional int64 size = 8;
    optional bytes blocks_hash = 13;
    optional Vector version = 9;
    optional int64 sequence = 10;
    repeated BlockInfo blocks = 16;
    optional bytes symlink_target = 17;
    optional bytes blocks_sha256 = 18;
    optional int32 permissions = 14;
    optional FileType type = 2;
    optional bool deleted = 5;
    optional bool invalid = 6;
    optional PlatformData platform = 15;
    optional bytes encrypted = 19;
}
```

## BlockInfo 消息

```protobuf
message BlockInfo {
    required int64 offset = 1;
    required int32 size = 2;
    required bytes hash = 3;
}
```

## Request 消息

```protobuf
message Request {
    required int32 id = 1;
    required string folder = 2;
    required string name = 3;
    required int64 offset = 4;
    required int32 size = 5;
    optional bytes hash = 6;
    optional bool from_temp = 7;
}
```

## Response 消息

```protobuf
message Response {
    required int32 id = 1;
    optional bytes data = 2;
    optional ErrorCode code = 3;
    optional string message = 4;
}
```

### ErrorCode

| 值 | 名称 | 说明 |
| --- | --- | --- |
| 0 | `NO_ERROR` | 成功 |
| 1 | `GENERIC` | 通用错误 |
| 2 | `NO_SUCH_FILE` | 文件不存在 |
| 3 | `INVALID_FILE` | 文件无效 |
| 4 | `TIMEOUT` | 超时 |

## DownloadProgress 消息

```protobuf
message DownloadProgress {
    required string folder = 1;
    repeated BlockMapEntry updates = 2;
}

message BlockMapEntry {
    required string name = 1;
    required int32 state = 2;
    repeated BlockInfo blocks = 3;
}
```

## Ping 消息

```protobuf
message Ping {
}
```

空消息，仅用于活性检测。

## Close 消息

```protobuf
message Close {
    optional string reason = 1;
}
```

## Hello 消息（非 protobuf）

Hello 消息使用自定义线格式（非 protobuf）：

```
[4字节 magic][4字节 version][8字节 timestamp][2字节 client name 长度][client name][2字节 client version 长度][client version]
```

- `magic`：`0x2EA7D90B`（当前版本）或 `0x9F79BC40`（旧版）
- `version`：协议版本（当前 1）
- `timestamp`：Unix 时间戳（纳秒）
- `client name`：如 "syncthing"
- `client version`：如 "v1.27.0"
