# 旧数据库迁移

## 概述

`internal/db/olddb/` 是基于 LevelDB 的旧数据库实现，仅用于从旧版本迁移到 SQLite。**不应在新代码中使用**。

## Key-Value 模型

旧数据库使用前缀字节区分 key 类型（`keyer.go`）：

| Key 类型 | 前缀 | 格式 | 值 |
| --- | ---: | --- | --- |
| `KeyTypeDevice` | 0 | `<folder ID><device ID><name>` | FileInfo |
| `KeyTypeGlobal` | 1 | `<folder ID><name>` | VersionList |
| `KeyTypeBlock` | 2 | `<folder ID><hash><name>` | block index |
| `KeyTypeVirtualMtime` | 5 | `<folder ID><name>` | mtimeMapping |
| `KeyTypeFolderIdx` | 6 | `<id>` | string |
| `KeyTypeDeviceIdx` | 7 | `<id>` | string |
| `KeyTypeIndexID` | 8 | `<device ID><folder ID>` | IndexID |
| `KeyTypeFolderMeta` | 9 | `<folder ID>` | CountsSet |
| `KeyTypeSequence` | 11 | `<folder ID><seq>` | KeyTypeDevice key |
| `KeyTypeNeed` | 12 | `<folder ID><name>` | `<nothing>` |
| `KeyTypeBlockList` | 13 | `<hash>` | BlockList |
| `KeyTypeBlockListMap` | 14 | `<folder ID><hash><name>` | `<nothing>` |
| `KeyTypeVersion` | 15 | `<hash>` | Vector |
| `KeyTypePendingFolder` | 16 | — | — |
| `KeyTypePendingDevice` | 17 | — | — |

Key 格式使用 BigEndian 编码的 uint32 索引（folder/device）。

## smallIndex

`smallindex.go` 实现内存中的双向 `[]byte ↔ uint32` 映射：

```go
type smallIndex struct {
    db     backend.Backend
    prefix []byte
    id2val map[uint32]string
    val2id map[string]uint32
    nextID uint32
    mut    sync.Mutex
}
```

**注意**：`ID()` 方法在找不到时会 `panic("missing ID")`（`smallindex.go:76`），因为旧数据库仅用于迁移，不应有新 folder/device 被添加。

## 事务处理

`transactions.go` 的 `readOnlyTransaction` 提供快照读取：

- `getFileByKey()`/`getFileTrunc()` — 获取文件信息
- `fillFileInfo()` — 解析 blocks 和 version 的间接引用
- `withHaveSequence()` — 按序列号范围迭代

### 间接引用解析

- `BlocksHash` → `BlockList`（通过 `KeyTypeBlockList`）
- `VersionHash` → `Vector`（通过 `KeyTypeVersion`）

## 迁移到 SQLite

迁移过程将 LevelDB 中的数据转换到 SQLite per-folder 数据库：

1. 遍历旧数据库的 folder 索引
2. 为每个 folder 创建新的 SQLite 数据库
3. 遍历 folder 的所有文件记录
4. 转换为 SQLite schema 格式
5. 更新主数据库的 folder 索引

## 为什么迁移到 SQLite

### LevelDB 的问题

- 单一全局数据库，无 per-folder 隔离
- 无事务支持，崩溃可能损坏数据
- 压缩（compaction）影响性能
- 无 SQL 查询能力

### SQLite 的优势

- Per-folder 数据库隔离
- 完整事务支持（ACID）
- WAL 模式高并发
- SQL 查询能力
- 触发器自动维护计数
- STRICT 表类型安全

## 维护者注意

- `internal/db/olddb/` **仅用于迁移**，不要在新代码中使用
- `smallIndex.ID()` 会 panic，因为不应有新条目
- 迁移完成后可考虑移除旧数据库代码
- 旧数据库的 key 格式与 SQLite schema 完全不同
