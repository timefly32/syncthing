# Protocol 加密

## 概述

`lib/protocol/encryption.go`（698 行）实现加密文件夹的协议支持。加密文件夹允许不可信设备（如云服务器）存储同步数据，而无法读取文件内容。

## 场景

- **可信设备**：持有加密密码，能解密所有数据
- **不可信设备**：仅存储加密数据，无法读取文件名、内容或元数据
- **加密文件夹类型**：`ReceiveEncrypted`，不可信设备使用此类型

## 双层架构

### encryptedModel（入站）

处理来自**不可信设备**的入站请求（`encryption.go:47-162`）：

- `Index`/`IndexUpdate`：解密 FileInfo
- `Request`：解密请求参数（name/offset/size/hash），调用真实 model，加密响应数据
- `DownloadProgress`：忽略（加密文件夹不传输进度）

### encryptedConnection（出站）

处理发往**不可信设备**的出站消息（`encryption.go:166-271`）：

- `Index`/`IndexUpdate`：加密 FileInfo
- `Request`：加密请求参数，解密响应
- `ClusterConfig`：设置 folder keys 后转发
- `DownloadProgress`：丢弃

## 密钥派生

### KeyGenerator

```go
type KeyGenerator struct {
    folderKeys *lru.TwoQueueCache[folderKeyCacheKey, *[32]byte]  // 1000 条
    fileKeys   *lru.TwoQueueCache[fileKeyCacheKey, *[32]byte]    // 5000 条
}
```

两级 LRU 缓存避免重复派生。

### Folder Key

`KeyFromPassword(folderID, password)`（`encryption.go:555-573`）：

```go
scrypt.Key(password, "syncthing"+folderID, 32768, 8, 1, 32)
```

- 盐 = `"syncthing" + folderID`
- N=32768（scrypt 内存成本参数）
- 输出 32 字节密钥

### File Key

`FileKey(filename, folderKey)`（`encryption.go:582-597`）：

```go
hkdf.New(sha256, folderKey+filename, salt="syncthing", nil)
```

每文件独立密钥，基于 folder key + filename 派生。

## 加密原语

### 确定性加密（AES-SIV）

`encryptDeterministic`/`decryptDeterministic`（`encryption.go:452-467`）：

- 用于**文件名**和**块 hash**（需确定性以便对端去重）
- `miscreant.NewAEAD("AES-SIV", key, 0)`，nonce 为 nil
- AES-SIV 是 misuse-resistant 的认证加密

### 随机 nonce 加密（ChaCha20-Poly1305 X）

`encrypt`/`DecryptBytes`（`encryption.go:469-508`）：

- 用于**文件数据**和**FileInfo 序列化**
- nonce 24 字节随机，前置在密文前
- `aead.Seal(nonce[:], nonce[:], data, nil)` — nonce 既作为 AEAD nonce 又前置传输

## encryptFileInfo 算法

`encryptFileInfo`（`encryption.go:281-369`）：

1. `fileKey = FileKey(name, folderKey)`
2. 序列化真实 FileInfo → `encryptBytes`（随机 nonce）→ 存入 `Encrypted` 字段
3. **伪造版本向量**：单 counter `{ID:1, Value=sum(所有真实 counter Value)}`
   - 目的：让不可信设备总是接受最新版本
   - 可信设备解密后看真实 Version
   - 需确定性以使所有可信设备一致
4. **伪造块列表**：
   - 每块 size = `max(realSize, minPaddedSize) + blockOverhead`
   - hash = `encryptBlockHash(realHash, realOffset, fileKey)`
   - offset 重算
5. **伪造 FileInfo**：
   - name = `encryptName(realName, folderKey)`
   - type：非文件 → Directory（symlink 用目录表示）
   - permissions = 0o644
   - ModifiedS = 1234567890（固定时间戳）
   - 保留 Deleted、Sequence
   - 若 invalid → `FlagLocalRemoteInvalid`

## 块 hash 加密

`encryptBlockHash`（`encryption.go:371-379`）：

```go
additional = uint64(offset)  // 大端
return encryptDeterministic(hash, fileKey, additional)
```

**关键设计**：offset 作为 AEAD additional data，使相同数据在不同 offset 产生不同密文 hash——**防止识别跨文件的相同数据块**。

## 请求/响应加密转换

### 出站 Request

`encryptedConnection.Request`（`encryption.go:205-246`）：

- `encSize = max(Size, minPaddedSize) + blockOverhead`
- `encOffset = Offset + BlockNo * blockOverhead`
- `encName = encryptName(Name)`
- `encHash = encryptBlockHash(Hash, Offset, fileKey)`
- 发送加密请求，收到响应后 `DecryptBytes` 并截取 `[:req.Size]`

### 入站 Request

`encryptedModel.Request`（`encryption.go:81-143`）：

- 反向解密 name/size/offset/hash
- `realSize = Size - blockOverhead`
- `realOffset = Offset - BlockNo*blockOverhead`（负值 panic）
- 解密 hash 时先尝试带 offset additional，失败则尝试 nil（兼容旧版）
- 响应数据若 < minPaddedSize 则随机填充

## 文件名加密与路径树

`encryptName` → base32hex → `slashify`（`encryption.go:607-636`）：

```
ABCDEFGH... => A.syncthing-enc/BC/DEFGH...
```

- 首字符 + `.syncthing-enc` 作为顶层目录
- 次 2 字符作为子目录
- 之后每 200 字符分段

`deslashify` 反向操作。`IsEncryptedParent`（`encryption.go:652-674`）判断路径组件是否指向加密数据的父目录。

## 加密参数

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `nonceSize` | 24 | chacha20poly1305 X 模式 |
| `tagSize` | 16 | 认证标签 |
| `keySize` | 32 | 密钥长度 |
| `minPaddedSize` | 1024 | 最小块大小（防大小泄露） |
| `blockOverhead` | 40 | tagSize + nonceSize |
| `encryptedDirExtension` | `.syncthing-enc` | 加密目录扩展名 |
| `folderKeyCacheEntries` | 1000 | folder key 缓存 |
| `fileKeyCacheEntries` | 5000 | file key 缓存 |

## folderKeyRegistry

```go
type folderKeyRegistry struct {
    keys map[string]*[32]byte
    mut  sync.RWMutex
}
```

`setPasswords` 在 `ClusterConfig` 时用 `keysFromPasswords` 重建整个 map。

## 安全性分析

### 保密性

- 文件名：AES-SIV 确定性加密（允许去重）
- 文件数据：ChaCha20-Poly1305 X 随机 nonce
- 块 hash：AES-SIV 确定性加密，绑定 offset

### 完整性

- AES-SIV 和 ChaCha20-Poly1305 X 都是认证加密
- 任何篡改都会被检测

### 大小泄露防护

- `minPaddedSize=1024`：小块填充到 1024 字节
- 固定时间戳（1234567890）
- 固定权限（0o644）
- symlink 用 Directory 表示

### 已知限制

- 文件数量和总大小对不可信设备可见
- 块大小模式可能泄露信息（padding 仅到 1024）
- 修改频率通过 IndexUpdate 时序可观察

## 测试覆盖

`encryption_test.go`（256 行）：

- `TestEnDecryptName` — 名字往返
- `TestKeyDerivation` — 密钥派生确定性
- `TestDecryptNameInvalid` — 篡改检测
- `TestEnDecryptBytes` — 数据往返
- `TestEnDecryptFileInfo` — FileInfo 往返
- `TestEncryptedFileInfoConsistency` — 加密后一致性
- `TestIsEncryptedParent` — 路径判断
