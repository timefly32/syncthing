# 拨号与监听

## TCP

### 拨号（`tcp_dial.go`）

- 10 秒超时
- `dialer.DialContextReusePortFunc`：端口复用拨号
- TLS 握手
- LAN/WAN 优先级判断

### 监听（`tcp_listen.go`）

- `net.ListenConfig{Control: dialer.ReusePortControl}`：端口复用
- NAT 映射支持
- 地址变更通知

## QUIC

### 拨号（`quic_dial.go`）

- 使用 `quic-go` 库
- 从 registry 获取或创建 Transport
- 支持 IPv4/IPv6 分离（quic4/quic6）

### 监听（`quic_listen.go`）

- `nat atomic.Uint64` 存储 NAT 类型
- STUN 集成
- `quicConfig`：`MaxIdleTimeout=30s`，`KeepAlivePeriod=15s`

### QUIC 配置（`quic_misc.go`）

```go
quicConfig = &quic.Config{
    MaxIdleTimeout:        30 * time.Second,
    KeepAlivePeriod:       15 * time.Second,
    DisablePathMTUDiscovery: true,  // 某些平台问题
}
```

## Relay

### 拨号（`relay_dial.go`）

通过中继服务器建立连接：

1. `client.GetInvitationFromRelay`：从中继获取邀请
2. `client.JoinSession`：加入会话
3. 返回连接

### 监听（`relay_listen.go`）

注册 relay/dynamic+http/dynamic+https scheme：

1. 创建 `relay.Client`
2. 接收 `SessionInvitation`
3. 建立连接

## 拨号器/监听器工厂接口

### dialerFactory（`structs.go:161`）

```go
type dialerFactory interface {
    New(cfg config.Wrapper, tlsCfg *tls.Config, registry *registry.Registry) genericDialer
    AlwaysWAN() bool
    Valid() bool
    String() string
}
```

### listenerFactory（`structs.go`）

```go
type listenerFactory interface {
    New(cfg config.Wrapper, tlsCfg *tls.Config, registry *registry.Registry) genericListener
    Valid() bool
}
```

### genericDialer

```go
type genericDialer interface {
    Dial(ctx context.Context, id protocol.DeviceID, addr string) (internalConn, error)
    RedialFrequency() time.Duration
    Priority() int
    AllowsMultiConns() bool
}
```

## 端口复用

`lib/dialer/` 实现端口复用拨号：

### DialContextReusePortFunc（`lib/dialer/public.go:107`）

1. 如果配置了代理，直接 `DialContext`
2. 从 registry 获取监听地址
3. `dialTwicePreferFirst`：同时尝试复用端口和不复用，优先复用

### dialTwicePreferFirst（`lib/dialer/public.go:139`）

双拨号策略：

1. 先发起 first 拨号（复用端口）
2. 延迟 `timeout/3` 后发起 second（不复用）
3. first 成功则取消 second
4. first 失败则等待 second

### ReusePortControl（`lib/dialer/control_unix.go`）

- 运行时检测 `SO_REUSEPORT` 支持
- 通过 `unix.SetsockoptInt` 设置

## Registry

`lib/connections/registry/registry.go`（81 行）跟踪监听地址：

- 支持 scheme 兼容匹配（quic:// 可用于 quic4:// 和 quic6://）
- 用于 NAT 端口映射和传出端口稳定

## 已废弃

`deprecated.go` 注册 kcp/kcp4/kcp6 为 `errDeprecated`。
