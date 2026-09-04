# Merkle 树模块

## 概述

`lib/merkle/` 实现 Merkle 树数据结构，约 200 行 Go 代码。用于高效验证数据完整性。

## 职责

- **Merkle 树构建**：从数据块构建 Merkle 树
- **根哈希计算**：计算 Merkle 根
- **包含性证明**：验证某块是否在树中

## 数据结构

```go
type Tree struct {
    leaves [][]byte  // 叶子节点哈希
    root   []byte    // 根哈希
}
```

## 构建

```go
func NewTree(blockHashes [][]byte) *Tree {
    t := &Tree{leaves: blockHashes}
    t.computeRoot()
    return t
}
```

### computeRoot

```go
func (t *Tree) computeRoot() {
    if len(t.leaves) == 0 {
        t.root = nil
        return
    }
    level := t.leaves
    for len(level) > 1 {
        var nextLevel [][]byte
        for i := 0; i < len(level); i += 2 {
            if i+1 < len(level) {
                combined := append(level[i], level[i+1]...)
                nextLevel = append(nextLevel, sha256.Sum256(combined))
            } else {
                nextLevel = append(nextLevel, level[i])
            }
        }
        level = nextLevel
    }
    t.root = level[0]
}
```

## 包含性证明

```go
func (t *Tree) Proof(index int) ([][]byte, error) {
    // 返回从叶子到根的路径上的兄弟节点哈希
}
```

## 验证

```go
func VerifyProof(leaf []byte, proof [][]byte, root []byte) bool {
    // 从叶子开始，沿路径计算到根，比较是否匹配
}
```

## 使用场景

- 文件完整性验证
- 增量同步验证
- 数据块去重

## 测试

`merkle_test.go`：

- 树构建测试
- 证明生成/验证测试
- 边界情况测试
