# 仓库地图

本页提供 Syncthing 仓库的顶层目录概览，标注每个区域的职责、重要性和编辑时机。

## 顶层目录

| 目录 | 文件数 | 职责 | 重要性 | 何时编辑 |
| --- | ---: | --- | --- | --- |
| `cmd/` | 91 | 可执行入口：主程序、CLI、密钥生成、解密工具、基础设施服务 | 核心 | 修改启动流程、CLI 命令 |
| `lib/` | 413 | 核心库：协议、模型、配置、连接、发现、扫描等 | 核心 | 修改同步逻辑、协议、配置 |
| `internal/` | 75 | 内部包：数据库、protobuf 生成代码、日志、迭代工具 | 核心 | 修改数据库、生成代码 |
| `gui/` | 136 | Web GUI 静态资源（HTML/JS/CSS/翻译） | 中等 | 修改 Web 界面 |
| `test/` | 41 | 端到端集成测试 | 高 | 验证同步行为变更 |
| `etc/` | 23 | 系统集成配置：systemd、launchd、FreeBSD RC 等 | 低 | 添加平台服务配置 |
| `man/` | 18 | Unix man 手册页 | 低 | 修改 CLI 选项 |
| `assets/` | 21 | 构建时嵌入的资源（图标、证书） | 低 | 替换图标 |
| `proto/` | 5 | protobuf 定义（BEP、发现、API、数据库） | 高 | 修改协议消息 |
| `script/` | 13 | 辅助脚本（作者列表、版权检查等） | 低 | 维护流程 |
| `relnotes/` | 5 | 发布说明 | 低 | 发布版本 |
| `meta/` | 4 | 元数据（构建触发器等） | 低 | 构建配置 |
| `.github/` | 18 | GitHub Actions 工作流、Issue 模板 | 中等 | 修改 CI/CD |

## 核心目录详解

### `cmd/` — 可执行入口

| 子目录 | 职责 |
| --- | --- |
| `cmd/syncthing/` | 主程序入口，包含 `main.go`（命令行解析）、`monitor.go`（子进程监控）、`crash_reporting.go`（崩溃报告）、`cli/`（CLI 命令）、`generate/`（密钥生成）、`decrypt/`（加密文件夹解密） |
| `cmd/stdiscosrv/` | 全局发现服务器 |
| `cmd/strelaysrv/` | 中继服务器 |
| `cmd/infra/` | 基础设施工具（崩溃接收、升级服务器等） |
| `cmd/dev/` | 开发工具 |

### `lib/` — 核心库

`lib/` 是最大的目录，包含 43 个子包。按职责分组：

**同步核心**

| 包 | 职责 | 关键文件 |
| --- | --- | --- |
| `lib/model/` | 中央同步协调器，管理文件夹生命周期和连接路由 | `model.go` (3486行), `folder_sendrecv.go` (2253行) |
| `lib/protocol/` | BEP 协议实现，连接、消息路由、版本向量、加密 | `protocol.go` (1175行), `encryption.go` (698行) |
| `lib/scanner/` | 文件系统遍历和块哈希计算 | `walk.go`, `blocks.go`, `blockqueue.go` |
| `lib/config/` | 配置管理、版本迁移、订阅通知 | `config.go`, `wrapper.go`, `migrations.go` |
| `lib/syncthing/` | 主 Service 容器，组件组装和启动 | `syncthing.go` (478行) |

**网络与连接**

| 包 | 职责 |
| --- | --- |
| `lib/connections/` | TCP/QUIC/Relay 连接管理、拨号、监听、限速 |
| `lib/discover/` | 全局发现（HTTPS）和本地发现（UDP 广播） |
| `lib/relay/` | 中继客户端和协议 |
| `lib/dialer/` | 拨号器，支持代理和端口复用 |
| `lib/beacon/` | UDP 广播/多播信标 |
| `lib/nat/` | NAT 穿透服务 |
| `lib/upnp/` | UPnP IGD 发现和端口映射 |
| `lib/pmp/` | NAT-PMP 协议 |
| `lib/stun/` | STUN NAT 类型发现 |
| `lib/netutil/` | 网络工具 |

**文件与存储**

| 包 | 职责 |
| --- | --- |
| `lib/fs/` | 文件系统抽象（basicfs、fakefs、walkfs、casefs 等） |
| `lib/ignore/` | `.stignore` 模式匹配 |
| `lib/versioner/` | 文件版本管理（simple、staggered、trashcan、external） |
| `lib/osutil/` | OS 工具（原子写入、重命名、隐藏文件） |

**服务与工具**

| 包 | 职责 |
| --- | --- |
| `lib/api/` | REST HTTP API 和 Web GUI 服务 |
| `lib/events/` | 事件系统（30+ 事件类型，位掩码订阅） |
| `lib/stats/` | 文件夹和设备统计 |
| `lib/ur/` | 使用报告和失败报告 |
| `lib/upgrade/` | 自动升级（签名验证） |
| `lib/locations/` | 配置/数据/日志路径定位 |
| `lib/tlsutil/` | TLS 配置和证书生成 |
| `lib/svcutil/` | suture 服务框架工具 |
| `lib/build/` | 版本信息和平台标志 |
| `lib/semaphore/` | 信号量并发控制 |
| `lib/syncutil/` | 同步原语工具 |

### `internal/` — 内部包

| 子目录 | 职责 |
| --- | --- |
| `internal/db/` | 数据库接口层和类型安全访问 |
| `internal/db/sqlite/` | SQLite 后端实现（per-folder 数据库） |
| `internal/db/olddb/` | 旧 LevelDB 后端（仅用于迁移） |
| `internal/gen/bep/` | BEP protobuf 生成代码 |
| `internal/gen/dbproto/` | 数据库 protobuf 生成代码 |
| `internal/gen/discoproto/` | 发现协议 protobuf 生成代码 |
| `internal/gen/apiproto/` | API protobuf 生成代码 |
| `internal/protoutil/` | protobuf 序列化辅助 |
| `internal/slogutil/` | 结构化日志（slog）初始化和适配 |
| `internal/blob/` | Blob 存储工具 |
| `internal/itererr/` | 迭代器错误收集 |
| `internal/timeutil/` | 时间工具 |

### `proto/` — Protobuf 定义

| 文件 | 职责 |
| --- | --- |
| `proto/bep/bep.proto` | Block Exchange Protocol 消息定义（262行） |
| `proto/discoproto/` | 发现协议消息 |
| `proto/dbproto/` | 数据库序列化消息 |
| `proto/apiproto/` | API 消息 |
| `proto/discosrv/` | 发现服务器消息 |

### `test/` — 集成测试

端到端测试目录，测试完整的同步行为：

| 文件 | 测试内容 |
| --- | --- |
| `test/sync_test.go` | 基本同步 |
| `test/transfer-bench_test.go` | 传输性能基准 |
| `test/ignore_test.go` | 忽略规则 |
| `test/conflict_test.go` | 冲突处理 |
| `test/reconnect_test.go` | 重连 |
| `test/scan_test.go` | 扫描 |
| `test/symlink_test.go` | 符号链接 |
| `test/http_test.go` | HTTP API |
| `test/manypeers_test.go` | 多设备 |

## 语言与构建

| 语言/类型 | 文件数 | 说明 |
| --- | ---: | --- |
| Go | 540 | 主要语言 |
| JSON | 57 | GUI 翻译、测试数据 |
| HTML | 38 | GUI 模板 |
| Markdown | 27 | 文档 |
| JavaScript | 26 | GUI 脚本 |
| XML | 25 | GUI 配置 |
| YAML | 22 | CI/CD、配置 |
| SQL | 14 | 数据库 schema |
| CSS | 8 | GUI 样式 |
| Shell | 6 | 构建脚本 |
| Proto | 5 | 协议定义 |

## 构建系统

- **构建入口**：`build.sh` → `go run build.go`
- **构建标签**：`tools`（build.go）、`noupgrade`（禁用升级）、`ios`（iOS 限制）
- **版本注入**：ldflags 注入 `lib/build` 包变量
- **交叉编译**：`build.go` 支持 `goos`/`goarch` 参数
- **打包**：tar、zip、deb 格式

## 生成与第三方代码

- `internal/gen/` — protobuf 生成代码，由 `buf.gen.yaml` 配置生成，**不要手动编辑**
- `lib/protocol/mocks/` — counterfeiter 生成的 mock，**不要手动编辑**
- `lib/config/mocks/` — counterfeiter 生成的 mock
- `lib/build/runtimeos.gen.go` — 由 `runtimeos.sh` 生成
- `gui/` — 包含翻译文件（`.json`），由 Weblate 维护

## 配置文件

| 文件 | 职责 |
| --- | --- |
| `go.mod` / `go.sum` | Go 模块依赖 |
| `.golangci.yml` | golangci-lint 配置 |
| `buf.yaml` / `buf.gen.yaml` | protobuf 生成配置 |
| `.policy.yml` | 策略配置 |
| `compat.yaml` | 兼容性配置 |
| `Dockerfile*` | 多个 Docker 构建文件 |
