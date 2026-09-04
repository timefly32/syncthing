# 版本控制深入

## 简单版本控制详细

`lib/versioner/simple.go`：

### Archive 流程

```go
func (v *Simple) Archive(filePath string) error {
    // 1. 检查文件是否存在
    if _, err := v.fs.Lstat(filePath); err != nil {
        return err
    }

    // 2. 构造归档路径
    archivePath := v.archivePath(filePath)

    // 3. 创建归档目录
    v.fs.MkdirAll(filepath.Dir(archivePath), 0755)

    // 4. 移动文件
    if err := v.fs.Rename(filePath, archivePath); err != nil {
        // 跨设备，退化为复制+删除
        return copyAndDelete(v.fs, filePath, archivePath)
    }

    // 5. 清理旧版本
    if v.keep > 0 {
        v.cleanupOldVersions(filePath)
    }

    return nil
}
```

### 归档路径

```
.stversions/<name>.~<timestamp>~
```

- `.stversions/` 是归档目录
- `<timestamp>` 是 Unix 时间戳
- `~` 分隔符

### 清理旧版本

```go
func (v *Simple) cleanupOldVersions(filePath string) {
    // 1. 列出所有版本
    versions := v.getVersions(filePath)
    // 2. 按时间排序
    sort.Sort(byTime(versions))
    // 3. 删除超出 keep 的旧版本
    for i := v.keep; i < len(versions); i++ {
        v.fs.Remove(versions[i].Path)
    }
}
```

## 错开版本控制详细

`lib/versioner/staggered.go`：

### 保留策略

```go
var intervals = []struct {
    age      time.Duration
    interval time.Duration
}{
    {30 * 24 * 3600 * time.Second, 7 * 24 * 3600 * time.Second},  // > 30 天，每周
    {7 * 24 * 3600 * time.Second, 24 * 3600 * time.Second},       // > 7 天，每天
    {24 * 3600 * time.Second, 6 * 3600 * time.Second},            // > 1 天，每 6 小时
    {3600 * time.Second, 3600 * time.Second},                     // > 1 小时，每小时
    {0, 30 * 60 * time.Second},                                   // < 1 小时，每 30 分钟
}
```

### 清理算法

```go
func (v *Staggered) cleanup(filePath string) {
    versions := v.getVersions(filePath)
    // 按时间倒序排序（最新在前）
    sort.Sort(sort.Reverse(byTime(versions)))

    var keep []FileVersion
    for _, version := range versions {
        age := time.Since(version.Time)
        interval := intervalFor(age)
        if len(keep) == 0 || version.Time.Before(keep[len(keep)-1].Time.Add(-interval)) {
            keep = append(keep, version)
        }
    }

    // 删除未保留的版本
    for _, version := range versions {
        if !contains(keep, version) {
            v.fs.Remove(version.Path)
        }
    }
}
```

### 后台清理

```go
func (v *Staggered) Start() {
    go func() {
        ticker := time.NewTicker(v.cleanInterval)
        for {
            select {
            case <-ticker.C:
                v.cleanAll()
            case <-v.ctx.Done():
                return
            }
        }
    }()
}
```

## 垃圾回收版本控制

`lib/versioner/garbage_collector.go`：

### 清理策略

```go
func (v *GarbageCollector) Start() {
    go func() {
        ticker := time.NewTicker(v.gcInterval)
        for {
            select {
            case <-ticker.C:
                v.collect()
            case <-v.ctx.Done():
                return
            }
        }
    }()
}

func (v *GarbageCollector) collect() {
    // 遍历 .stversions/
    v.fs.Walk(".stversions", func(path string, info fs.FileInfo, err error) error {
        if time.Since(info.ModTime()) > v.maxAge {
            v.fs.Remove(path)
        }
        return nil
    })
}
```

## 外部版本控制

`lib/versioner/external.go`：

### Archive

```go
func (v *External) Archive(filePath string) error {
    cmd := exec.Command(v.command, "archive", filePath)
    cmd.Dir = v.folderPath
    return cmd.Run()
}
```

### GetVersions

```go
func (v *External) GetVersions(filePath string) ([]FileVersion, error) {
    cmd := exec.Command(v.command, "get-versions", filePath)
    output, _ := cmd.Output()
    // 解析 JSON 输出
    var versions []FileVersion
    json.Unmarshal(output, &versions)
    return versions, nil
}
```

### Restore

```go
func (v *External) Restore(filePath string, version time.Time) error {
    cmd := exec.Command(v.command, "restore", filePath, version.Format(time.RFC3339))
    cmd.Dir = v.folderPath
    return cmd.Run()
}
```

## 版本文件命名

所有版本控制使用相同的命名约定：

```
.stversions/<relative_path>.~<timestamp>~
```

示例：

```
.stversions/docs/readme.md.~1700000000~
.stversions/docs/readme.md.~1700100000~
.stversions/docs/readme.md.~1700200000~
```

## 集成

### Model

`lib/model/folder_sendrecv.go`：

```go
func (f *sendReceiveFolder) archiveBeforeReplace(name string) error {
    if f.versioner != nil {
        return f.versioner.Archive(name)
    }
    return nil
}
```

### API

`/rest/folder/versions`：

```http
GET /rest/folder/versions?folder=xxx&file=docs/readme.md
```

响应：

```json
[
    {"version": "2024-01-01T00:00:00Z", "size": 1024, "modified": "2024-01-01T00:00:00Z"},
    {"version": "2024-01-02T00:00:00Z", "size": 2048, "modified": "2024-01-02T00:00:00Z"}
]
```

## 测试

`lib/versioner/simple_test.go`：

- 归档测试
- 清理测试
- 跨设备测试

`lib/versioner/staggered_test.go`：

- 保留策略测试
- 清理算法测试

`lib/versioner/garbage_collector_test.go`：

- 垃圾回收测试

`lib/versioner/external_test.go`：

- 外部命令测试
