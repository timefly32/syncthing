# Protocol 缓冲池深入

## BufferPool

`lib/protocol/bufferpool.go`（101 行）实现分级字节缓冲池，减少 GC 压力。

## 数据结构

```go
type BufferPool struct {
    pools map[int]*sync.Pool
    sizes []int
    mut   sync.Mutex
}

var blockSizeToBucket = map[int]int{
    128 * 1024:   0,
    256 * 1024:   1,
    512 * 1024:   2,
    1024 * 1024:  3,
    2 * 1024 * 1024:  4,
    4 * 1024 * 1024:  5,
    8 * 1024 * 1024:  6,
    16 * 1024 * 1024: 7,
}
```

## Get

```go
func (p *BufferPool) Get(size int) []byte {
    // 1. 找到合适的桶
    bucket := p.bucketForSize(size)
    // 2. 从 sync.Pool 获取
    buf := p.pools[bucket].Get()
    // 3. 返回适当大小的切片
    return buf.([]byte)[:size]
}
```

## Put

```go
func (p *BufferPool) Put(buf []byte) {
    // 1. 找到对应的桶
    bucket := p.bucketForSize(cap(buf))
    // 2. 归还到 sync.Pool
    p.pools[bucket].Put(buf)
}
```

## 桶映射

```go
func (p *BufferPool) bucketForSize(size int) int {
    // 找到 >= size 的最小桶
    for i, s := range p.sizes {
        if s >= size {
            return i
        }
    }
    // 超过最大桶，返回最大桶
    return len(p.sizes) - 1
}
```

## 不变量

**关键**：桶内切片的 `cap` 必须等于对应的 BlockSize。

`init()` 校验：

```go
func init() {
    for size := range blockSizeToBucket {
        // 预计算 sha256OfEmptyBlock
        empty := sha256.Sum256(make([]byte, size))
        sha256OfEmptyBlock[size] = empty
    }
}
```

如果块大小不在 `blockSizeToBucket` 中，`init()` 会 panic。

## 使用场景

### readMessage

```go
func (c *rawConnection) readMessage() (proto.Message, error) {
    buf := c.bufferPool.Get(msgLen)
    defer c.bufferPool.Put(buf)
    // 读取消息到 buf
    // 反序列化
}
```

### writeMessage

```go
func (c *rawConnection) writeMessage(msg asyncMessage) {
    buf := c.bufferPool.Get(size)
    defer c.bufferPool.Put(buf)
    // 序列化 msg 到 buf
    // 写入网络
}
```

## 性能

### 优点

- 减少 GC 压力（复用大缓冲）
- 减少内存分配
- 分级减少内部碎片

### 缺点

- 增加内存使用（保留池）
- 桶映射可能浪费（小请求用大桶）

## 测试

`lib/protocol/bufferpool_test.go`（134 行）：

- `TestBufferPoolBucketMapping` — 桶映射正确性
- `TestBufferPoolGetPut` — 获取/归还
- `TestBufferPoolStress` — 并发压力测试
