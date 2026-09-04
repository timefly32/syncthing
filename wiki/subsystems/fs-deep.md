# 文件系统抽象深入

## Filesystem 接口完整

`lib/fs/fs.go`：

```go
type Filesystem interface {
    Chmod(name string, mode FileMode) error
    Chtimes(name string, atime, mtime time.Time) error
    Create(name string) (File, error)
    CreateSymlink(target, name string) error
    DirNames(dir string) ([]string, error)
    Lstat(name string) (FileInfo, error)
    Mkdir(name string, perm FileMode) error
    MkdirAll(name string, perm FileMode) error
    Open(name string) (File, error)
    OpenFile(name string, flags int, mode FileMode) (File, error)
    ReadSymlink(name string) (string, error)
    Remove(name string) error
    RemoveAll(name string) error
    Rename(oldname, newname string) error
    Stat(name string) (FileInfo, error)
    Symlink(target, name string) error
    Walk(name string, walkFn WalkFunc) error
    URI() string
    Type() FilesystemType
    SameFile(name1, name2 string) bool
}
```

## File 接口

```go
type File interface {
    io.Closer
    io.Reader
    io.ReaderAt
    io.Writer
    io.WriterAt
    io.Seeker
    Name() string
    Sync() error
    Truncate(size int64) error
    Stat() (FileInfo, error)
}
```

## FileInfo 接口

```go
type FileInfo interface {
    Name() string
    Mode() FileMode
    Size() int64
    ModTime() time.Time
    IsDir() bool
    Sys() interface{}
    IsRegular() bool
    IsSymlink() bool
}
```

## LocalFilesystem 实现

`lib/fs/basicfs/basicfs.go`：

### 路径处理

```go
func (f *BasicFilesystem) resolved(name string) string {
    // 1. 清理路径
    // 2. 拼接 root
    // 3. 解析符号链接
    return resolved
}
```

### Walk 实现

```go
func (f *BasicFilesystem) Walk(root string, walkFn WalkFunc) error {
    info, err := f.Lstat(root)
    if err != nil {
        return walkFn(root, nil, err)
    }
    return walk(f, root, info, walkFn)
}
```

## CaseFilesystem

`lib/fs/basicfs/case_fs.go`：

### 大小写解析

```go
func (f *CaseFilesystem) realCase(name string) string {
    // 1. 检查缓存
    if cached, ok := f.cache[name]; ok {
        return cached
    }
    // 2. 遍历目录，查找匹配的真实大小写
    real := f.findRealCase(name)
    // 3. 缓存结果
    f.cache[name] = real
    return real
}
```

### 缓存失效

- 文件变更时清空缓存
- 定期清空避免内存增长

## FakeFilesystem

`lib/fs/fakefs/fakefs.go`：

### 内存文件系统

```go
type FakeFilesystem struct {
    mu    sync.Mutex
    files map[string]*fakeFile
}

type fakeFile struct {
    name    string
    content []byte
    mode    FileMode
    modTime time.Time
    isDir   bool
    symlink string
}
```

### 模拟错误

```go
func (f *FakeFilesystem) SetError(name string, err error) {
    f.errors[name] = err
}
```

用于测试错误处理。

## 扩展属性（xattr）

`lib/fs/basicfs/xattr.go`：

### Linux

```go
import "github.com/pkg/xattr"

func (f *BasicFilesystem) Getxattr(name, attr string) ([]byte, error) {
    return xattr.Get(f.resolved(name), attr)
}
```

- 命名空间：`user.*`
- 需要 `CAP_SYS_ADMIN` 或文件所有者

### macOS

```go
import "github.com/pkg/xattr"

func (f *BasicFilesystem) Getxattr(name, attr string) ([]byte, error) {
    return xattr.Get(f.resolved(name), attr)
}
```

- 任意命名空间
- `com.apple.*` 系统属性

### Windows

- 不支持
- 返回 `ErrXattrNotSupported`

## 文件监视

`lib/fs/basicfs/watch.go`：

### Watch 实现

```go
func (f *BasicFilesystem) Watch(path string, ignoreFn func(name string) bool, ctx context.Context) (chan Event, error) {
    // 1. 创建 fsnotify watcher
    watcher, _ := fsnotify.NewWatcher()
    // 2. 添加路径
    watcher.Add(f.resolved(path))
    // 3. 转发事件
    ch := make(chan Event)
    go func() {
        for {
            select {
            case event := <-watcher.Events:
                if ignoreFn(event.Name) {
                    continue
                }
                ch <- Event{event.Name, event.Op}
            case <-ctx.Done():
                watcher.Close()
                close(ch)
                return
            }
        }
    }()
    return ch, nil
}
```

### 递归监视

```go
func (f *BasicFilesystem) WatchRecursive(path string, ...) {
    // 1. 遍历目录树
    // 2. 添加所有子目录
    // 3. 新建目录时自动添加
}
```

## 平台差异

### Unix

```go
// 权限位
mode := fi.Mode().Perm()

// 符号链接
target, _ := os.Readlink(name)

// xattr
xattr.Get(name, attr)
```

### Windows

```go
// 权限位（简化）
if fi.Mode()&0200 == 0 {
    // 只读
}

// 符号链接（需要管理员）
os.Symlink(target, name)

// 无 xattr
```

### macOS

```go
// 权限位
mode := fi.Mode().Perm()

// 符号链接
target, _ := os.Readlink(name)

// xattr
xattr.Get(name, attr)

// 大小写不敏感（默认）
// 需要 CaseFilesystem
```

## 测试

`lib/fs/basicfs/basicfs_test.go`：

- 接口方法测试
- 路径处理测试
- 权限测试

`lib/fs/fakefs/fakefs_test.go`：

- 内存文件系统测试
- 错误模拟测试

`lib/fs/basicfs/case_fs_test.go`：

- 大小写不敏感测试
- 缓存测试

`lib/fs/basicfs/xattr_test.go`：

- xattr 读写测试
- 平台差异测试
