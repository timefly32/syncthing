# 随机数模块

## 概述

`lib/rand/` 提供随机数生成工具，约 100 行 Go 代码。

## 职责

- **随机字符串**：生成指定长度的随机字符串
- **随机字节**：生成随机字节序列
- **随机选择**：从切片随机选择

## 函数

### String

```go
func String(n int) string {
    b := make([]byte, n)
    io.ReadFull(rand.Reader, b)
    return base32.StdEncoding.EncodeToString(b)
}
```

### Bytes

```go
func Bytes(n int) []byte {
    b := make([]byte, n)
    io.ReadFull(rand.Reader, b)
    return b
}
```

### Int64

```go
func Int64() int64 {
    var buf [8]byte
    io.ReadFull(rand.Reader, buf[:])
    return int64(binary.BigEndian.Uint64(buf[:]))
}
```

## 随机源

使用 `crypto/rand`：

- 加密安全
- 适用于密钥、令牌生成

## 使用场景

- API Key 生成
- CSRF 令牌
- 连接 ID
- 临时文件名

## 测试

`rand_test.go`：

- 随机性测试
- 长度测试
- 唯一性测试
