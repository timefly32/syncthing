# 服务工具模块

## 概述

`lib/svcutil/` 提供服务管理工具，约 300 行 Go 代码。包括 Supervisor 和 Service 接口。

## 职责

- **Supervisor**：管理一组服务的生命周期
- **Service 接口**：统一的服务接口
- **错误处理**：服务失败的处理

## Supervisor

`lib/svcutil/supervisor.go`：

```go
type Group struct {
    name     string
    services []Service
    cancel   context.CancelFunc
    err      error
    ...
}
```

### Add

```go
func (g *Group) Add(service Service) {
    g.services = append(g.services, service)
}
```

### Start

```go
func (g *Group) Start() error {
    for _, svc := range g.services {
        if err := svc.Start(); err != nil {
            g.Stop()
            return err
        }
    }
    return nil
}
```

### Stop

```go
func (g *Group) Stop() {
    g.cancel()
    for _, svc := range g.services {
        svc.Stop()
    }
}
```

### Wait

```go
func (g *Group) Wait() error {
    for _, svc := range g.services {
        if err := svc.Wait(); err != nil {
            return err
        }
    }
    return nil
}
```

## Service 接口

```go
type Service interface {
    Start() error
    Stop() error
    Wait() error
    String() string
}
```

## ServiceFunc

`ServiceFunc` 将普通函数包装为 Service：

```go
type ServiceFunc struct {
    Func    func(ctx context.Context) error
    cancel  context.CancelFunc
    done    chan error
}

func (s *ServiceFunc) Start() error {
    ctx, cancel := context.WithCancel(context.Background())
    s.cancel = cancel
    s.done = make(chan error, 1)
    go func() {
        s.done <- s.Func(ctx)
    }()
    return nil
}
```

## AsService

`AsService(fn, name)` 创建命名服务：

```go
func AsService(fn func(ctx context.Context) error, name string) Service {
    return &namedService{ServiceFunc{Func: fn}, name}
}
```

## 错误处理

- 任一服务失败，Supervisor 停止所有服务
- `Wait()` 返回第一个错误
- `Stop()` 有序停止

## 使用示例

```go
sup := svcutil.NewGroup("syncthing")
sup.Add(model)
sup.Add(connections)
sup.Add(api)

if err := sup.Start(); err != nil {
    log.Fatal(err)
}

if err := sup.Wait(); err != nil {
    log.Fatal(err)
}
```

## 测试

`supervisor_test.go`：

- 启动/停止测试
- 错误处理测试
- 并发测试

## 设计权衡

### Supervisor vs 独立管理

**选择**：Supervisor 统一管理。
**优势**：有序启动/停止，集中错误处理。
**劣势**：增加抽象层。

### 顺序 vs 并行启动

**选择**：顺序启动。
**优势**：依赖关系明确。
**劣势**：启动时间较长。
