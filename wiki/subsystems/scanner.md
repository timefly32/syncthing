# 扫描器子系统

## 概述

`lib/scanner/` 负责遍历文件系统、检测文件变更、计算文件块的 SHA-256 哈希。是 Syncthing 同步流程的起点。

## 职责

- **文件系统遍历**：递归遍历目录
- **变更检测**：对比现有 FileInfo 和文件系统状态
- **块哈希**：并行计算文件块的 SHA-256
- **忽略规则**：应用 `.stignore` 模式

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `walk.go` | ~500 | `Config` 结构体、`Walk()` 方法、文件系统遍历 |
| `blocks.go` | ~200 | `Blocks()` 函数、SHA-256 哈希、sync.Pool |
| `blockqueue.go` | ~300 | `HashFile()` 函数、并行哈希器 |
| `metrics.go` | 35 | Prometheus 指标 |

## Config 结构体

```go
type Config struct {
    Folder            string
    Subs              []string          // 子路径扫描
    Matcher           *ignore.Matcher
    Progress          *Progress
    RescanDelay       time.Duration
    ScanDelay         time.Duration
    Hashers           int               // 并行哈希器数量
    ShortID           protocol.ShortID
    TempLifetime      time.Duration
    CurrentFiler      CurrentFiler      // 现有文件信息
    BlockSize         int               // 块大小
    WalkPool          *WalkPool         // 并行控制
    UseLargeBlocks    bool              // 大块模式
    PerFileBlocksize  bool              // 每文件独立块大小
    MaxFileBlocksize  int               // 最大块大小
    MinFileBlocksize  int               // 最小块大小
}
```

## 文件系统遍历

`Walk()` 方法是扫描器入口：

1. **遍历文件系统**：递归遍历目录，应用忽略规则
2. **检测变更**：对比 `CurrentFiler` 和文件系统状态
3. **哈希计算**：对变更的文件计算块哈希
4. **返回结果**：通过 channel 返回 `ScanResult`

### 遍历策略

- **全量扫描**：`Subs` 为空，遍历整个 folder
- **增量扫描**：`Subs` 指定子路径，仅扫描变更部分
- **忽略规则**：`Matcher` 应用 `.stignore` 模式
- **符号链接**：特殊处理
- **错误容忍**：单个文件错误不中断整个扫描

## 块哈希计算

### Blocks() 函数

```go
func Blocks(ctx context.Context, f fs.File, blockSize int, sizeHint int64, progress *Progress) ([]protocol.BlockInfo, error)
```

使用 `sync.Pool` 管理 buffer 和哈希器：

```go
var bufPool = sync.Pool{
    New: func() interface{} { return make([]byte, BlockSize) },
}

var hashPool = sync.Pool{
    New: func() interface{} { return sha256.New() },
}
```

### 块大小

- 默认 `BlockSize = 128 * 1024`（128KB）
- 大块模式（`UseLargeBlocks`）：根据文件大小动态调整
- 每文件独立块大小（`PerFileBlocksize`）：在 `MinFileBlocksize` 和 `MaxFileBlocksize` 之间
- 目标：每文件约 2000 块（`DesiredPerFileBlocks`）

## 并行哈希

### HashFile() 函数

```go
func HashFile(ctx context.Context, tfs fs.Filesystem, name string, blockSize int, progress *Progress, ...) ([]protocol.BlockInfo, error)
```

使用 `parallelHasher` 实现并行哈希：

- 多个 goroutine 并行计算不同块的哈希
- 通过 channel 协调块的分发和结果收集
- 保持块的顺序（通过 offset 排序）
- `Hashers` 配置控制并行度

## 文件变更检测

扫描器对比现有 FileInfo 和文件系统状态：

| 条件 | 行为 |
| --- | --- |
| mtime 变化 | 重新哈希 |
| size 变化 | 重新哈希 |
| 权限位变化 | 更新元数据 |
| 未变更 | 跳过哈希（性能优化） |
| 新文件 | 哈希并添加 |
| 已删除 | 标记 Deleted |

## Prometheus 指标

```go
metricHashedBytes   // 每文件夹哈希字节数
metricScannedItems  // 每文件夹扫描项数
```

`registerFolderMetrics()` 在 folder 注册时预创建指标。

## 测试覆盖

### walk_test.go（999 行）

- 使用 `fs.FilesystemTypeFake` 创建虚拟文件系统
- `newTestFs()` 创建包含多个目录和文件的测试环境
- `.stignore` 支持 `#include` 指令的测试
- `TestWalkSub` 测试子路径扫描
- 设置 `rdebug.SetMaxStack(10 * 1 << 20)` 防止无限递归

### blocks_test.go（141 行）

- `TestBlocks` 测试各种块大小和文件内容组合
- `BenchmarkValidate` 基准测试哈希验证性能

## 设计权衡

### sync.Pool 复用

**选择**：`sync.Pool` 复用 buffer 和哈希器。
**优势**：减少 GC 压力和内存分配。
**劣势**：需要正确重置状态。

### 并行哈希

**选择**：多 goroutine 并行哈希。
**优势**：提高大文件哈希性能。
**劣势**：增加内存使用（多个 buffer 同时存在）。

### 增量扫描

**选择**：通过 `Subs` 支持子路径扫描。
**优势**：减少全量扫描开销。
**劣势**：可能遗漏 folder 外部的变更。

### 大块模式

**选择**：大文件使用更大块。
**优势**：减少块数量（减少元数据开销）。
**劣势**：增加单块传输粒度。

### 错误容忍

**选择**：单个文件错误不中断扫描。
**优势**：扫描鲁棒性。
**劣势**：可能导致数据不一致（需后续扫描纠正）。

## WalkPool 并行控制

`WalkPool` 限制并行扫描的 folder 数量，避免资源耗尽。
