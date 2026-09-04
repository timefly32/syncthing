# Protocol 线格式归一化

## wireFormatConnection

`lib/protocol/wireformat.go`（39 行）实现线格式归一化。

## 职责

- **NFC 归一化**：文件名转为 NFC（Normalization Form Canonical Composition）
- **斜杠转换**：本地路径分隔符转为正斜杠
- **folder-relative**：确保路径是 folder 相对的

## 实现

```go
type wireFormatConnection struct {
    Connection
}

func (c *wireFormatConnection) Index(deviceID protocol.DeviceID, folder string, files []protocol.FileInfo) {
    c.toWire(files)
    c.Connection.Index(deviceID, folder, files)
}

func (c *wireFormatConnection) IndexUpdate(deviceID protocol.DeviceID, folder string, files []protocol.FileInfo) {
    c.toWire(files)
    c.Connection.IndexUpdate(deviceID, folder, files)
}
```

## toWire

```go
func (c *wireFormatConnection) toWire(files []protocol.FileInfo) {
    for i := range files {
        files[i].Name = normalizeWire(files[i].Name)
    }
}

func normalizeWire(name string) string {
    // 1. NFC 归一化
    name = norm.NFC.String(name)
    // 2. 转为正斜杠
    name = filepath.ToSlash(name)
    // 3. 清理路径
    name = path.Clean(name)
    return name
}
```

## fromWire

入站方向由 `nativeModel` 处理（平台特定）：

### Unix

```go
// nativemodel_unix.go
// wire 格式（NFC + /）即本地格式，直通
```

### macOS

```go
// nativemodel_darwin.go
// macOS 使用 NFD，需要 NFC → NFD 转换
func (m *nativeModel) Index(deviceID protocol.DeviceID, folder string, files []protocol.FileInfo) {
    for i := range files {
        files[i].Name = norm.NFD.String(files[i].Name)
    }
    m.model.Index(deviceID, folder, files)
}
```

### Windows

```go
// nativemodel_windows.go
// Windows 使用反斜杠
func (m *nativeModel) Index(deviceID protocol.DeviceID, folder string, files []protocol.FileInfo) {
    files = m.fixupFiles(files)
    m.model.Index(deviceID, folder, files)
}

func (m *nativeModel) fixupFiles(files []protocol.FileInfo) []protocol.FileInfo {
    var result []protocol.FileInfo
    for _, f := range files {
        if strings.Contains(f.Name, `\`) {
            // 含反斜杠的条目被丢弃
            if !f.Deleted {
                l.Warnf("dropping file with backslash: %s", f.Name)
            }
            continue
        }
        f.Name = filepath.FromSlash(f.Name)
        result = append(result, f)
    }
    return result
}
```

## 安全考虑

### 路径遍历防护

`checkFilename`（`protocol.go:662-686`）：

- `path.Clean(name)` 必须等于原 name
- 禁止 `""`、`.`、`..`
- 禁止以 `/` 开头（folder-relative）
- 禁止以 `../` 开头

### 归一化攻击防护

NFC 归一化确保：

- 不同 Unicode 表示的相同字符被统一
- 防止通过不同 Unicode 表示绕过忽略规则

## 设计权衡

### NFC vs NFD

**选择**：wire 格式使用 NFC。
**优势**：NFC 是大多数系统的默认形式。
**劣势**：macOS 使用 NFD，需要转换。

### 正斜杠 vs 本地分隔符

**选择**：wire 格式使用正斜杠。
**优势**：跨平台一致。
**劣势**：Windows 需要转换。

### folder-relative vs 绝对路径

**选择**：folder-relative。
**优势**：防止路径遍历，简化处理。
**劣势**：需要确保路径不以 `/` 开头。
