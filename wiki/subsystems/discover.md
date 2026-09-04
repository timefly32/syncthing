# 发现子系统

## 概述

`lib/discover/` 实现设备发现机制，帮助设备找到彼此的网络地址。支持本地和全局两种发现方式。

## 发现类型

| 类型 | 文件 | 协议 | 范围 |
| --- | --- | --- | --- |
| 本地发现 | `local.go` | UDP 广播/multicast | 同一局域网 |
| 全局发现 | `global.go` | HTTPS（Discovery Server） | 互联网 |

## 本地发现

`lib/discover/local.go`（~400 行）：

### 协议

- **IPv4**：UDP 广播到 `255.255.255.255:21027`
- **IPv6**：multicast 到 `[ff12::8384]:21027`
- 包格式：`magic(4B) + DeviceID(32B) + 地址列表`

### 工作流程

1. 监听 UDP 端口 21027
2. 周期性广播自身地址（`broadcastInterval = 30s`）
3. 接收其他设备的广播
4. 缓存发现的设备地址
5. 通过 `Found` channel 通知

### 地址列表

每个广播包含设备的多个地址：
- TCP 监听地址
- QUIC 监听地址
- 本地 IP 地址

## 全局发现

`lib/discover/global.go`（~300 行）：

### 协议

- HTTPS 请求到 Discovery Server（`https://discovery.syncthing.net/`）
- 设备注册自身地址
- 查询其他设备地址

### 工作流程

1. 周期性注册自身地址到 Discovery Server
2. 需要时查询目标设备的地址
3. 缓存查询结果
4. 通过 `Found` channel 通知

### Discovery Server

- 默认：`https://discovery.syncthing.net/`
- 可配置自定义服务器
- 设备 ID 作为查询键

## 发现接口

```go
type Finder interface {
    Lookup(deviceID protocol.DeviceID) ([]string, error)
    Error() error
    String() string
    Cache() map[protocol.DeviceID][]string
}

type Server interface {
    RegisterDevice(deviceID protocol.DeviceID, addr string)
    UnregisterDevice(deviceID protocol.DeviceID)
}
```

## 集成

`lib/syncthing` 中的 `discoverer` 组合本地和全局发现：

1. `NewDiscoverer` 创建组合发现器
2. 添加本地和全局 Finder
3. `Lookup` 查询所有 Finder，合并结果
4. 发现的地址传递给连接子系统

## 配置

| 配置项 | 说明 |
| --- | --- |
| `Options.GlobalAnnounceEnabled` | 启用全局发现 |
| `Options.LocalAnnounceEnabled` | 启用本地发现 |
| `Options.GlobalAnnounceServers` | 自定义 Discovery Server |

## 测试覆盖

`local_test.go`、`global_test.go`：

- 广播/接收测试
- 地址解析测试
- 缓存测试

## 设计权衡

### 本地 vs 全局

**选择**：两者都支持。
**优势**：本地发现快速且无需服务器；全局发现跨网络。
**劣势**：本地发现仅限局域网；全局发现依赖服务器。

### UDP 广播 vs multicast

**选择**：IPv4 用广播，IPv6 用 multicast。
**优势**：兼容性。
**劣势**：广播在某些网络被禁用。
