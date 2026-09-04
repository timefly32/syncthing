# OS 工具模块

## 概述

`lib/osutil/` 提供跨平台操作系统工具函数，约 800 行 Go 代码。

## 职责

- **进程管理**：设置进程组、信号处理
- **文件操作**：跨平台文件工具
- **网络工具**：DNS、地址解析
- **用户/组**：权限降级

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `osutil.go` | ~200 | 通用工具 |
| `lowprio.go` | ~100 | 低优先级设置 |
| `users_unix.go` | ~100 | Unix 用户/组 |
| `users_windows.go` | ~50 | Windows 用户 |
| `notfound.go` | ~50 | 文件未找到处理 |

## 进程管理

### SetPriority

降低 Syncthing 进程优先级，避免影响系统响应：

- Unix：`setpriority`/`nice`
- Windows：`SetPriorityClass`

### 信号处理

`InstallSignalHandler` 安装信号处理器：

- `SIGTERM`/`SIGINT`：优雅关闭
- `SIGHUP`：重启
- `SIGUSR1`：刷新配置

## 文件操作

### Copy

跨平台文件复制，保留权限。

### RenameOrCopy

先尝试 `rename`，失败则 `copy + delete`（跨设备）。

### IsDeleted

检查文件是否已删除（处理竞态）。

## 网络

### DNS

`LookupHost`、`LookupIP` 包装，处理平台差异。

### Address

地址解析和格式化工具。

## 用户/组

### Unix

- `DropPrivileges`：降权到指定用户/组
- `GetGroups`：获取用户组列表

### Windows

- 有限支持
- 主要用于服务账户

## 低优先级 I/O

`lowprio.go`：

- Unix：`ionice` 设置 I/O 优先级
- Windows：`SetThreadPriority`

## 测试覆盖

`osutil_test.go`、`lowprio_test.go`：

- 文件操作测试
- 信号处理测试

## 设计权衡

### 跨平台抽象

**选择**：统一接口，平台特定实现。
**优势**：上层代码平台无关。
**劣势**：部分功能平台受限。

### 降权运行

**选择**：支持降权到非 root。
**优势**：安全性。
**劣势**：需要正确配置。
