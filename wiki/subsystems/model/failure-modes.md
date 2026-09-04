# Model 失败模式

## 错误分类

### 致命错误

| 错误 | 位置 | 处理 |
| --- | --- | --- |
| `fatal(err)` | `model.go` | 发送到 `fatalChan`，导致服务退出 |

### 文件夹状态错误

| 错误 | 位置 | 处理 |
| --- | --- | --- |
| `ErrFolderMissing` | `model.go` | `checkFolderRunningRLocked` 返回 |
| `ErrFolderPaused` | `model.go` | `checkFolderRunningRLocked` 返回 |
| `ErrFolderNotRunning` | `model.go` | `checkFolderRunningRLocked` 返回 |

### 加密错误

| 错误 | 位置 | 处理 |
| --- | --- | --- |
| `errEncryption*` | `model.go` | 记录到 `folderEncryptionFailures`，redact 路径 |

加密错误不暴露路径信息（`redactPathError`），因为不可信设备不应看到真实路径。

### 拉取错误

| 错误 | 位置 | 处理 |
| --- | --- | --- |
| `errModified` | `folder_sendrecv.go` | 调度扫描后重试 |
| `errDirHasToBeScanned` | `folder_sendrecv.go` | 目录删除障碍，调度扫描 |
| `errDirHasIgnored` | `folder_sendrecv.go` | 目录含忽略项，调度扫描 |
| `errDirNotEmpty` | `folder_sendrecv.go` | 目录非空，调度扫描 |
| `errNotAvailable` | `folder_sendrecv.go` | 无可用设备，记录 pull error |
| `errNoDevice` | `folder_sendrecv.go` | 无设备有此文件，记录 pull error |

### 可忽略错误

| 错误 | 位置 | 处理 |
| --- | --- | --- |
| `context.Canceled` | `folder_sendrecv.go` | 忽略，不记录 pull error |
| `context.DeadlineExceeded` | `folder_sendrecv.go` | 忽略，不记录 pull error |

## 重试与退避

### 拉取退避

`folder.go` 的指数退避机制：

```
初始 pullPause = 配置值
失败: pullPause *= 2
上限: 60x 初始值
扫描成功: 重置为初始值
```

退避期间文件夹状态保持 `Syncing`，但实际不执行拉取。

### 扫描重试

扫描失败时：
- 记录错误事件
- 保持 `Error` 状态
- 下次 `scanTimer` 触发时重试

### 连接重拨

连接断开时：
- 触发 `dialNow` 信号
- `connections.Service` 的 `connect` 循环立即唤醒
- `nextDialRegistry` 冷却机制防止过度重拨

## 并发风险

### 死锁风险

**已避免**：`mut` 锁内不执行网络操作。

**潜在风险**：如果新增代码在 `mut` 锁内调用 `conn.Index()` 等阻塞方法，可能死锁。因为 `conn.Index()` 等待 `writerLoop`，而 `writerLoop` 可能等待 `dispatcherLoop`，`dispatcherLoop` 调用 `model.Index()` 需要 `mut` 锁。

### 竞态条件

**sharedPullerState**：mutable 字段通过 `sync.Mutex` 保护。`finalClose` 检查 `copyNeeded + pullNeeded == 0` 时持锁。

**folder Serve**：单 goroutine 串行化所有状态变更，避免竞态。但外部读取状态（如 `getState()`）需要通过 channel 或 atomic 操作。

### 资源泄漏

**临时文件泄漏**：如果 `sharedPullerState` 未正常 `finalClose`，临时文件可能残留。`folder_sendrecv.go` 在 `abort` 时清理。

**goroutine 泄漏**：流水线 goroutine 通过 context 取消。如果 context 未正确传播，goroutine 可能泄漏。

## 部分状态处理

### 部分拉取的文件

- 临时文件保留在 `.syncthing.tmp.*`
- 下次拉取时检查是否可复用（`reused` 计数）
- 稀疏文件 `Truncate` 失败时，若 `reused > 0` 则删除重试

### 目录删除障碍

目录非空时无法删除，可能原因：
- 含忽略文件（`errDirHasIgnored`）
- 含未扫描文件（`errDirHasToBeScanned`）
- 含其他文件（`errDirNotEmpty`）

处理：调度扫描，下次拉取时重试。

### 连接断开中途

- 正在传输的块请求超时或失败
- `sharedPullerState` 的块标记为未完成
- 下次拉取时从其他设备重试

## 调试入口点

### 日志

- `lib/model/debug.go`：`l.Debugln` 调试日志
- 启用：`STTRACE=model` 环境变量
- 关键日志点：`pull()`、`Serve()`、`addConnection()`

### 指标

`lib/model/metrics.go` 定义 Prometheus 指标：
- `metricSourceLocalOrigin`：从 origin 复制的块
- `metricSourceLocalOther`：从其他文件复制的块
- `metricSourceSkipped`：跳过的稀疏块
- `metricSourceNetwork`：从网络拉取的块

### 事件

关键事件用于调试：
- `StateChanged`：文件夹状态变更
- `ItemFinished`：单个文件同步完成
- `FolderScanCompleted`：扫描完成
- `DeviceConnected`/`DeviceDisconnected`：连接变更

### 端到端测试

`test/` 目录的集成测试覆盖完整同步场景：
- `test/sync_test.go`：基本同步
- `test/conflict_test.go`：冲突处理
- `test/reconnect_test.go`：重连
- `test/override_test.go`：覆盖操作
