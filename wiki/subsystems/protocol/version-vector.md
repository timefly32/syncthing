# 版本向量

## 概述

版本向量（`lib/protocol/vector.go`，329 行）是 Syncthing 解决并发修改冲突的核心数据结构。每个文件的 `FileInfo` 携带一个 `Version` 向量，记录所有设备的修改计数。

## 数据结构

```go
type Vector struct {
    Counters []Counter
}

type Counter struct {
    ID    ShortID  // 设备短 ID（设备 ID 前 8 字节）
    Value uint64   // 修改计数
}
```

**不变量**：`Counters` 按 `ID` 升序排列。`Update` 和 `Merge` 都维护此序。

## 核心操作

### Update(id)

递增某 ID 的计数器（`vector.go:122-149`）：

```go
func (v *Vector) Update(id ShortID) {
    now := time.Now().Unix()
    // 查找现有 counter
    for i, c := range v.Counters {
        if c.ID == id {
            v.Counters[i].Value = max(c.Value+1, now)  // 防回拨
            return
        }
        if c.ID > id {
            // 插入新 counter
            v.Counters = slices.Insert(v.Counters, i, Counter{id, max(1, now)})
            return
        }
    }
    // 末尾追加
    v.Counters = append(v.Counters, Counter{id, max(1, now)})
}
```

**防回拨**：`Value = max(Value+1, now)`。若时钟回拨导致 `now` 较小，仍至少 +1。counter 值同时承载逻辑时钟和物理时间。

### Merge(b)

归并取各 counter 的最大值（`vector.go:154-184`）：

- 遇到 b 中更小的 ID 时插入新 counter
- 维护升序

### Compare(b) — 核心算法

返回 `Ordering`（`vector.go:248-254`）：

```go
type Ordering int
const (
    Equal Ordering = iota
    Greater
    Lesser
    ConcurrentLesser
    ConcurrentGreater
)
```

**算法**（`vector.go:263-328`）：双指针遍历两个已排序 counter 列表，跟踪 `result` 状态：

- 同 ID：比较 Value，更新 result；若与已有 result 矛盾（一方大一方小）→ 返回 Concurrent
- a 有 b 无：若 `av.Value > 0`，a 在此维度更大
- b 有 a 无：若 `bv.Value > 0`，b 在此维度更大
- 任何时候发现 result 从 Greater 变 Lesser 或反之 → 返回 Concurrent

**语义**（`vector.go:256-260`）：版本向量本无"concurrent lesser/greater"之分，只有"concurrent"。但为稳定排序提供严格序，返回两种变体。

### 比较规则示例

| Vector A | Vector B | 结果 | 说明 |
| --- | --- | --- | --- |
| `{42:0}` | `{}` | Equal | 零值等价于缺失 |
| `{42:1}` | `{}` | Greater | A 有非零值 |
| `{42:2}` | `{42:1}` | Greater | 同 ID，A 值更大 |
| `{42:1}` | `{43:1}` | ConcurrentGreater | 不同 ID 都有非零值 |
| `{22:23, 42:1}` | `{22:22, 42:2}` | ConcurrentGreater | 22 大但 42 小 |

## 便捷方法

| 方法 | 说明 |
| --- | --- |
| `Equal(b)` | `Compare(b) == Equal` |
| `LesserEqual(b)` | `Compare(b) <= Greater` |
| `GreaterEqual(b)` | `Compare(b) >= Lesser` |
| `Concurrent(b)` | 两者都非空且非 Equal |
| `Counter(id)` | 取某 ID 的值 |
| `IsEmpty()` | 无非零 counter |
| `DropOthers(id)` | 只保留指定 ID 的 counter |
| `Copy()` | 深拷贝 |

## 序列化

### Wire 格式

`ToWire`/`VectorFromWire`（`vector.go:52-72`）：protobuf 序列化。

### 字符串格式

`String()`（`vector.go:28-39`）：`hex_id:value` 逗号分隔。

```
16进制ID:值,16进制ID:值
```

`VectorFromString`（`vector.go:74-97`）：反向解析。

## 在冲突解决中的应用

### WinsConflict 仲裁

`FileInfo.WinsConflict(other)`（`bep_fileinfo.go:212-229`）：

1. 仅一方 invalid → 非 invalid 方胜
2. 修改时间更晚者胜
3. 时间相等 → `FileVersion().Compare()` 的 `ConcurrentGreater` 决定

### InConflictWith 判定

`FileInfo.InConflictWith(previous)`（`bep_fileinfo.go:190-208`）：

1. 若 `f.Version >= previous.Version` → 非冲突
2. 若任一 `BlocksHash` 缺失 → 冲突（无法做内容判定）
3. 若 `f.PreviousBlocksHash == previous.BlocksHash` → 非冲突（基于旧内容修改）
4. 否则 → 冲突

## 设计权衡

### 版本向量 vs 时间戳

**选择**：版本向量。
**优势**：正确处理并发修改，不依赖时钟同步。
**劣势**：存储开销（每设备一个 counter），大集群时向量较长。

### 逻辑时钟 + 物理时间

**选择**：counter 值 = `max(Value+1, now)`。
**优势**：防时钟回拨，同时保留物理时间信息（用于冲突仲裁的 tiebreaker）。
**劣势**：时钟跳跃可能影响冲突仲裁。

### ConcurrentLesser/ConcurrentGreater

**选择**：区分两种 Concurrent 变体。
**优势**：为稳定排序提供严格序。
**劣势**：语义上 concurrent 本无方向，这是实现便利。
