# Protocol 维护者笔记

## 安全编辑点

### 添加新的消息类型

1. 在 `proto/bep/bep.proto` 添加消息定义
2. 运行 `go run build.go proto` 重新生成
3. 在 `lib/protocol/` 添加 `bep_<type>.go` 实现 wire 转换
4. 在 `protocol.go` 的 `dispatcherLoop` 添加分发逻辑
5. 在 `Model` 接口添加回调方法
6. 在 `lib/model` 实现回调
7. 添加测试

### 修改消息线格式

**高风险**：线格式变更影响所有版本的兼容性。

1. 修改 `proto/bep/bep.proto`
2. 考虑向后兼容（旧版本能否解析新格式）
3. 更新 `readMessage`/`writeMessage` 如需新头部字段
4. 添加版本协商（如需）

### 修改压缩策略

1. 编辑 `protocol.go` 的 `shouldCompressMessage`（L935-952）
2. 注意：Response 通常不压缩（数据已不可压缩）
3. 更新压缩收益门槛（`n - n/32`）如需

### 修改块大小

1. 编辑 `bep_fileinfo.go` 的 `BlockSizes` 或 `BlockSize()`
2. **注意**：`sha256OfEmptyBlock` 预计算必须覆盖所有块大小（`init()` 校验，否则 panic）
3. 更新 `BufferPool` 桶（`bufferpool.go`）

### 修改加密参数

1. 编辑 `encryption.go` 的常量
2. **注意**：`minPaddedSize` 变更影响兼容性
3. 更新 `encryptFileInfo` 的伪造逻辑
4. 添加兼容性测试

## 风险区域

### 高风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `protocol.go` | 死锁 | 读写循环与关闭路径交互复杂 |
| `encryption.go` | 数据泄露 | 加密错误可能泄露明文 |
| `bep_fileinfo.go` | 数据损坏 | 序列化错误可能导致文件元数据丢失 |
| `vector.go` | 冲突错误 | 比较算法错误可能导致错误冲突解决 |

### 中风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `deviceid.go` | 信任错误 | ID 计算错误可能导致错误认证 |
| `bufferpool.go` | 内存泄漏 | 池管理错误可能导致缓冲泄漏 |
| `wireformat.go` | 路径错误 | 归一化错误可能导致路径问题 |

## 代码审查清单

修改 `lib/protocol/` 时检查：

- [ ] 新消息类型是否在 dispatcherLoop 正确分发？
- [ ] 状态机检查是否覆盖新消息？（stateInitial vs stateReady）
- [ ] filename 校验是否保持？（`checkFilename`）
- [ ] 一致性校验是否更新？（`checkIndexConsistency`）
- [ ] LocalFlags 是否泄露到 wire？（禁止）
- [ ] 加密路径是否正确处理？（encrypt/decrypt 对称）
- [ ] 版本向量不变量是否保持？（Counters 升序）
- [ ] 缓冲是否正确归还？（BufferPool.Put）
- [ ] 关闭路径是否无死锁？（Close vs internalClose）
- [ ] 是否添加了测试？

## 性能考虑

### 吞吐量

- 无缓冲 channel 形成背压，可能限制吞吐
- LZ4 压缩减少网络传输，但增加 CPU
- `BufferPool` 复用缓冲减少 GC

### 延迟

- `PingSendInterval=90s`，`ReceiveTimeout=300s`
- 请求-响应配对阻塞等待
- `idxMut` 序列化 Index 调用

### 内存

- `MaxMessageLen=500MB` 防止恶意对端耗尽内存
- `BufferPool` 按块大小分级
- `awaiting` map 跟踪在途请求

## 兼容性

### 协议版本

- `HelloMessageMagic=0x2EA7D90B`：当前版本
- `Version13HelloMagic=0x9F79BC40`：旧版，返回 `ErrTooOldVersion`
- 未知魔数返回 `ErrUnknownMagic`

### 消息兼容

- 未知消息类型跳过（未来兼容）
- `LocalFlags` 不上 wire（内部状态）
- `truncated` FileInfo 只能序列化 Deleted/Invalid/Ignored 条目

### 加密兼容

- 解密 hash 时先尝试带 offset additional，失败则尝试 nil（兼容旧版）
- `untypeoify` 纠正易混淆数字（兼容手动输入）

## 调试

### 启用调试日志

```bash
STTRACE=protocol syncthing
```

### 检查连接

```bash
curl http://localhost:8384/rest/system/connections
```

### 检查设备 ID

```bash
syncthing cli show system
```
