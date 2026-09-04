# 发布与运维

## 发布流程

### 版本号

Syncthing 使用语义版本号：`vMAJOR.MINOR.PATCH`

- `MAJOR`：重大变更
- `MINOR`：新功能
- `PATCH`：错误修复

### 发布步骤

1. 更新 `VERSION` 文件
2. 更新 `CHANGELOG.md`
3. 创建 git tag：`git tag v1.27.0`
4. 推送 tag：`git push origin v1.27.0`
5. CI 自动构建发布二进制
6. 创建 GitHub Release
7. 公告

### 构建矩阵

发布版本覆盖：

| OS | 架构 |
| --- | --- |
| Linux | amd64, arm64, arm-5, arm-6, arm-7 |
| macOS | amd64, arm64 (universal) |
| Windows | amd64, arm64 |
| FreeBSD | amd64, arm64 |
| OpenBSD | amd64 |
| NetBSD | amd64 |
| Dragonfly | amd64 |
| Solaris | amd64 |

### 校验和

每个发布包含 `sha256sum.txt`：

```
1234...  syncthing-linux-amd64-v1.27.0.tar.gz
5678...  syncthing-linux-arm64-v1.27.0.tar.gz
...
```

## 自动升级

- 默认启用（`autoUpgradeIntervalH=12`）
- 检查 GitHub Releases
- 下载并替换二进制
- 重启服务
- 可通过 `STNOUPGRADE=1` 或 `--no-upgrade` 禁用

## 部署

### 系统服务

#### systemd

`/etc/systemd/system/syncthing.service`：

```ini
[Unit]
Description=Syncthing
After=network.target

[Service]
User=syncthing
ExecStart=/usr/local/bin/syncthing -no-browser -no-restart -logflags=3
Restart=on-failure
SuccessExitStatus=3 4 5 6

[Install]
WantedBy=multi-user.target
```

用户级服务：

```bash
systemctl --user enable syncthing.service
systemctl --user start syncthing.service
```

#### launchd (macOS)

`~/Library/LaunchAgents/syncthing.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>syncthing</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/syncthing</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### Docker

```bash
docker run -d \
    --name syncthing \
    -p 8384:8384 \
    -p 22000:22000/tcp \
    -p 22000:22000/udp \
    -p 21027:21027/udp \
    -v syncthing-config:/var/syncthing \
    syncthing/syncthing
```

## 监控

### Prometheus 指标

端点：`http://localhost:8384/rest/metrics`

关键指标：

| 指标 | 说明 |
| --- | --- |
| `syncthing_bytes_sent` | 发送字节数 |
| `syncthing_bytes_received` | 接收字节数 |
| `syncthing_folder_seen` | 文件夹最后看到时间 |
| `syncthing_db_total_queries` | 数据库查询总数 |
| `syncthing_db_query_duration_seconds` | 数据库查询耗时 |

### REST API

```bash
# 系统状态
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/system/status

# 连接
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/system/connections

# 文件夹状态
curl -H "X-API-Key: $API_KEY" http://localhost:8384/rest/db/status?folder=xxx
```

## 日志

### 日志级别

- INFO：正常操作
- WARN：潜在问题
- ERROR：错误
- DEBUG：调试（通过 `STTRACE` 启用）

### 日志位置

- 默认：stdout
- systemd：`journalctl -u syncthing`
- 文件：配置 `--logfile`

### 调试日志

```bash
# 启用特定子系统
STTRACE=model,protocol ./syncthing

# 查看可用子系统
STTRACE=help ./syncthing
```

## 备份

### 配置备份

```bash
# 备份配置和密钥
cp -r ~/.local/state/syncthing/ /backup/syncthing-$(date +%Y%m%d)/
```

### 数据库备份

```bash
# 停止 Syncthing
systemctl stop syncthing

# 备份数据库
cp -r ~/.local/state/syncthing/index-v0.15.0/ /backup/

# 启动 Syncthing
systemctl start syncthing
```

## 安全

### 防火墙

| 端口 | 协议 | 用途 |
| --- | --- | --- |
| 8384 | TCP | Web UI |
| 22000 | TCP | 同步 |
| 22000 | UDP | QUIC |
| 21027 | UDP | 本地发现 |

### TLS

- 自签名证书
- 设备 ID = 证书 SHA-256
- 通过带外渠道交换设备 ID

### Web UI 认证

- 启用用户名/密码
- 使用 HTTPS（配置 TLS 证书）
- 限制监听地址
