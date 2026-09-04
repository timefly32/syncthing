# 开发环境搭建

## 前置要求

- **Go**：1.26.2 或更高（见 `go.mod`）
- **Git**：用于克隆仓库和贡献代码
- **Make**（可选）：简化构建命令
- **Node.js**（可选）：构建 Web UI

## 克隆仓库

```bash
git clone https://github.com/syncthing/syncthing.git
cd syncthing
```

## 构建命令

Syncthing 使用自定义构建脚本 `build.go`：

```bash
# 构建 syncthing 二进制
go run build.go

# 构建所有组件
go run build.go all

# 构建 Web UI 资源
go run build.go assets

# 交叉编译
go run build.go -goos linux -goarch arm64
```

## 运行测试

```bash
# 运行所有测试
go run build.go test

# 运行特定包测试
go run build.go test ./lib/protocol/...

# 带竞态检测
go run build.go test -race

# 运行基准测试
go run build.go bench ./lib/protocol/...
```

## 运行 Syncthing

```bash
# 构建并运行
go run build.go && ./syncthing

# 开发模式（启用调试日志）
STTRACE=model STRESTART=0 ./syncthing

# 指定主目录
./syncthing --home=/tmp/syncthing-dev
```

## 调试

### 启用调试日志

```bash
# 单个子系统
STTRACE=model ./syncthing

# 多个子系统
STTRACE=model,protocol,connections ./syncthing

# 所有子系统
STTRACE=* ./syncthing
```

可用子系统：`model`, `protocol`, `connections`, `discover`, `relay`, `scanner`, `config`, `db`, `events`, `api`, `fs`, `ignore`, `versioner`, `nat`

### 重启行为

```bash
# 开发时禁用自动重启
STRESTART=0 ./syncthing
```

### Web UI

默认 `http://127.0.0.1:8384`，开发时可用：

```bash
# 监听所有接口
./syncthing -gui-address=0.0.0.0:8384
```

## IDE 配置

### VS Code

`.vscode/settings.json` 示例：

```json
{
    "go.buildTags": "",
    "go.testFlags": ["-race"],
    "go.lintTool": "golangci-lint"
}
```

### GoLand

- 设置 Go 版本为 1.26.2
- 配置运行配置使用 `go run build.go`
- 启用竞态检测器

## 代码风格

- 遵循 [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- 使用 `gofmt` 格式化
- 使用 `goimports` 管理导入
- 运行 `go vet` 检查

## 贡献流程

1. Fork 仓库
2. 创建特性分支
3. 编写代码和测试
4. 运行 `go run build.go test`
5. 提交 PR

### 提交消息

遵循 [Conventional Commits](https://www.conventionalcommits.org/)：

```
feat: add new feature
fix: fix bug
docs: update documentation
refactor: refactor code
test: add tests
```

## 常见问题

### 构建失败

```bash
# 清理构建缓存
go clean -cache
go run build.go
```

### 测试失败

```bash
# 查看详细输出
go run build.go test -v ./lib/protocol/...
```

### 依赖问题

```bash
# 更新依赖
go mod tidy
go mod download
```
