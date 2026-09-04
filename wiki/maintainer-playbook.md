# 维护者手册

## 仓库概览

Syncthing 是一个去中心化的文件同步工具，约 12 万行 Go 代码。

### 核心目录

| 目录 | 说明 |
| --- | --- |
| `cmd/` | 命令入口（syncthing, strelaysrv, strelaysrv, etc.） |
| `lib/` | 核心库（model, protocol, config, connections, etc.） |
| `internal/` | 内部包（db, gen, upgrade, etc.） |
| `proto/` | protobuf 定义 |
| `gui/` | Web UI |
| `test/` | 集成测试 |
| `man/` | 手册页 |
| `script/` | 辅助脚本 |

### 模块依赖方向

```
cmd/syncthing
    └─ lib/syncthing
         ├─ lib/model
         │    ├─ lib/protocol
         │    ├─ lib/scanner
         │    ├─ lib/versioner
         │    ├─ lib/ignore
         │    ├─ lib/fs
         │    └─ internal/db
         ├─ lib/connections
         │    ├─ lib/discover
         │    ├─ lib/relay
         │    └─ lib/nat
         ├─ lib/config
         ├─ lib/api
         ├─ lib/events
         └─ lib/logger
```

## 核心设计原则

### 1. 去中心化

- 无中心服务器
- 设备间直接同步
- 中继服务器仅用于 NAT 穿透

### 2. 最终一致性

- 基于版本向量解决冲突
- 不保证强一致性
- 冲突文件保留供用户处理

### 3. 安全默认

- TLS 加密所有连接
- 设备 ID 自验证（无 PKI）
- 加密文件夹支持不可信设备

### 4. 鲁棒性

- 单个文件错误不中断同步
- 数据库崩溃可重建
- 连接断开自动重连

## 关键决策记录

### 为什么选择 SQLite 而非 LevelDB？

- 完整事务支持（ACID）
- Per-folder 数据库隔离
- SQL 查询能力
- WAL 模式高并发
- 触发器自动维护计数

### 为什么使用版本向量而非时间戳？

- 正确处理并发修改
- 不依赖时钟同步
- 物理时间作为 tiebreaker

### 为什么 ClusterConfig 必须是第一条消息？

- 确保双方知道对方的文件夹和设备
- 防止在配置未知时处理索引
- 简化状态机

### 为什么使用无缓冲 channel？

- 形成背压，防止内存耗尽
- 简化同步语义
- 明确的流量控制

### 为什么 LocalFlags 不上 wire？

- 内部状态不应通过协议传输
- 防止设备间状态污染
- 保持协议简洁

## 贡献指南

### 代码审查清单

#### 通用

- [ ] 代码遵循 Go 风格指南
- [ ] 添加了适当的测试
- [ ] 通过 `go run build.go test`
- [ ] 通过 `go run build.go lint`
- [ ] 提交消息符合规范

#### 协议变更

- [ ] 不破坏向后兼容
- [ ] 更新 `proto/bep/bep.proto`
- [ ] 重新生成 protobuf 代码
- [ ] 添加协议测试
- [ ] 考虑版本协商

#### 数据库变更

- [ ] 添加 Schema 迁移
- [ ] 迁移可逆或向前兼容
- [ ] 更新查询代码
- [ ] 添加迁移测试

#### 配置变更

- [ ] 添加默认值
- [ ] 处理迁移
- [ ] 更新文档
- [ ] 添加配置测试

### 发布检查清单

- [ ] 更新 `VERSION`
- [ ] 更新 `CHANGELOG.md`
- [ ] 运行完整测试套件
- [ ] 构建所有平台二进制
- [ ] 生成校验和
- [ ] 创建 GitHub Release
- [ ] 更新文档
- [ ] 公告

## 常见任务

### 添加新的配置选项

1. 在 `lib/config/optionsconfiguration.go` 添加字段
2. 添加 XML 标签
3. 添加默认值
4. 添加迁移（如需）
5. 在 `lib/api` 添加 API 支持
6. 在 Web UI 添加界面
7. 更新文档

### 添加新的 REST API 端点

1. 在 `lib/api/api.go` 添加处理函数
2. 在路由器注册
3. 添加认证检查
4. 添加 CSRF 检查
5. 添加测试
6. 更新 API 文档

### 添加新的事件类型

1. 在 `lib/events/events.go` 添加 `EventType` 常量
2. 在发布者添加 `Log` 调用
3. 在 Web UI 添加处理
4. 更新文档

### 修改 BEP 协议

1. 修改 `proto/bep/bep.proto`
2. 运行 `go run build.go proto`
3. 在 `lib/protocol/` 添加 wire 转换
4. 在 `protocol.go` 添加分发逻辑
5. 在 `lib/model` 添加处理
6. 添加测试
7. 考虑兼容性

## 长期方向

### 已知技术债务

- `internal/db/olddb/` 仅用于迁移，可考虑移除
- 部分 protobuf 生成代码可优化
- Web UI 可现代化

### 未来方向

- 改进大文件夹性能
- 更好的冲突解决 UI
- 增强加密功能
- 改进移动平台支持
