# 连接维护者笔记

## 安全编辑点

### 添加新的连接类型

1. 实现 `dialerFactory` 和 `listenerFactory` 接口
2. 在 `service.go` 的 `init()` 中注册
3. 添加配置支持（`lib/config/optionsconfiguration.go`）
4. 添加测试

### 修改拨号策略

1. 编辑 `service.go` 的 `connect`/`dialDevices`/`dialParallel`
2. **注意**：并行拨号信号量（`dialMaxParallel=64`）
3. 更新冷却机制如需

### 修改限速

1. 编辑 `limiter.go`
2. **注意**：`singleWriteSize` 影响写粒度
3. 更新 LAN 判断逻辑如需

## 风险区域

### 高风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `service.go` | 死锁 | 拨号循环与连接处理交互复杂 |
| `service.go` | 连接泄漏 | 连接跟踪错误 |
| `limiter.go` | 性能 | 限速配置错误影响吞吐 |

### 中风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `tcp_dial.go` | 端口冲突 | 端口复用错误 |
| `quic_listen.go` | NAT 问题 | NAT 类型检测错误 |
| `relay_dial.go` | 中继失败 | 中继服务器不可用 |

## 常见变更配方

### 修改最大连接数

```go
// service.go
const maxNumConnections = 128  // 修改此值

// 注意：影响内存使用和性能
```

### 修改拨号并行度

```go
// service.go
const dialMaxParallel = 64         // 全局
const dialMaxParallelPerDevice = 8 // 每设备

// 注意：过高可能导致资源耗尽
```

### 添加新的拒绝原因

```go
// service.go
var errMyReason = errors.New("my reason")

// connectionCheckEarly 中添加检查
if myCondition {
    return errMyReason
}
```

## 代码审查清单

修改 `lib/connections/` 时检查：

- [ ] 新连接类型是否正确注册？
- [ ] 拨号超时是否合理？
- [ ] 连接是否正确跟踪和清理？
- [ ] 限速器是否正确应用？
- [ ] 冷却机制是否被绕过？
- [ ] 端口复用是否正确？
- [ ] NAT 穿透是否集成？
- [ ] 是否添加了测试？

## 调试

### 检查连接

```bash
curl http://localhost:8384/rest/system/connections
```

### 启用调试日志

```bash
STTRACE=connections syncthing
```

### 检查监听器

```bash
curl http://localhost:8384/rest/system/status | jq '.listenAddresses'
```

## 性能考虑

### 连接数

- `maxNumConnections=128` 限制总连接数
- 每设备期望连接数协商（取较大值）
- 过多连接增加内存和 CPU 开销

### 拨号频率

- `stdConnectionLoopSleep=60s` 标准间隔
- 冷却机制防止过度重拨
- `dialNow` 信号允许立即拨号

### 限速

- 全局限速影响所有连接
- 每设备限速防止单设备占满带宽
- LAN 不限速优化本地同步
