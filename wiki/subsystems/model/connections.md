# Model 连接管理深入

## 连接注册

`lib/model/model.go`：

```go
func (m *Model) AddConnection(conn protocol.Connection) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    deviceID := conn.DeviceID()
    m.conns[deviceID] = append(m.conns[deviceID], conn)
    
    // 通知 folderRunners
    for _, folder := range m.folderRunners {
        folder.DeviceConnected(deviceID)
    }
}
```

## 连接移除

```go
func (m *Model) Closed(err error) {
    // 连接关闭回调
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    deviceID := conn.DeviceID()
    conns := m.conns[deviceID]
    // 移除关闭的连接
    for i, c := range conns {
        if c == conn {
            m.conns[deviceID] = append(conns[:i], conns[i+1:]...)
            break
        }
    }
    
    // 通知 folderRunners
    for _, folder := range m.folderRunners {
        folder.DeviceDisconnected(deviceID)
    }
}
```

## ClusterConfig 处理

```go
func (m *Model) ClusterConfig(deviceID protocol.DeviceID, config protocol.ClusterConfig) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    for _, folder := range config.Folders {
        // 1. 检查是否共享此文件夹
        if !m.hasFolder(folder.ID) {
            // 拒绝
            m.evLogger.Log(events.FolderRejected, ...)
            continue
        }
        
        // 2. 检查设备是否在文件夹配置中
        if !m.folderHasDevice(folder.ID, deviceID) {
            // 拒绝
            continue
        }
        
        // 3. 处理加密密码
        for _, device := range folder.Devices {
            if device.ID == m.id {
                // 设置加密密码
            }
        }
        
        // 4. 触发索引发送
        m.folderRunners[folder.ID].ClusterConfig(deviceID, folder)
    }
}
```

## folderRunner.ClusterConfig

```go
func (f *folderRunner) ClusterConfig(deviceID protocol.DeviceID, config protocol.Folder) {
    // 1. 记录设备的 IndexID 和 maxSequence
    f.deviceInfos[deviceID] = deviceInfo{
        indexID:     config.IndexID,
        maxSequence: config.MaxSequence,
    }
    
    // 2. 触发索引发送
    f.scheduleIndexSend(deviceID)
    
    // 3. 如果对端有更新的索引，触发拉取
    if config.MaxSequence > f.knownSequence[deviceID] {
        f.schedulePull()
    }
}
```

## 多连接支持

Syncthing 支持每设备多个连接：

```go
type Model struct {
    conns map[protocol.DeviceID][]protocol.Connection
    ...
}
```

### 连接选择

```go
func (m *Model) getConnection(deviceID protocol.DeviceID) protocol.Connection {
    conns := m.conns[deviceID]
    if len(conns) == 0 {
        return nil
    }
    // 选择最佳连接（优先级最高）
    best := conns[0]
    for _, conn := range conns[1:] {
        if conn.Priority() < best.Priority() {
            best = conn
        }
    }
    return best
}
```

## 设备暂停

```go
func (m *Model) PauseDevice(deviceID protocol.DeviceID) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    m.devicePaused[deviceID] = true
    
    // 断开所有连接
    for _, conn := range m.conns[deviceID] {
        conn.Close(errors.New("device paused"))
    }
}

func (m *Model) ResumeDevice(deviceID protocol.DeviceID) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    m.devicePaused[deviceID] = false
    // 连接服务会自动重连
}
```

## 文件夹暂停

```go
func (m *Model) PauseFolder(folder string) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    if runner, ok := m.folderRunners[folder]; ok {
        runner.Pause()
    }
}

func (m *Model) ResumeFolder(folder string) {
    m.connMut.Lock()
    defer m.connMut.Unlock()
    
    if runner, ok := m.folderRunners[folder]; ok {
        runner.Resume()
    }
}
```

## 连接健康检查

```go
func (m *Model) checkConnectionHealth(deviceID protocol.DeviceID) {
    conns := m.conns[deviceID]
    for _, conn := range conns {
        if conn.IsClosed() {
            // 清理已关闭的连接
            m.removeConnection(deviceID, conn)
        }
    }
}
```

## 事件通知

连接状态变更触发事件：

```go
m.evLogger.Log(events.DeviceConnected, map[string]interface{}{
    "id":   deviceID.String(),
    "addr": conn.Address(),
    "type": conn.Type(),
})

m.evLogger.Log(events.DeviceDisconnected, map[string]interface{}{
    "id":    deviceID.String(),
    "error": err.Error(),
})
```

## 测试

`lib/model/model_test.go`：

- 连接注册/移除测试
- ClusterConfig 处理测试
- 暂停/恢复测试
- 多连接测试
