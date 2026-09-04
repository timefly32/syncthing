# Model 测试

## 测试文件概览

`lib/model/` 包含大量测试文件，覆盖核心功能和回归场景：

| 测试文件 | 行数 | 覆盖内容 |
| --- | ---: | --- |
| `model_test.go` | ~2000 | Model 核心功能、连接管理、索引路由 |
| `folder_sendrecv_test.go` | ~1500 | send-receive 文件夹同步流水线 |
| `folder_recvonly_test.go` | ~500 | receive-only 文件夹 |
| `folder_test.go` | ~800 | 文件夹抽象、状态机 |
| `requests_test.go` | ~1000 | 块请求处理 |
| `indexhandler_test.go` | ~400 | 索引处理 |
| `sharedpullerstate_test.go` | ~300 | 每文件状态 |
| `blockpullreorderer_test.go` | ~200 | 块顺序优化 |
| `deviceactivity_test.go` | ~150 | 设备负载均衡 |
| `devicedownloadstate_test.go` | ~200 | 下载状态跟踪 |
| `progressemitter_test.go` | ~200 | 进度发射 |
| `queue_test.go` | ~150 | 拉取队列 |
| `fileinfobatch_test.go` | ~150 | 批量索引 |
| `service_map_test.go` | ~120 | 服务管理 |
| `fakeconns_test.go` | ~400 | 假连接测试辅助 |
| `testutils_test.go` | ~300 | 测试工具 |
| `testos_test.go` | ~100 | 测试 OS 抽象 |

## 测试策略

### 假连接

`fakeconns_test.go` 提供假 `Connection` 实现，用于测试 Model 而不依赖真实网络：

- `fakeConnection`：模拟 BEP 连接
- 可控制 Index/Request/Response 行为
- 支持断开、延迟等场景

### 测试文件系统

`testos_test.go` 提供测试用的文件系统抽象：

- 使用 `fs.FilesystemTypeFake` 内存文件系统
- 避免测试污染真实文件系统
- 支持时间控制

### 集成测试

`test/` 目录的端到端测试启动多个 syncthing 实例：

| 测试 | 场景 |
| --- | --- |
| `test/sync_test.go` | 基本文件同步 |
| `test/conflict_test.go` | 并发修改冲突 |
| `test/reconnect_test.go` | 连接断开重连 |
| `test/override_test.go` | SendOnly 覆盖 |
| `test/scan_test.go` | 扫描行为 |
| `test/symlink_test.go` | 符号链接同步 |
| `test/ignore_test.go` | 忽略规则 |
| `test/manypeers_test.go` | 多设备同步 |
| `test/parallel_scan_test.go` | 并行扫描 |
| `test/delay_scan_test.go` | 延迟扫描 |
| `test/transfer-bench_test.go` | 传输性能基准 |

## 关键测试用例

### 连接管理

- `TestAddConnection`：添加连接、提升主连接
- `TestConnectionClosed`：连接断开后的清理
- `TestMultipleConnections`：多连接优先级处理

### 索引处理

- `TestIndex`：接收全量索引
- `TestIndexUpdate`：接收增量索引
- `TestIndexID`：IndexID 匹配/不匹配场景

### 同步流水线

- `TestPull`：基本拉取
- `TestPullConflict`：冲突处理
- `TestPullWithVersioning`：版本归档
- `TestPullSparseFile`：稀疏文件
- `TestPullEncrypted`：加密文件夹

### 失败场景

- `TestPullFailed`：拉取失败和重试
- `TestPullDeviceUnavailable`：设备不可用
- `TestFolderPaused`：暂停文件夹
- `TestFolderRestart`：文件夹重启

## 测试覆盖评估

### 强覆盖

- 连接管理和优先级
- 索引交换和序列号跟踪
- 基本拉取流水线
- 冲突解决
- 文件夹状态机

### 中等覆盖

- 加密文件夹同步
- 版本归档
- 进度报告
- 设备负载均衡

### 弱覆盖

- 极端并发场景（大量文件、大量设备）
- 网络分区恢复
- 磁盘满/权限错误
- 大文件（>16MiB 块）

## 运行测试

```bash
# 单元测试
go run build.go test ./lib/model/...

# 带竞态检测
go run build.go test -race ./lib/model/...

# 集成测试
go run build.go test ./test/...

# 特定测试
go run build.go test ./lib/model/ -run TestPull
```

## 添加测试

### 新功能测试

1. 在对应的 `_test.go` 文件中添加测试函数
2. 使用 `fakeconns_test.go` 的假连接
3. 使用 `testos_test.go` 的测试文件系统
4. 验证正常路径和错误路径

### 回归测试

1. 复现 bug 的最小场景
2. 在 `test/` 目录添加端到端测试（如涉及多设备）
3. 在 `lib/model/` 添加单元测试（如涉及单组件）
