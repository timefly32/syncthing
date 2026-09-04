# 忽略规则深入

## 模式解析

`lib/ignore/parse.go`：

### Glob 模式

```
*.bak         # 匹配所有 .bak 文件
/foo          # 锚定到根的 foo
foo/          # foo 目录
*.{tmp,bak}   # 扩展：匹配 *.tmp 和 *.bak
**/secret     # 任意层级的 secret
```

### 正则表达式

```
(?i)*.tmp     # 不区分大小写
(?#comment)   # 注释
```

`(?` 前缀标识正则表达式。

### 反向匹配

```
!important.bak    # 取消忽略 important.bak
```

最后匹配的模式决定结果。

## 匹配算法

`lib/ignore/ignore.go` 的 `Match`：

```go
func (m *Matcher) Match(file string) MatchResult {
    for _, pattern := range m.patterns {
        if pattern.Match(file) {
            if pattern.IsNegated() {
                return MatchResult{Ignored: false, MatchedBy: pattern.String()}
            } else {
                return MatchResult{Ignored: true, MatchedBy: pattern.String()}
            }
        }
    }
    return MatchResult{Ignored: false}
}
```

**注意**：实际实现更复杂，处理多个模式的优先级。

## #include 解析

`lib/ignore/ignore.go`：

```go
func (m *Matcher) Load(file string) error {
    content, _ := fs.ReadFile(file)
    for _, line := range strings.Split(content, "\n") {
        line = strings.TrimSpace(line)
        if strings.HasPrefix(line, "#include ") {
            includeFile := strings.TrimPrefix(line, "#include ")
            // 递归加载
            m.Load(includeFile)
            continue
        }
        // 解析模式
        pattern := parsePattern(line)
        m.patterns = append(m.patterns, pattern)
    }
}
```

### 循环引用检测

`Load` 跟踪已加载的文件，防止循环引用：

```go
type Matcher struct {
    ...
    includedFiles map[string]bool  // 已加载的文件
}
```

## 缓存

`lib/ignore/cache.go`：

```go
type cache struct {
    entries map[string]MatchResult
    size    int
    maxSize int
}
```

- LRU 缓存
- 模式变更时清空
- 默认 1000 条

## Pattern 结构

```go
type Pattern struct {
    pattern  string
    regex    *regexp.Regexp  // 正则模式
    glob     glob.Glob       // glob 模式
    negated  bool            // 反向匹配
    anchored bool            // 锚定到根
    dirOnly  bool            // 仅目录
}
```

## Match 方法

```go
func (p Pattern) Match(file string) bool {
    if p.dirOnly && !isDir(file) {
        return false
    }
    if p.regex != nil {
        return p.regex.MatchString(file)
    }
    return p.glob.Match(file)
}
```

## 路径归一化

匹配前路径归一化：

- 转为正斜杠
- 去除前导 `./`
- folder-relative（无前导 `/`）

## 大小写敏感性

- 默认大小写敏感
- `(?i)` 前缀使正则不区分大小写
- glob 模式始终大小写敏感

## 测试用例

`lib/ignore/ignore_test.go`：

```go
func TestMatcher(t *testing.T) {
    m := NewMatcher(...)
    m.Parse([]string{
        "*.bak",
        "!important.bak",
        "/secret",
        "foo/",
    })

    tests := []struct {
        file    string
        ignored bool
    }{
        {"test.bak", true},
        {"important.bak", false},
        {"secret", true},
        {"foo/file.txt", true},
        {"bar/foo", false},  // foo/ 仅匹配目录
    }
    ...
}
```

## 性能考虑

### 模式数量

- 大量模式影响匹配性能
- 缓存缓解重复匹配
- 考虑合并相似模式

### 正则 vs glob

- glob 通常更快
- 正则更灵活但更慢
- 优先使用 glob

## 集成

### Model

`lib/model` 为每个 folder 创建 `Matcher`：

1. 启动时加载 `.stignore`
2. 扫描时调用 `Match` 过滤
3. `.stignore` 变更时重新加载

### API

`/rest/db/ignores` 端点：

- `GET`：读取忽略规则
- `POST`：写入忽略规则

### 事件

`.stignore` 变更触发 `FolderWatchStateChanged` 事件。
