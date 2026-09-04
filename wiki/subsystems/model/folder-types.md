# Model 文件夹类型深入

## 文件夹类型

| 类型 | 实现 | 说明 |
| --- | --- | --- |
| `SendReceive` | `folder_sendrecv.go` | 发送和接收（默认） |
| `SendOnly` | `folder_rofolder.go` | 仅发送（主设备） |
| `ReceiveOnly` | `folder_recvonly.go` | 仅接收 |
| `ReceiveEncrypted` | `folder_sendrecv.go` | 接收加密 |

## SendReceiveFolder

`lib/model/folder_sendrecv.go`：

### 职责

- 扫描本地文件
- 接收远端索引
- 拉取需要的文件
- 发送本地变更

### 拉取循环

```go
func (f *sendReceiveFolder) pull() {
    // 1. 获取 Need 列表
    need := f.db.GetNeed(f.folder, f.localDeviceID)
    // 2. 排序
    need = f.orderFiles(need)
    // 3. 逐个拉取
    for _, file := range need {
        f.pullFile(file)
    }
}
```

## SendOnlyFolder

`lib/model/folder_rofolder.go`：

### 职责

- 扫描本地文件
- 发送本地变更
- **不拉取远端变更**

### 实现

```go
type roFolder struct {
    *folderRunner
}

func (f *roFolder) pull() {
    // SendOnly 不拉取
    // 但检测远端变更并标记冲突
    need := f.db.GetNeed(f.folder, f.localDeviceID)
    for _, file := range need {
        // 标记为冲突（本地是主，远端不应变更）
        f.markConflict(file)
    }
}
```

### 使用场景

- 主设备（权威源）
- 备份设备
- 防止远端误修改

## ReceiveOnlyFolder

`lib/model/folder_recvonly.go`：

### 职责

- 接收远端变更
- **不发送本地变更**
- 本地变更标记为"receive only changed"

### 实现

```go
type recvonlyFolder struct {
    *folderRunner
}

func (f *recvonlyFolder) scan() {
    // 扫描时检测本地变更
    // 标记为 receiveOnlyChanged
    for _, file := range scanResults {
        if file.IsLocalChange() {
            file.LocalFlags |= protocol.FlagLocalReceiveOnly
        }
    }
}

func (f *recvonlyFolder) pull() {
    // 正常拉取
    need := f.db.GetNeed(f.folder, f.localDeviceID)
    for _, file := range need {
        f.pullFile(file)
    }
}
```

### 使用场景

- 只读共享
- 接收备份
- 防止本地变更传播

## ReceiveEncryptedFolder

使用 `folder_sendrecv.go` 但配置为加密：

### 职责

- 接收加密数据
- **无法解密**
- 仅存储

### 实现

```go
// folder 配置 Type = ReceiveEncrypted
// model 使用 encryptedModel 包装
// 所有数据以加密形式存储
```

### 使用场景

- 不可信设备（云服务器）
- 加密备份
- 隐私保护

## 文件夹类型选择

### 配置

```xml
<folder id="xxx" type="sendreceive">
```

| 值 | 类型 |
| --- | --- |
| `sendreceive` | SendReceive |
| `sendonly` | SendOnly |
| `receiveonly` | ReceiveOnly |
| `receiveencrypted` | ReceiveEncrypted |

### 运行时切换

```go
func (m *Model) SetFolderType(folder string, folderType config.FolderType) {
    // 1. 停止现有 folderRunner
    m.folderRunners[folder].Stop()
    // 2. 创建新 folderRunner
    m.folderRunners[folder] = m.newFolderRunner(folder, folderType)
    // 3. 启动
    m.folderRunners[folder].Start()
}
```

## 共同基类

所有文件夹类型继承 `folderRunner`：

```go
type folderRunner struct {
    folder string
    cfg    config.FolderConfiguration
    db     db.FolderDB
    fs     fs.Filesystem
    ...
}
```

### 共同功能

- 扫描调度
- 索引交换
- 事件通知
- 暂停/恢复

### 差异

- `pull()` 实现不同
- `scan()` 可能不同（ReceiveOnly 标记本地变更）
- 索引处理可能不同

## 测试

`lib/model/folder_sendrecv_test.go`、`folder_rofolder_test.go`、`folder_recvonly_test.go`：

- 各类型特定行为测试
- 类型切换测试
