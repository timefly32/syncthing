# 路径定位模块

## 概述

`lib/locations/` 管理 Syncthing 的所有文件路径，约 200 行 Go 代码。提供平台相关的默认路径和路径覆盖。

## 职责

- **路径管理**：配置、数据库、证书、日志等路径
- **平台适配**：Linux/macOS/Windows 不同路径
- **环境变量**：`STHOMEDIR`/`STCONFIG` 等覆盖

## 路径类型

```go
type Location string

const (
    LocationConfig    Location = "config"
    LocationCert      Location = "cert"
    LocationKey       Location = "key"
    LocationDatabase  Location = "database"
    LocationLogs      Location = "logs"
    LocationPanicLog  Location = "panicLog"
    LocationAuditLog  Location = "auditLog"
    LocationGUIAssets Location = "guiAssets"
    LocationDefFolder Location = "defFolder"
)
```

## 默认路径

### Linux

| 类型 | 路径 |
| --- | --- |
| Config | `~/.local/state/syncthing/config.xml` |
| Cert | `~/.local/state/syncthing/cert.pem` |
| Key | `~/.local/state/syncthing/key.pem` |
| Database | `~/.local/state/syncthing/index-v0.15.0/` |
| Logs | `~/.local/state/syncthing/logs/` |

### macOS

| 类型 | 路径 |
| --- | --- |
| Config | `~/Library/Application Support/Syncthing/config.xml` |
| Database | `~/Library/Application Support/Syncthing/index-v0.15.0/` |

### Windows

| 类型 | 路径 |
| --- | --- |
| Config | `%LOCALAPPDATA%\Syncthing\config.xml` |
| Database | `%LOCALAPPDATA%\Syncthing\index-v0.15.0\` |

## 环境变量

| 变量 | 说明 |
| --- | --- |
| `STHOMEDIR` | 主目录 |
| `STCONFIG` | 配置文件路径 |
| `STCERT` | 证书路径 |
| `STKEY` | 密钥路径 |
| `STGUIASSETS` | GUI 资源路径 |

## 初始化

`SetBaseDir(baseDir string)` 设置主目录，所有路径基于此计算。

## 获取路径

```go
func Get(location Location) string {
    return locations[location]
}
```

## 测试

`locations_test.go`：

- 平台路径测试
- 环境变量覆盖测试
