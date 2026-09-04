# 配置迁移深入

## 迁移机制

`lib/config/config.go` 的 `migrateIfNeeded`：

```go
func migrateIfNeeded(cfg *Configuration) error {
    for cfg.Version < CurrentVersion {
        migrator, ok := migrations[cfg.Version+1]
        if !ok {
            return fmt.Errorf("no migrator for version %d", cfg.Version+1)
        }
        if err := migrator(cfg); err != nil {
            return err
        }
        cfg.Version++
    }
    return nil
}
```

## 迁移函数

```go
var migrations = map[int]func(*Configuration) error{
    11: migrateToV11,
    12: migrateToV12,
    ...
    37: migrateToV37,
}
```

## 关键迁移

### v11: 设备 ID 格式

```go
func migrateToV11(cfg *Configuration) error {
    // 旧格式：52 字符无校验位
    // 新格式：56 字符带 Luhn 校验位
    for i := range cfg.Devices {
        deviceID, _ := protocol.DeviceIDFromString(cfg.Devices[i].DeviceID)
        cfg.Devices[i].DeviceID = deviceID.String()
    }
    return nil
}
```

### v15: 文件夹类型

```go
func migrateToV15(cfg *Configuration) error {
    // 旧：ReadOnly bool
    // 新：FolderType string
    for i := range cfg.Folders {
        if cfg.Folders[i].ReadOnly {
            cfg.Folders[i].Type = config.FolderTypeSendOnly
        } else {
            cfg.Folders[i].Type = config.FolderTypeSendReceive
        }
    }
    return nil
}
```

### v20: 加密文件夹

```go
func migrateToV20(cfg *Configuration) error {
    // 添加 ReceiveEncrypted 文件夹类型支持
    // 无需数据迁移，仅版本号
    return nil
}
```

### v37: 当前版本

```go
func migrateToV37(cfg *Configuration) error {
    // 最新迁移
    return nil
}
```

## 迁移测试

`lib/config/migration_test.go`：

```go
func TestMigrateFromV10(t *testing.T) {
    cfg := &Configuration{Version: 10, ...}
    err := migrateIfNeeded(cfg)
    require.NoError(t, err)
    assert.Equal(t, CurrentVersion, cfg.Version)
    // 验证迁移结果
}
```

## 迁移安全性

### 原子性

- 迁移在内存中完成
- 保存前验证
- 失败则不保存

### 备份

```go
func Load(path string) (*Configuration, error) {
    // 1. 读取配置
    cfg, err := readConfig(path)
    // 2. 备份原始配置
    if cfg.Version < CurrentVersion {
        backupPath := path + ".v" + strconv.Itoa(cfg.Version) + ".bak"
        copyFile(path, backupPath)
    }
    // 3. 迁移
    migrateIfNeeded(cfg)
    // 4. 保存
    saveConfig(path, cfg)
    return cfg, nil
}
```

## 版本兼容性

### 向前兼容

- 旧版本忽略未知字段
- 新字段使用默认值

### 向后兼容

- 新版本能读取旧配置
- 迁移函数处理差异

## 配置文件格式

### XML

```xml
<configuration version="37">
    ...
</configuration>
```

`version` 属性标识配置版本。

### 序列化

```go
func (cfg *Configuration) Marshal() ([]byte, error) {
    var buf bytes.Buffer
    encoder := xml.NewEncoder(&buf)
    encoder.Indent("", "  ")
    err := encoder.Encode(cfg)
    return buf.Bytes(), err
}
```

## 设计权衡

### 渐进迁移 vs 一次性迁移

**选择**：渐进迁移（每版本一个迁移函数）。
**优势**：可追踪，可回滚。
**劣势**：迁移函数累积。

### 备份 vs 不备份

**选择**：迁移前备份。
**优势**：迁移失败可恢复。
**劣势**：增加磁盘使用。
