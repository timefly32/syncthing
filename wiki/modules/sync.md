# 同步原语模块

## 概述

`lib/sync/` 提供 Syncthing 使用的同步原语包装，约 200 行 Go 代码。主要在调试构建中启用死锁检测。

## 职责

- **Mutex 包装**：带死锁检测的互斥锁
- **RWMutex 包装**：带死锁检测的读写锁
- **WaitGroup 包装**：带死锁检测的 WaitGroup

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `sync.go` | ~100 | 类型定义和工厂函数 |
| `debug.go` | ~100 | 调试模式实现 |

## 类型

```go
type Mutex struct {
    deadlock.RWMutex  // 调试模式
    sync.Mutex        // 生产模式
}

type RWMutex struct {
    deadlock.RWMutex  // 调试模式
    sync.RWMutex      // 生产模式
}

type WaitGroup struct {
    deadlock.WaitGroup  // 调试模式
    sync.WaitGroup      // 生产模式
}
```

## 构建标签

- 默认（调试）：使用 `go-deadlock` 库
- `-no-deadlock`：使用标准库 `sync`

## 死锁检测

`go-deadlock` 库提供：

- 锁顺序检测
- 等待图检测
- 超时检测（默认 30 秒）
- 调用栈输出

## 使用

```go
import "github.com/syncthing/syncthing/lib/sync"

var mut sync.Mutex

mut.Lock()
defer mut.Unlock()
```

## 设计权衡

### 调试 vs 性能

**选择**：调试构建启用死锁检测，生产构建禁用。
**优势**：开发时检测死锁，生产时无性能开销。
**劣势**：需要正确配置构建标签。

### 包装 vs 直接使用

**选择**：包装标准库类型。
**优势**：统一接口，便于切换实现。
**劣势**：增加间接性。
