# 运行时流程

本页追踪 Syncthing 的主要运行时路径，从启动到文件同步的完整链路。

## 启动流程

```mermaid
sequenceDiagram
    participant User
    participant CLI as cmd/syncthing
    participant Monitor as monitor.go
    participant Child as 工作进程
    participant App as lib/syncthing.App
    participant Sup as suture.Supervisor

    User->>CLI: syncthing serve
    CLI->>Monitor: monitorMain()
    Monitor->>Child: exec 子进程
    Child->>App: startup()
    App->>Sup: 创建 mainService

    Note over App: 1. 失败报告处理器
    Note over App: 2. 数据库服务
    Note over App: 3. 事件订阅
    Note over App: 4. 最大化 fd 限制
    Note over App: 5. 计算设备 ID
    Note over App: 6. 检查短 ID 冲突
    Note over App: 7. 清理已删除 folder
    Note over App: 8. 版本迁移
    Note over App: 9. 创建 Model
    Note over App: 10. 配置 TLS
    Note over App: 11. 连接服务 + 发现
    Note over App: 12. 使用报告
    Note over App: 13. GUI/API

    App->>Sup: Serve()
    Sup-->>Child: 运行中

    Note over Monitor: 监控子进程
    Child-->>Monitor: 退出（崩溃/正常）
    alt 崩溃且未超阈值
        Monitor->>Child: 重启
    else 正常退出或超阈值
        Monitor-->>User: 退出
    end
```

启动顺序的关键细节见 `lib/syncthing/syncthing.go:129-330`。第 11 步使用 `lateAddressLister` 解决发现与连接的循环依赖：先创建 `connectionsService`，再将其绑定到 `discoveryManager` 的 `AddressLister`。

## 连接建立流程

```mermaid
sequenceDiagram
    participant Local as 本地设备
    participant ConnSvc as connections.Service
    participant Disco as discover.Manager
    participant Remote as 对端设备
    participant Proto as protocol.Connection
    participant Model as model.Model

    Note over Local,ConnSvc: 拨号路径
    Local->>ConnSvc: connect() 循环
    ConnSvc->>Disco: Lookup(deviceID)
    Disco-->>ConnSvc: addresses[]
    ConnSvc->>Remote: TCP/QUIC/Relay 拨号
    ConnSvc->>Remote: TLS 握手 (NextProtos=bep/1.0)
    ConnSvc->>Remote: ExchangeHello()
    Remote-->>ConnSvc: Hello{DeviceName, ClientVersion}

    Note over ConnSvc: connectionCheckEarly
    ConnSvc->>ConnSvc: 检查忽略/暂停/限制
    ConnSvc->>Model: OnHello(deviceID, addr, hello)
    Model-->>ConnSvc: 接受/拒绝

    ConnSvc->>Proto: NewConnection(...)
    Proto->>Proto: Start() (5 goroutines)
    Proto->>Remote: ClusterConfig (首条消息)
    Remote-->>Proto: ClusterConfig

    Proto->>Model: ClusterConfig(conn, cc)
    Model->>Model: 检查共享文件夹
    Model->>Model: ensureIndexHandler

    Note over Proto,Model: 此后可交换 Index/Request
```

### 拨号策略

拨号循环（`lib/connections/service.go:447`）维护 `nextDialRegistry` 记录每设备+地址的下次拨号时间：

1. **初始退避**：1s → 2s → 4s → ... → `stdConnectionLoopSleep`(60s)
2. **并行拨号**：`dialMaxParallel=64` 信号量，按优先级分桶
3. **冷却机制**：`dialCoolDownInterval=2min`，`dialCoolDownMaxAttempts=3`
4. **队列排序**：最近看到的设备优先，短生命周期设备排后

### 连接接受路径

监听器接受的连接通过 `s.conns` channel 进入 `handleConns`（`service.go:242`）：

1. 检查 BEP 协议协商（警告但不拒绝，兼容 iOS）
2. 验证对端证书数量为 1
3. 计算对端设备 ID
4. 拒绝自连接（NAT hairpinning）
5. `connectionCheckEarly`：拒绝忽略/暂停/超限设备
6. 20 秒超时内异步交换 Hello
7. 生成 connectionID（基于双方时间戳 + 随机数）

## 文件同步流程

### 扫描阶段

```mermaid
sequenceDiagram
    participant Folder as folder.Serve
    participant Scanner as scanner.Walk
    participant FS as 文件系统
    participant DB as 数据库
    participant Model as model.Model

    Folder->>Folder: scanTimer 触发或手动
    Folder->>Scanner: Walk(subs, matcher)
    Scanner->>FS: 遍历目录
    FS-->>Scanner: 文件列表
    Scanner->>DB: CurrentFiler.Get(name)
    DB-->>Scanner: 现有 FileInfo

    alt 文件未变更 (mtime/size/perm 一致)
        Scanner-->>Scanner: 跳过
    else 文件变更
        Scanner->>Scanner: Blocks(file, blockSize)
        Scanner->>Scanner: 并行 SHA-256 哈希
        Scanner-->>Folder: ScanResult{FileInfo}
    end

    Folder->>DB: Update(folder, files)
    Folder->>Model: 触发 IndexUpdate 发送
    Folder->>Folder: 状态 → Idle
```

扫描器（`lib/scanner/walk.go`）的关键行为：
- **增量扫描**：通过 `Subs` 指定子路径，仅扫描变更部分
- **忽略规则**：`Matcher` 应用 `.stignore` 模式
- **并行哈希**：`Hashers` 控制并行度，`sync.Pool` 复用 buffer 和哈希器
- **块大小**：默认 128KB，大块模式按文件大小动态调整（目标 2000 块/文件）

### 索引交换阶段

```mermaid
sequenceDiagram
    participant Local as 本地 Model
    participant IH as indexHandler
    participant DB as 数据库
    participant Proto as protocol.Connection
    participant Remote as 对端

    Note over Local,IH: 发送索引
    Local->>IH: ensureIndexHandler(conn)
    IH->>DB: withHaveSequence(folder, startSeq)
    DB-->>IH: FileInfo 批次
    IH->>Proto: Index(folder, files, lastSeq)
    Proto->>Remote: BEP Index 消息

    Note over Local,IH: 接收索引
    Remote->>Proto: BEP IndexUpdate
    Proto->>Local: IndexUpdate(conn, folder, files)
    Local->>IH: 接收并校验
    IH->>IH: checkIndexConsistency
    IH->>DB: Update(folder, device, files)
    IH->>IH: 更新 localPrevSequence
```

索引处理（`lib/model/indexhandler.go`）的关键不变量：
- `localPrevSequence`：DB 迭代游标，单调递增
- `sentPrevSequence`：已发送给对端的序列号，单调递增
- `IndexID`：标识索引序列，重连后判断是否需要全量重发
- `PrevSequence`（IndexUpdate 独有）：让接收方检测缺失批次

### 拉取阶段（send-receive 文件夹）

这是最复杂的流程，在 `lib/model/folder_sendrecv.go` 中实现：

```mermaid
flowchart TD
    Start["pull() 开始"] --> Need["获取 Need 列表<br/>(DB 查询)"]
    Need --> Sort["按 PullOrder 排序"]
    Sort --> ForEach["遍历每个需要的文件"]

    ForEach --> Check{"文件存在且<br/>blocks 匹配?"}
    Check -->|是| Skip["跳过"]
    Check -->|否| Prepare["准备 sharedPullerState"]

    Prepare --> Copier["Copier goroutine"]
    Prepare --> Puller["Puller goroutine"]
    Prepare --> Finisher["Finisher goroutine"]

    subgraph "Copier"
        C1["查找本地可用块<br/>(origin 或其他文件)"]
        C2["copyDone / copiedFromOrigin"]
        C1 --> C2
    end

    subgraph "Puller"
        P1["选择最佳设备<br/>(deviceActivity)"]
        P2["发送 Request<br/>(block, offset, hash)"]
        P3["接收 Response"]
        P4["pullDone"]
        P1 --> P2 --> P3 --> P4
    end

    Copier --> Ready{"所有块就绪?"}
    Puller --> Ready
    Ready -->|否| Wait["等待"]
    Ready -->|是| Finisher

    subgraph "Finisher"
        F1["finalClose<br/>(SyncClose + fsync)"]
        F2["syncOwnership<br/>(平台特定)"]
        F3["rename 临时文件 → 正式名"]
        F4["versioner.Archive<br/>(如果配置)"]
        F1 --> F2 --> F3 --> F4
    end

    Finisher --> DBUpdate["DB Updater"]
    DBUpdate --> Commit["提交到数据库"]
    Commit --> Next["下一个文件"]
    Next --> ForEach
```

#### sharedPullerState 状态机

每个正在拉取的文件有一个 `sharedPullerState`（`lib/model/sharedpullerstate.go`），跟踪块的就绪状态：

- `copyNeeded`：还需从本地复制的块数
- `pullNeeded`：还需从网络拉取的块数
- **完成条件**：`copyNeeded + pullNeeded == 0`
- `finalClose`：同步并关闭临时文件，写入加密 trailer（如适用），rename 为正式名

#### 块拉取调度

`deviceActivity`（`lib/model/deviceactivity.go`）实现负载均衡：
- 跟踪每设备的在途请求数
- 选择负载最低的设备
- `blockPullReorderer` 优化拉取顺序（顺序/随机/标准）

#### 冲突处理

当本地和远端同时修改同一文件时：
1. `WinsConflict`（`lib/protocol/bep_fileinfo.go:212`）仲裁：
   - 仅一方 invalid → 非 invalid 方胜
   - 修改时间更晚者胜
   - 时间相等 → 版本向量 `ConcurrentGreater` 决定
2. 败方文件被重命名为冲突文件（`name.sync-conflict-YYYYMMDD-HHMMSS-DEVICE.ext`）

## 配置变更流程

```mermaid
sequenceDiagram
    participant API as REST API
    participant Wrapper as config.Wrapper
    participant Sub as 订阅者
    participant Committer as Committer

    API->>Wrapper: Modify(func(cfg))
    Wrapper->>Wrapper: 执行修改函数
    Wrapper->>Wrapper: 验证 (maxModifications=1000)
    Wrapper->>Sub: OnCommit(committer)
    Sub->>Committer: 检查变更
    Sub-->>Wrapper: requiresRestart?
    alt 需要重启
        Sub->>Sub: 标记重启
    else 可热重载
        Sub->>Sub: 应用变更
    end
    Wrapper->>Wrapper: 防抖保存 (minSaveInterval=5s)
```

配置变更通过观察者模式通知所有订阅者（Model、Connections、Discover 等）。`Committer` 接口允许订阅者区分 `requiresRestart`（需重启）和普通变更。

## 事件流

```mermaid
flowchart LR
    Source["事件源<br/>(Model/Scanner/...)"] --> Logger["events.Logger"]
    Logger --> Sub1["默认订阅<br/>(API 长轮询)"]
    Logger --> Sub2["Disk 订阅<br/>(磁盘事件)"]
    Logger --> Sub3["UR 订阅<br/>(使用报告)"]
    Logger --> Sub4["自定义订阅"]
```

事件系统（`lib/events/events.go`）使用位掩码订阅，30+ 事件类型。API 端点使用**长轮询**（非 WebSocket）：先 flush 响应头，再阻塞等待事件（`lib/api/api.go:1356`）。

## 关键定时器

| 组件 | 定时器 | 间隔 | 触发条件 |
| --- | --- | --- | --- |
| Folder 扫描 | `scanTimer` | 配置 `RescanInterval` | 默认 1 小时 |
| Folder 拉取 | `pullPause` | 指数退避 | 失败时翻倍，上限 60x |
| 连接拨号 | `connect` 循环 | `stdConnectionLoopSleep` | 60 秒 |
| BEP Ping | `pingSender` | `PingSendInterval/2` | 45 秒 |
| BEP 超时 | `pingReceiver` | `ReceiveTimeout/2` | 150 秒检查 |
| 全局发现 | `sendAnnouncement` | `defaultReannounceInterval` | 30 分钟 |
| 本地发现 | `sendLocalAnnouncements` | `BroadcastInterval` | 30 秒 |
| STUN 保活 | `stunKeepAlive` | 自适应 | 端口变化时减半 |
| 使用报告 | `Service` | 24 小时 | 首次延迟 10s-1m |
