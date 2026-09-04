# 文件索引

## 顶层目录

| 路径 | 说明 |
| --- | --- |
| `cmd/` | 命令入口 |
| `lib/` | 核心库 |
| `internal/` | 内部包 |
| `proto/` | protobuf 定义 |
| `gui/` | Web UI |
| `test/` | 集成测试 |
| `man/` | 手册页 |
| `script/` | 辅助脚本 |
| `build.go` | 构建脚本 |
| `go.mod`/`go.sum` | Go 模块定义 |

## cmd/

| 路径 | 说明 |
| --- | --- |
| `cmd/syncthing/` | 主程序入口 |
| `cmd/strelaysrv/` | 中继服务器 |
| `cmd/strelaypoolsrv/` | 中继池服务器 |
| `cmd/ursrv/` | 使用率报告服务器 |
| `cmd/stgenfiles/` | 测试文件生成工具 |
| `cmd/stbench/` | 基准测试工具 |
| `cmd/stcrashreceiver/` | 崩溃报告接收器 |

## lib/

### 核心同步

| 路径 | 说明 |
| --- | --- |
| `lib/model/` | 核心同步引擎 |
| `lib/protocol/` | BEP 协议实现 |
| `lib/scanner/` | 文件系统扫描器 |
| `lib/versioner/` | 版本控制 |
| `lib/ignore/` | 忽略规则 |
| `lib/fs/` | 文件系统抽象 |

### 网络

| 路径 | 说明 |
| --- | --- |
| `lib/connections/` | 连接管理 |
| `lib/discover/` | 设备发现 |
| `lib/relay/` | 中继支持 |
| `lib/nat/` | NAT 穿透 |
| `lib/dialer/` | 拨号器 |
| `lib/stun/` | STUN 协议 |

### 配置与管理

| 路径 | 说明 |
| --- | --- |
| `lib/config/` | 配置管理 |
| `lib/syncthing/` | 顶层服务编排 |
| `lib/api/` | REST API |
| `lib/events/` | 事件总线 |
| `lib/logger/` | 日志 |
| `lib/locations/` | 路径管理 |

### 安全

| 路径 | 说明 |
| --- | --- |
| `lib/tlsutil/` | TLS 工具 |
| `lib/rand/` | 随机数 |
| `lib/sha256/` | SHA-256 |

### 辅助

| 路径 | 说明 |
| --- | --- |
| `lib/buildinfo/` | 构建信息 |
| `lib/upgrade/` | 自动升级 |
| `lib/ur/` | 使用率报告 |
| `lib/svcutil/` | 服务工具 |
| `lib/sync/` | 同步原语 |
| `lib/stringsutil/` | 字符串工具 |
| `lib/slicesutil/` | 切片工具 |
| `lib/timeutil/` | 时间工具 |
| `lib/osutil/` | OS 工具 |
| `lib/semaphore/` | 信号量 |
| `lib/merkle/` | Merkle 树 |

## internal/

| 路径 | 说明 |
| --- | --- |
| `internal/db/` | 数据库接口 |
| `internal/db/sqlite/` | SQLite 后端 |
| `internal/db/olddb/` | 旧 LevelDB（仅迁移） |
| `internal/gen/` | protobuf 生成代码 |
| `internal/upgrade/` | 升级实现 |
| `internal/ur/` | 使用率报告实现 |

## proto/

| 路径 | 说明 |
| --- | --- |
| `proto/bep/bep.proto` | BEP 协议定义 |
| `proto/ext/` | protobuf 扩展 |

## test/

| 路径 | 说明 |
| --- | --- |
| `test/` | 集成测试 |
| `test/fuzz/` | 模糊测试 |

## gui/

| 路径 | 说明 |
| --- | --- |
| `gui/default/` | 默认 Web UI |
| `gui/default/assets/` | 静态资源 |
| `gui/default/lang/` | 翻译文件 |
