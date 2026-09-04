# 数据库维护者笔记

## 安全编辑点

### 添加新的 Schema 迁移

1. 在 `internal/db/sqlite/schema/folder/` 添加 `NN-description.sql`
2. 递增 `currentSchemaVersion`
3. 在 `db_service.go` 的迁移逻辑中注册
4. 添加迁移测试

### 添加新的 KV 键

1. 在 `lib/locations/` 或使用方定义键名
2. 使用 `db.Typed` 进行类型安全访问
3. 考虑迁移（如从旧格式转换）

### 修改文件表查询

1. 在 `folderdb_*.go` 中修改查询
2. **注意**：名称和版本规范化（v5）需要 JOIN `file_names`/`file_versions`
3. 更新预处理语句缓存

## 风险区域

### 高风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `folderdb_update.go` | 数据损坏 | 事务处理错误可能导致数据丢失 |
| `db_open.go` | 连接泄漏 | 连接池管理错误 |
| `folderdb_open.go` | 数据库损坏 | WAL/checkpoint 错误 |
| Schema 迁移 | 数据丢失 | 迁移错误可能丢失数据 |

### 中风险

| 文件 | 风险 | 原因 |
| --- | --- | --- |
| `db_prepared.go` | 语句泄漏 | 预处理语句未正确关闭 |
| `db_service.go` | GC 错误 | 维护逻辑可能误删数据 |

## 常见变更配方

### 添加新的文件表列

```sql
-- 1. 创建迁移文件 internal/db/sqlite/schema/folder/07-add-column.sql
ALTER TABLE files ADD COLUMN new_column INTEGER NOT NULL DEFAULT 0;

-- 2. 递增 currentSchemaVersion
-- 3. 更新 folderdb_*.go 的查询
-- 4. 更新 FileInfo 序列化/反序列化
```

### 添加新的计数类型

```sql
-- 修改 counts 表的触发器
-- 注意：触发器变更需要重建（SQLite 不支持 ALTER TRIGGER）
```

## 性能考虑

### 写入性能

- 触发器维护 counts 增加写入开销
- WAL 模式允许读写并发
- 预处理语句缓存减少预处理开销
- 批量更新比单条更新高效

### 查询性能

- `sequence` 主键支持高效范围查询
- `WITHOUT ROWID` 减少存储和提高查询性能
- 名称/版本规范化增加 JOIN，但减少存储
- 考虑添加索引如需按其他字段查询

### 连接池

- `maxOpenConns=8`，`maxIdleConns=4`
- 每个 folder DB 独立连接池
- cgo 和非 cgo 构建模式

## 调试

### 数据库统计

```bash
curl http://localhost:8384/rest/db/status?folder=xxx
```

### Prometheus 指标

```
http://localhost:8384/rest/metrics
```

关注：
- `syncthing_db_total_queries`
- `syncthing_db_query_duration_seconds`

### SQLite PRAGMA

可通过 `sqlite3` 命令行工具检查数据库：

```bash
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite "PRAGMA integrity_check;"
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite "PRAGMA journal_mode;"
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite ".schema"
```

## 代码审查清单

修改 `internal/db/` 时检查：

- [ ] 事务是否正确提交/回滚？
- [ ] 预处理语句是否正确关闭？
- [ ] 连接是否正确归还连接池？
- [ ] Schema 迁移是否可逆或向前兼容？
- [ ] 触发器是否需要更新？
- [ ] 计数表是否一致？
- [ ] WAL checkpoint 是否正确触发？
- [ ] 是否添加了测试？
