# 故障排除

## 常见问题

### 设备无法连接

**症状**：设备显示"已断开"

**排查**：

1. 检查防火墙是否开放端口 22000（TCP/UDP）
2. 检查监听地址配置
3. 查看连接日志：`STTRACE=connections ./syncthing`
4. 检查 NAT 穿透：`curl http://localhost:8384/rest/system/status`
5. 尝试中继连接

### 文件不同步

**症状**：文件未同步到对端

**排查**：

1. 检查文件夹状态：`curl http://localhost:8384/rest/db/status?folder=xxx`
2. 检查忽略规则：`.stignore` 文件
3. 查看扫描日志：`STTRACE=scanner ./syncthing`
4. 查看模型日志：`STTRACE=model ./syncthing`
5. 检查文件权限

### 冲突文件

**症状**：出现 `*.sync-conflict-*.txt` 文件

**原因**：多设备同时修改同一文件

**处理**：

1. 检查冲突文件内容
2. 手动合并或选择保留版本
3. 删除冲突文件

### 数据库损坏

**症状**：启动失败，数据库错误

**排查**：

1. 检查磁盘空间
2. 运行 SQLite 完整性检查：
   ```bash
   sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite "PRAGMA integrity_check;"
   ```
3. 删除数据库并重建：
   ```bash
   rm -rf ~/.local/state/syncthing/index-v0.15.0/<folder>/
   ```
4. 重启 Syncthing（会重新扫描）

### 高 CPU 使用率

**原因**：

1. 初始扫描大文件夹
2. 频繁文件变更
3. 数据库压缩
4. 加密计算

**排查**：

1. 查看扫描进度：`STTRACE=scanner ./syncthing`
2. 检查文件夹大小
3. 调整扫描间隔
4. 减少并行哈希器数量

### 高内存使用

**原因**：

1. 大量文件元数据
2. 大文件块缓存
3. 连接缓冲

**排查**：

1. 检查文件数量
2. 调整块大小
3. 减少连接数

## 调试工具

### STTRACE

启用特定子系统的调试日志：

```bash
STTRACE=model ./syncthing
STTRACE=model,protocol,connections ./syncthing
STTRACE=* ./syncthing
```

### REST API

```bash
# 系统状态
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/system/status

# 连接详情
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/system/connections | jq

# 文件夹完成度
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/db/completion?folder=xxx&device=yyy

# 系统日志
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/system/log
```

### CLI 工具

```bash
# 显示配置路径
syncthing --paths

# 显示设备 ID
syncthing --device-id

# 生成密钥
syncthing --generate
```

### 数据库检查

```bash
# 完整性检查
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite "PRAGMA integrity_check;"

# 表结构
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite ".schema"

# 文件计数
sqlite3 ~/.local/state/syncthing/index-v0.15.0/<folder>/db.sqlite "SELECT count(*) FROM files;"
```

## 性能优化

### 大文件夹

- 增加扫描间隔
- 启用文件系统监视（`fsWatcherEnabled`）
- 调整并行哈希器（`hashers`）
- 使用大块模式（`useLargeBlocks`）

### 低带宽

- 启用压缩（`compression=metadata`）
- 限制带宽（`maxSendKbps`/`maxRecvKbps`）
- 减少并行连接

### 多设备

- 调整 `maxFolderConcurrency`
- 限制连接数
- 使用 SendOnly 文件夹减少冲突

## 日志分析

### 关键日志消息

| 消息 | 含义 |
| --- | --- |
| `Connected to device` | 设备连接成功 |
| `Disconnected from device` | 设备断开 |
| `Folder X completed` | 文件夹同步完成 |
| `Puller (folder X): ...` | 拉取错误 |
| `Scanner: ...` | 扫描错误 |
| `Database error` | 数据库错误 |

### 日志级别

- INFO：正常操作
- WARN：潜在问题
- ERROR：需要处理
- DEBUG：调试信息

## 获取帮助

- [文档](https://docs.syncthing.net/)
- [论坛](https://forum.syncthing.net/)
- [GitHub Issues](https://github.com/syncthing/syncthing/issues)

报告问题时提供：

1. Syncthing 版本
2. 操作系统
3. 配置（脱敏）
4. 相关日志
5. 复现步骤
