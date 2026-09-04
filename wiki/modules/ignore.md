# 忽略规则模块

## 概述

`lib/ignore/` 实现 Syncthing 的文件忽略规则，约 1500 行 Go 代码。支持 `.stignore` 文件、glob 模式、include 指令和正则表达式。

## 职责

- **模式匹配**：glob 和正则表达式
- **include 指令**：`#include` 引用其他忽略文件
- **缓存**：匹配结果缓存
- **反向匹配**：`!` 前缀取消忽略

## .stignore 格式

```
# 注释
#include other-ignore-file
(?i)*.tmp          # 正则表达式（不区分大小写）
*.bak              # glob
!important.bak     # 反向匹配
/secret            # folder 根的 secret 目录
foo/               # foo 目录
*.{tmp,bak,log}    # glob 扩展
```

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `ignore.go` | ~800 | `Matcher`、模式解析、匹配 |
| `cache.go` | ~200 | 匹配结果缓存 |
| `parse.go` | ~300 | 模式解析 |

## Matcher

```go
type Matcher struct {
    fs        fs.Filesystem
    patterns  []Pattern
    cache     *cache
    lines     []string
    mut       sync.RWMutex
    ...
}
```

### Load

```go
func (m *Matcher) Load(file string) error
```

1. 读取 `.stignore` 文件
2. 解析每行模式
3. 处理 `#include` 指令（递归）
4. 编译模式

### Match

```go
func (m *Matcher) Match(file string) (result MatchResult)
```

```go
type MatchResult struct {
    Ignored    bool
    MatchedBy  string  // 匹配的模式
}
```

## 模式类型

### Glob

- `*` 匹配非 `/` 字符
- `**` 匹配任意字符（包括 `/`）
- `?` 匹配单个非 `/` 字符
- `[abc]` 字符类
- `{a,b,c}` 扩展

### 正则表达式

- `(?pattern)` 前缀
- `(?i)` 不区分大小写

### 反向匹配

- `!` 前缀取消忽略
- 最后匹配的模式决定结果

### 路径锚定

- `/foo` 锚定到 folder 根
- `foo` 匹配任意层级
- `foo/` 匹配目录

## #include 指令

```
#include other-ignore-file
#include /absolute/path
#include relative/path
```

- 递归加载引用的文件
- 防止循环引用
- 路径相对于 `.stignore` 文件

## 缓存

`cache.go` 缓存匹配结果：

- LRU 缓存
- 模式变更时清空
- 提高重复匹配性能

## 集成

1. `lib/model` 为每个 folder 创建 `Matcher`
2. 扫描时调用 `Match` 过滤文件
3. `.stignore` 变更时重新加载
4. 通过 API 可读写忽略规则

## 测试覆盖

`ignore_test.go`（~1000 行）：

- 各种模式类型测试
- include 指令测试
- 反向匹配测试
- 边界情况测试

## 设计权衡

### Glob vs 正则

**选择**：两者都支持。
**优势**：glob 简单易用，正则灵活。
**劣势**：增加复杂度。

### 缓存 vs 实时匹配

**选择**：缓存匹配结果。
**优势**：提高重复匹配性能。
**劣势**：内存使用，模式变更需清空。

### 反向匹配

**选择**：`!` 前缀。
**优势**：灵活的例外规则。
**劣势**：顺序依赖（最后匹配决定结果）。
