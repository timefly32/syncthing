# Protocol 测试

## 测试文件概览

| 测试文件 | 行数 | 测试数 | 覆盖重点 |
| --- | ---: | --- | --- |
| `protocol_test.go` | 743 | 19 Test + 1 Bench | 握手后流程、关闭、压缩、校验、请求限制 |
| `vector_test.go` | 390 | 5 Test | 版本向量 Update/Merge/Compare（~30 用例） |
| `bep_fileinfo_test.go` | 329 | 4 Test | 等价性、空块、块比较 |
| `encryption_test.go` | 256 | 7 Test | 加解密往返、一致性 |
| `deviceid_test.go` | 150 | 5 Test + 4 Bench | ID 编解码、ShortID |
| `benchmark_test.go` | 200 | 2 Bench | Raw TCP / TLS 吞吐 |
| `bep_hello_test.go` | 115 | 2 Test | Hello 往返、旧版识别 |
| `bufferpool_test.go` | 134 | 3 Test | 桶映射、压力 |
| `conflict_test.go` | 30 | 1 Test | WinsConflict |
| `luhn_test.go` | 30 | 1 Test | Luhn-32 |
| `nativemodel_windows_test.go` | 35 | 1 Test | fixupFiles |
| `mocked_connection_info_test.go` | 602 | — | counterfeiter 生成的 mock |

## 关键测试用例

### 协议流程

- `TestPing` — 双向 ping
- `TestClose` / `TestCloseOnBlockingSend` / `TestCloseRace` / `TestCloseTimeout` — 关闭路径与死锁
- `TestClusterConfigFirst` — ClusterConfig 必须先发
- `TestClusterConfigAfterClose` — 关闭后发送行为
- `TestDispatcherToCloseDeadlock` — dispatcher 与 close 死锁

### 压缩

- `TestWriteCompressed` / `TestLZ4Compression` / `TestLZ4CompressionUpdate` — 压缩

### 校验

- `TestCheckFilename` — 22 个 filename 用例
- `TestCheckConsistency` — 10 个 FileInfo 一致性用例

### 请求限制

- `TestRequestMaxSize` / `TestRequestZeroSize` / `TestRequestInvalidFilename` — 请求校验

### 版本向量

- `TestUpdate` — 含时钟回拨
- `TestCompare` — 约 30 个用例，覆盖 Equal/Greater/Lesser/Concurrent 全部分支

### 加密

- `TestEnDecryptName` — 名字往返
- `TestEnDecryptBytes` — 数据往返
- `TestEnDecryptFileInfo` — FileInfo 往返
- `TestEncryptedFileInfoConsistency` — 加密后一致性

## 覆盖评估

### 强覆盖

- 版本向量比较（30+ 用例）
- filename 校验（22 用例）
- FileInfo 一致性（10 用例）
- 加密往返
- 关闭路径与死锁

### 中等覆盖

- 协议握手
- 压缩
- 设备 ID

### 弱覆盖

- `bep_clusterconfig.go`、`bep_index_updates.go`、`bep_download_progress.go`、`wireformat.go`、`counting.go`、`metrics.go` 无独立单元测试，依赖集成测试

## Mock 支持

`mocks/` 目录（由 counterfeiter 生成）提供：
- `ConnectionInfo` mock
- `Connection` mock

便于上层测试（如 `lib/model`）。

## 运行测试

```bash
# 单元测试
go run build.go test ./lib/protocol/...

# 带竞态检测
go run build.go test -race ./lib/protocol/...

# 基准测试
go run build.go bench ./lib/protocol/...
```
