# TCP 连接深入

## TCP 拨号

`lib/connections/tcp_dial.go`：

```go
func (d *tcpDialer) Dial(ctx context.Context, id protocol.DeviceID, addr string) (internalConn, error) {
    // 1. 解析地址
    tcpAddr, err := net.ResolveTCPAddr("tcp", addr)
    // 2. 拨号（端口复用）
    conn, err := d.dialer.DialContextReusePortFunc(ctx, "tcp", addr)
    // 3. TLS 握手
    tlsConn := tls.Client(conn, d.tlsCfg)
    tlsConn.HandshakeContext(ctx)
    // 4. 包装为 internalConn
    return newTCPConn(tlsConn, ...), nil
}
```

## TCP 监听

`lib/connections/tcp_listen.go`：

```go
func (l *tcpListener) Serve(ctx context.Context) {
    // 1. 创建监听器（端口复用）
    listener, err := net.ListenConfig{
        Control: dialer.ReusePortControl,
    }.Listen(ctx, "tcp", l.listenAddr)
    // 2. 注册 NAT 映射
    l.natService.NewMapping("tcp", l.listenPort)
    // 3. 接受连接
    for {
        conn, err := listener.Accept()
        if err != nil {
            return
        }
        go l.handleConn(conn)
    }
}

func (l *tcpListener) handleConn(conn net.Conn) {
    // 1. TLS 握手
    tlsConn := tls.Server(conn, l.tlsCfg)
    tlsConn.HandshakeContext(ctx)
    // 2. 包装为 internalConn
    internalConn := newTCPConn(tlsConn, ...)
    // 3. 投递到 conns channel
    l.conns <- internalConn
}
```

## 端口复用

`lib/dialer/`：

### ReusePortControl

```go
func ReusePortControl(network, address string, c syscall.RawConn) error {
    // 设置 SO_REUSEPORT
    c.Control(func(fd uintptr) {
        unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
    })
    return nil
}
```

### 运行时检测

```go
var reusePortSupported = detectReusePortSupport()

func detectReusePortSupport() bool {
    // 尝试设置 SO_REUSEPORT
    // 返回是否支持
}
```

### DialContextReusePortFunc

```go
func DialContextReusePortFunc(ctx context.Context, network, addr string) (net.Conn, error) {
    // 1. 从 registry 获取监听地址
    listenAddr := registry.GetListenAddr()
    // 2. 双拨号策略
    return dialTwicePreferFirst(ctx, network, addr, listenAddr)
}
```

### dialTwicePreferFirst

```go
func dialTwicePreferFirst(ctx context.Context, network, addr string, listenAddr string) (net.Conn, error) {
    // 1. first: 复用端口
    firstCtx, firstCancel := context.WithCancel(ctx)
    defer firstCancel()
    firstCh := dialAsync(firstCtx, network, addr, listenAddr)  // 复用端口
    
    // 2. 延迟后 second: 不复用
    timer := time.NewTimer(timeout / 3)
    defer timer.Stop()
    
    var secondCh chan dialResult
    select {
    case res := <-firstCh:
        if res.err == nil {
            return res.conn, nil
        }
        // first 失败，启动 second
        secondCh = dialAsync(ctx, network, addr, "")  // 不复用
    case <-timer.C:
        // 超时，启动 second
        secondCh = dialAsync(ctx, network, addr, "")
    }
    
    // 3. 等待 first 或 second
    select {
    case res := <-firstCh:
        if res.err == nil {
            return res.conn, nil
        }
    case res := <-secondCh:
        if res.err == nil {
            return res.conn, nil
        }
    }
    return nil, errors.New("dial failed")
}
```

## tcpConn

```go
type tcpConn struct {
    *tls.Conn
    ...
}

func (c *tcpConn) Close() error {
    return c.Conn.Close()
}

func (c *tcpConn) isLocal() bool {
    return c.lanChecker.IsLan(c.remoteAddr)
}
```

## 优先级

```go
func (d *tcpDialer) Priority() int {
    if d.isLAN {
        return 10  // LAN TCP
    }
    return 30  // WAN TCP
}
```

## LAN 判断

```go
func (d *tcpDialer) isLAN(addr string) bool {
    tcpAddr, _ := net.ResolveTCPAddr("tcp", addr)
    return d.lanChecker.IsLan(tcpAddr.IP)
}
```

## NAT 映射

TCP 监听器注册 NAT 映射：

```go
mapping := l.natService.NewMapping("tcp", l.listenPort)
// NAT 服务自动创建 UPnP/PMP 映射
// 外部地址通知发现子系统
```

## 地址变更

```go
func (l *tcpListener) onAddressChanged() {
    // 1. 重新注册 NAT 映射
    // 2. 通知发现子系统
    // 3. 触发事件
}
```

## TLS 配置

```go
tlsCfg := &tls.Config{
    InsecureSkipVerify: true,  // 设备 ID 自验证
    MinVersion:         tls.VersionTLS12,
    NextProtos:         []string{"bep/1.0"},  // ALPN
    Certificates:       []tls.Certificate{cert},
}
```

## 超时

```go
const tlsHandshakeTimeout = 10 * time.Second

// 拨号超时
ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
defer cancel()
```

## 测试

`lib/connections/tcp_dial_test.go`、`tcp_listen_test.go`：

- 拨号/监听测试
- 端口复用测试
- NAT 映射测试

## 设计权衡

### 端口复用 vs 不复用

**选择**：双拨号，优先复用。
**优势**：复用端口有助于 NAT 穿透；不复用作为后备。
**劣势**：增加复杂度。

### TLS vs 明文

**选择**：强制 TLS。
**优势**：加密和认证。
**劣势**：握手开销。
