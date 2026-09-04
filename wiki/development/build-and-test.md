# 构建与测试

## 构建系统

Syncthing 使用自定义 `build.go` 脚本（约 600 行）作为构建入口，包装 `go build`/`go test` 命令。

### build.go 命令

| 命令 | 说明 |
| --- | --- |
| `go run build.go` | 构建 `syncthing` 二进制 |
| `go run build.go all` | 构建所有二进制（syncthing, strelaysrv, strelaysrv, etc.） |
| `go run build.go assets` | 构建 Web UI 资源 |
| `go run build.go test` | 运行测试 |
| `go run build.go bench` | 运行基准测试 |
| `go run build.go proto` | 重新生成 protobuf 代码 |
| `go run build.go translate` | 更新翻译 |
| `go run build.go lint` | 运行 linter |
| `go run build.go vet` | 运行 go vet |

### 构建标志

| 标志 | 说明 |
| --- | --- |
| `-goos` | 目标 OS |
| `-goarch` | 目标架构 |
| `-version` | 版本字符串 |
| `-no-deadlock` | 禁用死锁检测 |
| `-race` | 启用竞态检测 |

## 构建标签

| 标签 | 说明 |
| --- | --- |
| `noembedassets` | 不嵌入 Web UI 资源 |
| `cgo` | 启用 cgo（SQLite） |
| `nocgo` | 禁用 cgo（纯 Go SQLite） |
| `purego` | 纯 Go（无 cgo） |

## 依赖管理

`go.mod` 管理依赖：

- 模块路径：`github.com/syncthing/syncthing`
- Go 版本：1.26.2
- 关键依赖：
  - `quic-go/quic-go` — QUIC 支持
  - `mattn/go-sqlite3` — SQLite（cgo）
  - `modernc.org/sqlite` — SQLite（纯 Go）
  - `miscreant/miscreant.go` — AES-SIV
  - `golang.org/x/crypto` — 加密原语
  - `prometheus/client_golang` — 指标

## 测试

### 单元测试

```bash
# 所有测试
go run build.go test

# 特定包
go run build.go test ./lib/protocol/...

# 带竞态检测
go run build.go test -race

# 详细输出
go run build.go test -v ./lib/model/...
```

### 集成测试

`test/` 目录包含集成测试：

```bash
# 运行集成测试
cd test && go test -v

# 端到端测试
go run build.go test -tags integration
```

### 基准测试

```bash
# 运行基准测试
go run build.go bench ./lib/protocol/...

# 带内存分配
go run build.go bench -benchmem ./lib/protocol/...
```

### 覆盖率

```bash
# 生成覆盖率报告
go run build.go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

## CI/CD

GitHub Actions 工作流：

- `.github/workflows/build.yml` — 构建和测试
- `.github/workflows/release.yml` — 发布
- `.github/workflows/lint.yml` — 代码检查

### CI 矩阵

- OS：Linux, macOS, Windows
- 架构：amd64, arm64
- Go 版本：1.26.2

## 代码质量

### Lint

```bash
# 运行 linter
go run build.go lint

# golangci-lint
golangci-lint run
```

### Vet

```bash
go run build.go vet
```

### 死锁检测

构建时默认启用死锁检测（`go-deadlock` 库），可通过 `-no-deadlock` 禁用。

## Web UI 构建

```bash
# 构建前端资源
cd gui
npm install
npm run build

# 或通过 build.go
go run build.go assets
```

## Protobuf 生成

```bash
# 重新生成 BEP protobuf
go run build.go proto
```

生成代码到 `internal/gen/bep/bep.pb.go`。

## 交叉编译

```bash
# Linux ARM64
go run build.go -goos linux -goarch arm64

# Windows
go run build.go -goos windows -goarch amd64

# macOS
go run build.go -goos darwin -goarch arm64
```

## 发布

```bash
# 构建发布版本
go run build.go -version "v1.27.0" all

# 生成校验和
sha256sum syncthing-* > sha256sum.txt
```
