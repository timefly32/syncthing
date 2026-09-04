# Syncthing 维护者 Wiki

## 关于本 Wiki

本 Wiki 是为 Syncthing 代码库的代码学习者和新维护者编写的深度技术文档。Syncthing 是一个持续文件同步程序，在两台或多台计算机之间同步文件。项目以 Go 语言编写，约 12 万行代码，采用 MPLv2 许可证。

本 Wiki 独立于官方用户文档和 `docs/` 目录，专注于代码架构、子系统设计、运行时流程和维护者指南。所有内容基于源代码分析，引用具体文件路径和行号。

## 受众

- **代码学习者**：希望理解 Syncthing 内部工作原理的开发者
- **新维护者**：需要在大型代码库中快速定位修改点的贡献者
- **压力下的维护者**：需要快速排查问题或评估变更影响的开发者

## 推荐阅读路径

### 首次阅读（理解整体架构）

1. [仓库地图](repository-map.md) — 顶层目录和包概览
2. [架构概览](architecture/overview.md) — 系统边界、组件关系、依赖方向
3. [运行时流程](architecture/runtime-flows.md) — 端到端同步路径
4. [依赖图](architecture/dependency-graph.md) — 模块依赖关系

### 功能贡献者（修改同步逻辑）

1. [Model 子系统](subsystems/model/index.md) — 核心同步协调器
2. [Protocol 子系统](subsystems/protocol/index.md) — BEP 协议实现
3. [数据库子系统](subsystems/database/index.md) — 文件元数据存储
4. [扫描器子系统](subsystems/scanner.md) — 文件系统遍历和哈希
5. [配置子系统](subsystems/config.md) — 配置管理和迁移

### 网络与连接调试

1. [连接子系统](subsystems/connections/index.md) — TCP/QUIC/Relay 连接管理
2. [发现子系统](subsystems/discover.md) — 全局和本地设备发现
3. [中继子系统](subsystems/relay.md) — 中继协议和客户端
4. [NAT 穿透](subsystems/nat.md) — UPnP/PMP/STUN

### 调试与运维

1. [构建与测试](development/build-and-test.md) — 构建命令和测试策略
2. [配置参考](development/configuration.md) — 运行时配置
3. [维护者手册](maintainer-playbook.md) — 常见变更配方和风险点
4. [故障排查](development/troubleshooting.md) — 调试入口点

### 子系统所有权（深度维护）

1. [Model 深度](subsystems/model/design.md) — 同步设计决策
2. [Model 算法](subsystems/model/algorithms.md) — 拉取调度和冲突解决
3. [Protocol 加密](subsystems/protocol/encryption.md) — 加密文件夹协议
4. [数据库 Schema](subsystems/database/schema.md) — SQLite 表结构

## 执行摘要

Syncthing 的核心架构围绕以下设计中心组织：

- **Block Exchange Protocol (BEP)**：设备间交换文件元数据和数据块的协议，定义于 `lib/protocol/`。使用 TLS 加密，支持版本向量和增量索引。
- **Model**：中央协调器，管理文件夹生命周期、连接路由、索引交换和同步流水线，位于 `lib/model/`。
- **文件夹同步流水线**：扫描 → 索引交换 → 请求拉取 → 块组装 → 提交数据库，由 `folder_sendrecv.go` 编排。
- **多后端连接**：TCP、QUIC、中继三种连接类型，通过统一拨号器/监听器接口抽象，位于 `lib/connections/`。
- **设备发现**：全局发现（HTTPS）和本地发现（UDP 广播/多播）双机制，位于 `lib/discover/`。
- **Per-folder SQLite 数据库**：每个文件夹独立 SQLite 文件存储文件元数据，位于 `internal/db/sqlite/`。
- **配置版本迁移**：从 v10 到 v52 的渐进式迁移机制，位于 `lib/config/migrations.go`。
- **双进程监控架构**：主进程监控子进程，崩溃自动重启，位于 `cmd/syncthing/`。

## 完整页面索引

### 顶层

- [仓库地图](repository-map.md)
- [维护者手册](maintainer-playbook.md)

### 架构

- [架构概览](architecture/overview.md)
- [运行时流程](architecture/runtime-flows.md)
- [依赖图](architecture/dependency-graph.md)

### 子系统

- [Model 子系统](subsystems/model/index.md)
  - [设计](subsystems/model/design.md)
  - [算法](subsystems/model/algorithms.md)
  - [数据流](subsystems/model/data-flow.md)
  - [失败模式](subsystems/model/failure-modes.md)
  - [测试](subsystems/model/testing.md)
  - [维护者笔记](subsystems/model/maintainer-notes.md)
  - [拉取调度深入](subsystems/model/pull-scheduling.md)
  - [扫描调度深入](subsystems/model/scan-scheduling.md)
  - [连接管理深入](subsystems/model/connections.md)
  - [文件夹类型深入](subsystems/model/folder-types.md)
  - [索引交换](subsystems/model/index-exchange.md)
  - [文件同步](subsystems/model/file-sync.md)
- [Protocol 子系统](subsystems/protocol/index.md)
  - [连接与消息路由](subsystems/protocol/connection.md)
  - [版本向量](subsystems/protocol/version-vector.md)
  - [加密](subsystems/protocol/encryption.md)
  - [设备 ID](subsystems/protocol/device-id.md)
  - [消息格式](subsystems/protocol/message-formats.md)
  - [测试](subsystems/protocol/testing.md)
  - [维护者笔记](subsystems/protocol/maintainer-notes.md)
  - [压缩深入](subsystems/protocol/compression.md)
  - [缓冲池深入](subsystems/protocol/bufferpool.md)
  - [线格式归一化](subsystems/protocol/wireformat.md)
  - [错误处理深入](subsystems/protocol/error-handling.md)
- [数据库子系统](subsystems/database/index.md)
  - [Schema](subsystems/database/schema.md)
  - [查询深入](subsystems/database/queries.md)
  - [旧数据库迁移](subsystems/database/migration.md)
  - [维护者笔记](subsystems/database/maintainer-notes.md)
  - [事务深入](subsystems/database/transactions.md)
  - [维护深入](subsystems/database/maintenance.md)
- [连接子系统](subsystems/connections/index.md)
  - [拨号与监听](subsystems/connections/dial-listen.md)
  - [连接建立深入](subsystems/connections/handshake.md)
  - [限速](subsystems/connections/limiter.md)
  - [维护者笔记](subsystems/connections/maintainer-notes.md)
  - [限速深入](subsystems/connections/limiter-deep.md)
  - [QUIC 连接深入](subsystems/connections/quic-deep.md)
  - [TCP 连接深入](subsystems/connections/tcp-deep.md)
- [扫描器子系统](subsystems/scanner.md)
  - [扫描器深入](subsystems/scanner-deep.md)
- [配置子系统](subsystems/config.md)
  - [配置变更深入](subsystems/config-deep.md)
  - [配置迁移深入](subsystems/config-migration.md)
- [发现子系统](subsystems/discover.md)
  - [发现机制深入](subsystems/discover-deep.md)
- [中继子系统](subsystems/relay.md)
  - [中继协议深入](subsystems/relay-deep.md)
- [NAT 穿透](subsystems/nat.md)
  - [NAT 穿透深入](subsystems/nat-deep.md)
- [事件总线深入](subsystems/events-deep.md)
- [事件系统实现深入](subsystems/events-impl.md)
- [API 端点深入](subsystems/api-deep.md)
- [忽略规则深入](subsystems/ignore-deep.md)
- [文件系统深入](subsystems/fs-deep.md)
- [版本控制深入](subsystems/versioner-deep.md)

### 模块

- [API 模块](modules/api.md)
- [事件系统](modules/events.md)
- [文件系统抽象](modules/fs.md)
- [忽略规则](modules/ignore.md)
- [版本管理](modules/versioner.md)
- [使用报告](modules/ur.md)
- [升级](modules/upgrade.md)
- [日志系统](modules/logger.md)
- [TLS 工具](modules/tlsutil.md)
- [OS 工具](modules/osutil.md)
- [信号量](modules/semaphore.md)
- [STUN](modules/stun.md)
- [同步原语](modules/sync.md)
- [顶层服务编排](modules/syncthing.md)

### 开发

- [环境搭建](development/setup.md)
- [构建与测试](development/build-and-test.md)
- [配置](development/configuration.md)
- [发布与运维](development/release-and-operations.md)
- [故障排查](development/troubleshooting.md)

### 参考

- [术语表](reference/glossary.md)
- [文件索引](reference/file-index.md)
- [符号索引](reference/symbol-index.md)
- [开放问题](reference/open-questions.md)

## 版本与构建信息

- **Go 版本**：1.26.2
- **模块路径**：`github.com/syncthing/syncthing`
- **构建入口**：`build.sh` → `go run build.go`
- **版本注入**：通过 ldflags 注入 `lib/build` 包的 `Version`/`Stamp`/`User`/`Host`/`Tags` 变量
- **Codename**：Hafnium Hornet（由 `lib/build` 解析）
