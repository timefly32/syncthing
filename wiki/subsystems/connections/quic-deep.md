# QUIC 连接深入

## QUIC 协议

QUIC 是基于 UDP 的多路复用传输协议，Syncthing 支持作为 TCP 的替代。

## 优势

- **多路复用**：单连接支持多个流，无队头阻塞
- **0-RTT**：快速连接建立
- **连接迁移**：IP 变更不断连
- **内置加密**：TLS 1.3 集成

## quic-go 库

Syncthing 使用 `quic-go/quic-go` 库。

## QUIC 配置

`lib/connections/quic_misc.go`：

```go
quicConfig = &quic.Config{
    MaxIdleTimeout:          30 * time.Second,
    KeepAlivePeriod:         15 * time.Second,
    DisablePathMTUDiscovery: true,  // 某些平台问题
}
```

## 拨号

`lib/connections/quic_dial.go`：

```go
func (d *quicDialer) Dial(ctx context.Context, id protocol.DeviceID, addr string) (internalConn, error) {
    // 1. 获取或创建 Transport
    transport := d.registry.GetOrCreateTransport(addr)
    // 2. QUIC 拨号
    session, err := transport.Dial(ctx, addr, d.tlsCfg, quicConfig)
    // 3. 打开流
    stream, err := session.OpenStream()
    // 4. 包装为 internalConn
    return newQUICConn(stream, session, ...), nil
}
```

## 监听

`lib/connections/quic_listen.go`：

```go
func (l *quicListener) Serve(ctx context.Context) {
    // 1. 创建 Transport
    transport := l.registry.GetOrCreateTransport(l.listenAddr)
    // 2. 监听
    listener, err := transport.Listen(l.tlsCfg, quicConfig)
    // 3. 接受连接
    for {
        session, err := listener.Accept(ctx)
        if err != nil {
            return
        }
        go l.handleSession(session)
    }
}

func (l *quicListener) handleSession(session quic.Session) {
    // 1. 接受流
    stream, err := session.AcceptStream(ctx)
    // 2. 包装为 internalConn
    conn := newQUICConn(stream, session, ...)
    // 3. 投递到 conns channel
    l.conns <- conn
}
```

## NAT 类型检测

```go
type quicListener struct {
    nat atomic.Uint64  // NAT 类型
    ...
}
```

通过 STUN 检测 NAT 类型，影响 QUIC 连接行为。

## Transport 复用

`lib/connections/registry/` 管理 Transport：

- 同一本地地址复用 Transport
- 支持 IPv4/IPv6 分离（quic4/quic6）
- 端口复用

## quicConn

```go
type quicConn struct {
    quic.Stream
    session quic.Session
    ...
}

func (c *quicConn) Close() error {
    // 1. 关闭流
    c.Stream.Close()
    // 2. 关闭会话
    c.session.Close()
    return nil
}
```

## KeepAlive

```go
// quicConfig.KeepAlivePeriod = 15 秒
// quicConfig.MaxIdleTimeout = 30 秒
```

- 每 15 秒发送 keepalive
- 30 秒无活动断开

## 优先级

```go
func (d *quicDialer) Priority() int {
    if d.isLAN {
        return 20  // LAN QUIC
    }
    return 40  // WAN QUIC
}
```

## 限制

### Path MTU Discovery

```go
DisablePathMTUDiscovery: true
```

某些平台 Path MTU Discovery 有问题，因此禁用。

### UDP 缓冲区

QUIC 需要较大的 UDP 接收缓冲区：

```bash
# Linux
sysctl -w net.core.rmem_max=4194304
```

## 测试

`lib/connections/quic_dial_test.go`、`quic_listen_test.go`：

- 拨号/监听测试
- 连接迁移测试
- NAT 类型测试

## 设计权衡

### QUIC vs TCP

**选择**：两者都支持。
**优势**：QUIC 多路复用、0-RTT、连接迁移；TCP 广泛兼容。
**劣势**：QUIC 需要 UDP 支持；TCP 队头阻塞。

### 单流 vs 多流

**选择**：单流（每连接一个 BEP 流）。
**优势**：简单，与 TCP 模型一致。
**劣势**：未充分利用 QUIC 多路复用。
