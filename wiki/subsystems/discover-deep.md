# 发现机制深入

## 本地发现协议详细

`lib/discover/local.go`：

### 广播包格式

```
[4字节 magic: 0x2EA7D90B][32字节 DeviceID][2字节 地址数]
  每个地址:
    [1字节 地址长度][地址字符串]
```

### 广播地址

- **IPv4**：`255.255.255.255:21027`
- **IPv6**：`[ff12::8384]:21027`

### 广播周期

`broadcastInterval = 30 秒`

### 地址列表内容

```go
addresses := []string{
    "tcp://0.0.0.0:22000",
    "quic://0.0.0.0:22000",
    // NAT 映射的外部地址
}
```

### 缓存

```go
type localDiscovery struct {
    cache map[protocol.DeviceID]localEntry
    ...
}

type localEntry struct {
    addresses []string
    lastSeen  time.Time
}
```

- 缓存发现的设备
- `cacheTimeout = 5 分钟`（超过则移除）

## 全局发现协议详细

`lib/discover/global.go`：

### 注册

```http
POST https://discovery.syncthing.net/v2/deviceid
Content-Type: application/json

{
    "addresses": ["tcp://1.2.3.4:22000", "quic://1.2.3.4:22000"]
}
```

### 查询

```http
GET https://discovery.syncthing.net/v2/deviceid
```

响应：

```json
{
    "addresses": ["tcp://1.2.3.4:22000", "quic://1.2.3.4:22000"],
    "last_seen": "2024-01-01T00:00:00Z"
}
```

### 认证

- TLS 客户端证书（设备 ID）
- 无需额外认证

### 注册周期

`registerInterval = 30 分钟`

## 发现器组合

`lib/syncthing` 的 `discoverer`：

```go
type discoverer struct {
    finders []Finder
    ...
}

func (d *discoverer) Lookup(deviceID protocol.DeviceID) ([]string, error) {
    var addresses []string
    for _, finder := range d.finders {
        addrs, _ := finder.Lookup(deviceID)
        addresses = append(addresses, addrs...)
    }
    return deduplicate(addresses), nil
}
```

### 查找顺序

1. 本地发现（快速）
2. 全局发现（慢）
3. 配置的静态地址

### 结果合并

- 合并所有 Finder 的结果
- 去重
- 按优先级排序

## 集成

### 连接服务

`lib/connections/service.go`：

1. 拨号前查询发现器
2. 获取目标设备地址
3. 添加到拨号队列

### 配置变更

- 启用/禁用本地发现
- 启用/禁用全局发现
- 自定义 Discovery Server

## 缓存策略

### 本地发现缓存

- `cacheTimeout = 5 分钟`
- 每次接收广播更新缓存
- 超时移除

### 全局发现缓存

- 查询结果缓存
- `cacheTimeout = 10 分钟`
- 失败时使用缓存

## 错误处理

### 本地发现

- 广播失败：记录日志，继续监听
- 接收失败：记录日志，继续广播

### 全局发现

- 注册失败：重试
- 查询失败：返回缓存或空
- 服务器不可用：回退到本地发现

## 测试

`local_test.go`：

- 广播/接收测试
- 缓存测试
- 多设备测试

`global_test.go`：

- 注册/查询测试
- 错误处理测试
- 缓存测试

## 设计权衡

### 主动注册 vs 被动发现

**选择**：全局发现主动注册，本地发现被动广播。
**优势**：全局发现减少服务器查询负载；本地发现低延迟。
**劣势**：全局发现需要周期性注册。

### 缓存 vs 实时查询

**选择**：缓存查询结果。
**优势**：减少网络请求。
**劣势**：可能使用过期地址。
