# 构建信息模块

## 概述

`lib/buildinfo/` 提供构建时注入的信息，约 100 行 Go 代码。

## 职责

- **版本信息**：版本号、构建时间、构建者
- **Codename**：版本代号
- **Go 版本**：构建用的 Go 版本

## 变量

通过 ldflags 注入：

```go
var (
    Version = "unknown-dev"
    Stamp   = "0"
    User    = "unknown-user"
    Host    = "unknown-host"
    Tags    = ""
    Codename = ""
    IsRelease bool
    LongVersion string
)
```

## 注入方式

`build.go` 使用 ldflags：

```bash
go build -ldflags "
    -X github.com/syncthing/syncthing/lib/buildinfo.Version=$VERSION \
    -X github.com/syncthing/syncthing/lib/buildinfo.Stamp=$STAMP \
    -X github.com/syncthing/syncthing/lib/buildinfo.User=$USER \
    -X github.com/syncthing/syncthing/lib/buildinfo.Host=$HOST \
    -X github.com/syncthing/syncthing/lib/buildinfo.Tags=$TAGS
"
```

## 版本解析

`init()` 解析版本信息：

```go
func init() {
    // 解析 Codename
    // 设置 IsRelease
    // 构建 LongVersion
}
```

## API

`/rest/system/version`：

```json
{
    "version": "v1.27.0",
    "longVersion": "syncthing v1.27.0 \"Hafnium Hornet\" (go1.26.2 linux-amd64)",
    "codename": "Hafnium Hornet",
    "isRelease": true
}
```

## 测试

`buildinfo_test.go`：

- 版本解析测试
