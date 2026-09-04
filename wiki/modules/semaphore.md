# 信号量模块

## 概述

`lib/semaphore/` 提供可取消的信号量实现，约 100 行 Go 代码。用于限制并发操作数量。

## 职责

- **并发限制**：限制同时进行的操作数量
- **上下文取消**：支持 `context.Context` 取消

## 实现

```go
type Semaphore struct {
    tokens chan struct{}
}
```

### NewSemaphore

```go
func NewSemaphore(n int) *Semaphore {
    return &Semaphore{tokens: make(chan struct{}, n)}
}
```

### Acquire

```go
func (s *Semaphore) Acquire(ctx context.Context) error
```

- 阻塞直到获取令牌或 ctx 取消
- 返回 `ctx.Err()` 如被取消

### Release

```go
func (s *Semaphore) Release()
```

归还令牌。

## 使用场景

### 连接拨号

`lib/connections/service.go`：

```go
sem := semaphore.NewSemaphore(dialMaxParallel)  // 64
for _, target := range targets {
    sem.Acquire(ctx)
    go func() {
        defer sem.Release()
        dial(target)
    }()
}
```

### 文件扫描

限制并行扫描的 folder 数量。

### 哈希计算

限制并行哈希器数量。

## 测试覆盖

`semaphore_test.go`：

- 获取/释放测试
- 取消测试
- 并发测试

## 设计权衡

### channel vs 计数器

**选择**：基于 channel 的信号量。
**优势**：简单，天然支持并发。
**劣势**：令牌是 `struct{}`，有少量开销。

### 上下文取消

**选择**：支持 `context.Context`。
**优势**：可取消等待，避免 goroutine 泄漏。
**劣势**：增加 select 复杂度。
