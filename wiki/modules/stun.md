# STUN 模块

## 概述

`lib/stun/` 实现 STUN（Session Traversal Utilities for NAT）协议，约 1000 行 Go 代码。用于 NAT 类型检测和外部地址发现。

## 职责

- **NAT 类型检测**：识别 NAT 行为类型
- **外部地址发现**：获取 NAT 后的公网地址
- **保活**：维持 NAT 映射

## NAT 类型

| 类型 | 说明 |
| --- | --- |
| `NATUnknown` | 未知 |
| `NATNone` | 无 NAT |
| `NATBlocked` | 阻塞 |
| `NATSymmetric` | 对称 NAT（难以穿透） |
| `NATFullCone` | 完全锥形 |
| `NATRestricted` | 限制锥形 |
| `NATPortRestricted` | 端口限制锥形 |

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `stun.go` | ~300 | STUN 协议实现 |
| `nat.go` | ~200 | NAT 类型检测 |
| `keepalive.go` | ~100 | 保活逻辑 |

## STUN 协议

### 消息类型

| 类型 | 说明 |
| --- | --- |
| `BindingRequest` | 绑定请求 |
| `BindingResponse` | 绑定响应 |

### 属性

| 属性 | 说明 |
| --- | --- |
| `MappedAddress` | 映射地址 |
| `XorMappedAddress` | XOR 映射地址 |
| `Software` | 软件标识 |
| `ChangeRequest` | 变更请求 |

## NAT 检测算法

1. 发送 BindingRequest 到 STUN 服务器
2. 检查响应中的 MappedAddress
3. 从不同服务器发送，比较映射地址
4. 根据一致性判断 NAT 类型

## 保活

`keepalive.go`：

- 周期性发送 STUN 请求
- 维持 NAT 映射
- 默认间隔 `stunKeepaliveS`（配置）

## 集成

### QUIC 监听

`lib/connections/quic_listen.go`：

- 使用 STUN 检测 NAT 类型
- 存储 `nat atomic.Uint64`
- 影响 QUIC 连接行为

### 连接服务

`lib/connections/service.go`：

- STUN 保活维持映射
- 外部地址用于发现

## 配置

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `stunKeepaliveS` | 0（禁用） | STUN 保活间隔 |

## 测试覆盖

`stun_test.go`、`nat_test.go`：

- 协议消息测试
- NAT 检测测试

## 设计权衡

### STUN vs UPnP

**选择**：两者都支持。
**优势**：STUN 无需路由器支持；UPnP 主动映射。
**劣势**：STUN 对对称 NAT 无效。

### 保活间隔

**选择**：可配置，默认禁用。
**优势**：用户控制网络流量。
**劣势**：禁用时映射可能过期。
