# Model 算法

## 拉取调度算法

### Need 列表获取

`folder_sendrecv.go` 的 `pull()` 方法首先从数据库获取需要同步的文件列表：

1. 查询 DB 中 `FlagLocalNeeded` 标记的文件
2. 按 `PullOrder` 排序：
   - `OrderStandard`：按名称排序
   - `OrderNewestFirst`：按修改时间降序
   - `OrderOldestFirst`：按修改时间升序
   - `OrderRandom`：随机排序
   - `OrderSmallestFirst`：按文件大小升序
   - `OrderLargestFirst`：按文件大小降序

### 块级调度

对每个需要拉取的文件，算法决定哪些块从本地复制、哪些从网络拉取：

1. **本地块查找**（Copier 阶段）：
   - 检查 origin 文件（本地旧版本）是否有相同块
   - 检查其他文件是否有相同块（`BlocksHash` 匹配）
   - 找到则 `copyDone`，避免网络传输

2. **网络块拉取**（Puller 阶段）：
   - 对剩余块，选择最佳设备拉取
   - `deviceActivity` 跟踪每设备在途请求数
   - 选择负载最低的设备

### deviceActivity 负载均衡

`deviceactivity.go` 实现简单的负载均衡：

- `deviceActivity` map 跟踪每设备的在途请求数
- `selectDevice` 选择在途请求最少的设备
- 拉取开始时 `beginPull`（计数+1），完成时 `endPull`（计数-1）
- 这避免了单一设备过载

### blockPullReorderer 块顺序优化

`blockpullreorderer.go` 优化块拉取顺序：

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| `BlockPullOrderStandard` | 按 offset 顺序 | 顺序 I/O 优化 |
| `BlockPullOrderRandom` | 随机顺序 | 避免多设备同步热点 |
| `BlockPullOrderNone` | 不重排 | 保持 DB 顺序 |

## 冲突解决算法

### 版本向量比较

冲突判定基于版本向量（`lib/protocol/vector.go`）：

1. `InConflictWith(previous)`（`bep_fileinfo.go:190`）：
   - 若 `f.Version >= previous.Version` → 非冲突
   - 若任一 `BlocksHash` 缺失 → 冲突（无法做内容判定）
   - 若 `f.PreviousBlocksHash == previous.BlocksHash` → 非冲突（基于旧内容修改）
   - 否则 → 冲突

2. `WinsConflict(other)`（`bep_fileinfo.go:212`）仲裁：
   - 仅一方 invalid → 非 invalid 方胜
   - 修改时间更晚者胜
   - 时间相等 → `FileVersion().Compare()` 的 `ConcurrentGreater` 决定

### 冲突文件命名

败方文件被重命名为冲突文件：
```
<name>.sync-conflict-YYYYMMDD-HHMMSS-<DEVICEID>.<ext>
```

冲突文件标记为 `FlagLocalReceiveOnly`，不会传播到其他设备。

## 扫描调度算法

### 扫描触发

- **定时扫描**：`scanTimer`，间隔为配置的 `RescanInterval`（默认 1 小时）
- **手动扫描**：API 调用或 CLI
- **watcher 触发**：文件系统事件（`basicfs_watch.go`），防抖后触发
- **错误重扫**：拉取失败时调度扫描（`errModified`）

### 增量扫描

- `Subs []string` 指定子路径，仅扫描变更部分
- watcher 检测到变更时，将变更路径加入 `Subs`
- 全量扫描时 `Subs` 为空

### 文件变更检测

扫描器对比现有 FileInfo 和文件系统状态：
- mtime 变化 → 重新哈希
- size 变化 → 重新哈希
- 权限位变化 → 更新元数据
- 未变更 → 跳过哈希（性能优化）

## 块大小选择算法

`BlockSize(fileSize)`（`bep_fileinfo.go:403`）：

```go
BlockSizes = [128KiB, 256KiB, 512KiB, 1MiB, 2MiB, 4MiB, 8MiB, 16MiB]
DesiredPerFileBlocks = 2000
```

遍历 `BlockSizes`，选第一个使 `fileSize < DesiredPerFileBlocks * blockSize` 的块大小。即每文件约 2000 块的目标。

## 稀疏文件处理

`sharedpullerstate.go` 的稀疏文件优化：

1. 检测全零块：`BlockInfo.IsEmpty()`（`bep_fileinfo.go:637`）通过预计算的 8 种块大小全零 SHA256 查表
2. 稀疏文件创建：`Truncate(size)` 创建稀疏文件，跳过全零块写入
3. `skippedSparseBlock`：计数为 `copyOrigin`（历史原因）

## 临时文件大小估算

`blocksToSize`（`sharedpullerstate.go:454`）估算字节大小：

$$\text{size} = \text{blocks} \times \text{blockSize} - \frac{(\text{blockSize} - \text{fileSize} \bmod \text{blockSize}) \times \text{blocks}}{\text{blocksInFile}}$$

考虑最后一块可能较小，按概率分布估算。

## 拉取退避算法

`folder.go` 的指数退避：

- 初始 `pullPause` = 配置值
- 失败时 `pullPause *= 2`
- 上限 60x 初始值
- 扫描成功后重置为初始值
- `context.Canceled`/`DeadlineExceeded` 不触发退避

## 设备下载状态跟踪

`devicedownloadstate.go` 跟踪每设备已下载的块：

- `DownloadProgress` 消息告知对端已下载块索引
- 对端可优化请求调度，避免重复请求
- `sentdownloadstate.go` 跟踪已发送的状态，实现增量更新
- `progressemitter.go` 定期发射进度（默认 10 秒）
