# 配置参考

## 配置文件

- 路径：`~/.local/state/syncthing/config.xml`（Linux）
- 格式：XML
- 可通过 `--home` 或 `STHOMEDIR` 覆盖

## 顶层结构

```xml
<configuration version="37">
    <gui>...</gui>
    <ldap>...</ldap>
    <options>...</options>
    <folder>...</folder>
    <device>...</device>
    <ignoredDevice>...</ignoredDevice>
    <pendingDevice>...</pendingDevice>
    <ignoredFolder>...</ignoredFolder>
    <pendingFolder>...</pendingFolder>
</configuration>
```

## GUI 配置

```xml
<gui>
    <address>127.0.0.1:8384</address>
    <unixSocket></unixSocket>
    <user>admin</user>
    <password>$2a$10$...</password>
    <apiKey>abc123...</apiKey>
    <theme>default</theme>
    <enabled>true</enabled>
    <insecureAdminAccess>false</insecureAdminAccess>
    <debugging>false</debugging>
</gui>
```

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `address` | `127.0.0.1:8384` | 监听地址 |
| `unixSocket` | (空) | Unix socket 路径 |
| `user` | (空) | 用户名 |
| `password` | (空) | bcrypt 哈希 |
| `apiKey` | (随机) | API 密钥 |
| `theme` | `default` | 主题 |
| `enabled` | `true` | 启用 Web UI |
| `insecureAdminAccess` | `false` | 跳过认证 |

## Options 配置

```xml
<options>
    <listenAddress>tcp://0.0.0.0:22000,quic://0.0.0.0:22000</listenAddress>
    <globalAnnounceEnabled>true</globalAnnounceEnabled>
    <localAnnounceEnabled>true</localAnnounceEnabled>
    <localAnnouncePort>21027</localAnnouncePort>
    <relayEnabled>true</relayEnabled>
    <maxSendKbps>0</maxSendKbps>
    <maxRecvKbps>0</maxRecvKbps>
    <reconnectIntervalS>60</reconnectIntervalS>
    <relaysEnabled>true</relaysEnabled>
    <urAccepted>0</urAccepted>
    <restartOnWakeup>true</restartOnWakeup>
    <autoUpgradeIntervalH>12</autoUpgradeIntervalH>
    <stunKeepaliveS>0</stunKeepaliveS>
    <limitBandwidthInLan>false</limitBandwidthInLan>
    <cacheIgnoredFiles>false</cacheIgnoredFiles>
    <keepTemporariesH>24</keepTemporariesH>
    <natEnabled>true</natEnabled>
    <maxFolderConcurrency>1</maxFolderConcurrency>
    <blockPullInitialOrder>standard</blockPullInitialOrder>
    <useLargeBlocks>false</useLargeBlocks>
</options>
```

## Folder 配置

```xml
<folder id="abc123" path="/path/to/folder" type="sendreceive">
    <device id="DEVICE_ID" introducedBy=""></device>
    <rescanIntervalS>3600</rescanIntervalS>
    <fsWatcherEnabled>true</fsWatcherEnabled>
    <fsWatcherDelayS>10</fsWatcherDelayS>
    <ignorePerms>false</ignorePerms>
    <autoNormalize>true</autoNormalize>
    <minDiskFree>1%</minDiskFree>
    <blockSize>0</blockSize>
    <copiers>0</copiers>
    <pullers>0</pullers>
    <hashers>0</hashers>
    <order>random</order>
    <versioning type="simple">
        <param key="keep" val="5"></param>
    </versioning>
    <paused>false</paused>
    <type>sendreceive</type>
    <scanProgressIntervalS>0</scanProgressIntervalS>
    <maxReceivePercentage>0</maxReceivePercentage>
    <disableTempIndexes>false</disableTempIndexes>
    <sendXattrs>false</sendXattrs>
</folder>
```

### Folder 类型

| 类型 | 说明 |
| --- | --- |
| `sendreceive` | 发送和接收（默认） |
| `sendonly` | 仅发送（主设备） |
| `receiveonly` | 仅接收 |
| `receiveencrypted` | 接收加密（不可信设备） |

### 排序策略

| 值 | 说明 |
| --- | --- |
| `random` | 随机（默认） |
| `alphabetic` | 字母顺序 |
| `smallestFirst` | 最小文件优先 |
| `largestFirst` | 最大文件优先 |
| `oldestFirst` | 最旧文件优先 |
| `newestFirst` | 最新文件优先 |

## Device 配置

```xml
<device id="DEVICE_ID" name="Device Name" compression="metadata">
    <address>dynamic</address>
    <address>tcp://1.2.3.4:22000</address>
    <paused>false</paused>
    <autoAcceptFolders>false</autoAcceptFolders>
    <maxSendKbps>0</maxSendKbps>
    <maxRecvKbps>0</maxRecvKbps>
    <ignoredFolders></ignoredFolders>
    <pendingFolders></pendingFolders>
    <introducedBy></introducedBy>
</device>
```

### 压缩策略

| 值 | 说明 |
| --- | --- |
| `never` | 不压缩 |
| `metadata` | 仅压缩元数据（默认） |
| `always` | 总是压缩 |

## 环境变量

| 变量 | 说明 |
| --- | --- |
| `STHOMEDIR` | 主目录 |
| `STTRACE` | 调试子系统 |
| `STRESTART` | 重启行为 |
| `STNOUPGRADE` | 禁用自动升级 |
| `STGUIASSETS` | Web UI 资源路径 |
| `GOMAXPROCS` | 最大 CPU 数 |

## 命令行参数

```bash
syncthing [flags]
```

| 参数 | 说明 |
| --- | --- |
| `--home` | 主目录 |
| `--gui-address` | GUI 地址 |
| `--gui-apikey` | API 密钥 |
| `--no-browser` | 不打开浏览器 |
| `--no-restart` | 禁用重启 |
| `--no-upgrade` | 禁用升级 |
| `--paths` | 显示路径 |
| `--version` | 版本 |
| `--device-id` | 显示设备 ID |
| `--generate` | 生成密钥并退出 |
