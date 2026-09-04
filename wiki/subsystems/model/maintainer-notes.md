# Model 维护者笔记

## 安全编辑点

### 添加新的文件夹类型

1. 实现 `folder` 接口（参考 `folder_sendrecv.go`）
2. 在 `model.go` 的 `newFolderRunner` 中注册
3. 在 `lib/config/foldertype.go` 添加类型枚举
4. 在 `lib/protocol/bep_clusterconfig.go` 添加 BEP 类型映射
5. 添加配置迁移（`lib/config/migrations.go`）

### 修改拉取顺序

1. 编辑 `lib/config/pullorder.go` 添加新顺序
2. 在 `folder_sendrecv.go` 的 `pull()` 中实现排序逻辑
3. 添加测试用例

### 修改块大小策略

1. 编辑 `lib/protocol/bep_fileinfo.go` 的 `BlockSizes` 或 `BlockSize()`
2. **注意**：块大小变更影响兼容性，现有文件的块大小不变
3. 更新 `sha256OfEmptyBlock` 预计算（`init()` 校验）

### 添加新的事件类型

1. 在 `lib/events/events.go` 添加 `EventType` 常量
2. 在 `String()` 和 `UnmarshalEventType` 中添加映射
3. 在 Model 中发射事件
4. 在 API 中暴露（如需要）

## 风险区域

### 高风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `model.go` | 死锁 | `mut` 锁与网络操作交互复杂 |
| `folder_sendrecv.go` | 数据损坏 | 流水线协调错误可能导致文件损坏 |
| `sharedpullerstate.go` | 竞态 | mutable 字段并发访问 |
| `indexhandler.go` | 索引丢失 | 序列号跟踪错误可能导致索引不一致 |

### 中风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `folder.go` | 状态不一致 | Serve goroutine 状态变更 |
| `progressemitter.go` | 内存泄漏 | 订阅未正确清理 |
| `deviceactivity.go` | 负载不均 | 计数错误可能导致设备过载 |

## 常见变更配方

### 添加新的 PullOrder

```go
// 1. lib/config/pullorder.go
const (
    PullOrderStandard = iota
    PullOrderNewestFirst
    // ...
    PullOrderMyNewOrder  // 新增
)

func (o PullOrder) String() string {
    switch o {
    // ...
    case PullOrderMyNewOrder:
        return "my-new-order"
    }
}

// 2. lib/model/folder_sendrecv.go - pull() 方法
switch folderCfg.Order {
// ...
case config.PullOrderMyNewOrder:
    sort.Slice(files, func(i, j int) bool {
        // 自定义排序逻辑
    })
}
```

### 添加新的 LocalFlag

```go
// 1. lib/protocol/bep_fileinfo.go
const (
    FlagLocalUnsupported = 1 << 0
    // ...
    FlagLocalMyFlag = 1 << 7  // 新增
)

// 2. 更新 LocalAllFlags 聚合掩码
// 3. 更新 HumanString() 方法
// 4. 注意：LocalFlags 不通过 wire 传输，仅本地使用
```

### 修改扫描行为

```go
// lib/scanner/walk.go - Walk() 方法
// 修改文件变更检测逻辑
// 注意：变更检测影响性能，谨慎修改
```

## 代码审查清单

修改 `lib/model/` 时检查：

- [ ] `mut` 锁内是否有网络操作？（禁止）
- [ ] 新的 goroutine 是否正确取消？（context 传播）
- [ ] 临时文件是否正确清理？（abort 路径）
- [ ] 错误是否正确分类？（致命 vs 可重试）
- [ ] 状态变更是否通过 Serve goroutine？（避免竞态）
- [ ] 序列号是否单调递增？（索引一致性）
- [ ] 加密文件夹是否正确处理？（路径 redact）
- [ ] 是否添加了测试？（正常 + 错误路径）

## 性能考虑

### 内存

- `sharedPullerState` 每文件一个，大量文件时内存开销大
- `fileinfobatch` 批量处理索引，减少内存峰值
- `BufferPool` 复用块缓冲，减少 GC 压力

### I/O

- `fsync` 在 `finalClose` 中调用，保证数据持久性
- 稀疏文件减少磁盘占用
- 块大小影响 I/O 粒度（128KB-16MiB）

### 网络

- `deviceActivity` 负载均衡避免单设备过载
- `DownloadProgress` 避免重复请求
- 限速器（`lib/connections/limiter.go`）控制带宽

## 调试技巧

### 启用调试日志

```bash
STTRACE=model syncthing
```

### 检查文件夹状态

```bash
# REST API
curl http://localhost:8384/rest/system/status
curl http://localhost:8384/rest/db/status?folder=xxx
```

### 检查连接

```bash
curl http://localhost:8384/rest/system/connections
```

### Prometheus 指标

```
http://localhost:8384/rest/metrics
```

关注：
- `syncthing_model_bytes_pulled_total`
- `syncthing_model_files_pulled_total`
- `syncthing_model_files_conflicted_total`
