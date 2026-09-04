# 工具函数模块

## 概述

`lib/stringsutil/`、`lib/slicesutil/`、`lib/timeutil/` 提供通用工具函数。

## stringsutil

`lib/stringsutil/`：

### 函数

```go
func TruncateString(s string, max int) string  // 截断字符串
func Contains(haystack []string, needle string) bool  // 切片包含
func Equal(a, b []string) bool  // 切片相等
func UniqueStrings(a []string) []string  // 去重
```

## slicesutil

`lib/slicesutil/`：

### 函数

```go
func Contains[T comparable](s []T, v T) bool  // 泛型包含
func Equal[T comparable](a, b []T) bool  // 泛型相等
func Unique[T comparable](s []T) []T  // 泛型去重
func Difference[T comparable](a, b []T) []T  // 差集
func Intersection[T comparable](a, b []T) []T  // 交集
```

## timeutil

`lib/timeutil/`：

### 函数

```go
func Now() time.Time  // 当前时间（可 mock）
func Since(t time.Time) time.Duration  // 距今时长
func MockNow(t time.Time)  // mock 当前时间（测试用）
```

## 测试

各模块的 `_test.go`：

- 工具函数测试
- 边界情况测试
