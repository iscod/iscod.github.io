# sync.Pool

`sync.Pool` 是 Go 1.3 引入的**临时对象缓存池**，常用于复用短生命周期对象以降低分配与 GC 压力。它更适合“可丢弃的缓存”，而不是强一致、强容量约束的“连接池”。

本文用 `sync.Pool` 演示一种 **gRPC ClientConn 复用**的实现方式，并给出服务端封装与 Web 端使用示例。

## 示例：用 sync.Pool 复用 gRPC 连接

整体思路：

1. 定义 `Client` 抽象接口（`Get/Put`）。
2. 用 `sync.Pool` 承载连接对象的复用。
3. 在服务侧提供 `GetClient()`，用 `sync.Once` 保证只初始化一次。
4. 调用侧 `Get()` 取连接、`defer Put()` 归还。

```go
// sync.Pool 构建连接复用池
package client

import (
    "fmt"
    "google.golang.org/grpc"
    "google.golang.org/grpc/connectivity"
    "sync"
)

type Client interface {
    Get() *grpc.ClientConn
    Put(*grpc.ClientConn)
}

func NewClient(target string, opts ...grpc.DialOption) Client {
    return &clientPool{
        sync.Pool{New: func() any {
            conn, err := grpc.Dial(target, opts...)
            if err != nil {
                fmt.Println("error: getting client pool", err)
                return nil
            }
            fmt.Printf("new client pool target %s", target)
            return conn
        }},
    }
}

type clientPool struct {
    sync.Pool
}

func (c *clientPool) Get() *grpc.ClientConn {
    clientAny := c.Pool.Get()
    client, _ := clientAny.(*grpc.ClientConn)
    if client == nil || client.GetState() == connectivity.Shutdown || client.GetState() == connectivity.TransientFailure {
        if client != nil {
            fmt.Printf("error: getting client pool %s", client.GetState())
            client.Close()
        }
        // 再 Get 一次：池为空时会触发 Pool.New 创建新连接
        clientAny = c.Pool.Get()
        client, _ = clientAny.(*grpc.ClientConn)
    }
    return client
}

func (c *clientPool) Put(client *grpc.ClientConn) {
    if client == nil || client.GetState() == connectivity.Shutdown || client.GetState() == connectivity.TransientFailure {
        fmt.Printf("error: putting client pool %s", client.GetState())
        client.Close()
        return
    }
    c.Pool.Put(client)
}
```

## 服务侧封装（按 service 维度对外提供）

每个 `service` 可以实现自身的 `client`，为使用方提供一层抽象（隐藏拨号参数、连接复用细节等）。

```go
// service 构建 GetClient，引用上一步创建的 NewClient 方法
var pool client.Client
var once = sync.Once{}

func GetClient() client.Client {
    once.Do(func() { // 采用 once 保证初始化只执行一次
        pool = client.NewClient("127.0.0.1:5000", grpc.WithTransportCredentials(insecure.NewCredentials()))
    })
    return pool
}
```

## 调用侧使用（例如 Web Handler）

```go
// Web 客户端中使用连接
svc := server.GetClient()
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    c := svc.Get()
    defer svc.Put(c) // 使用完 Put 回去
    //s, err := client.GetName(pb.NewUserClient(c)) //new server client coding...
})
```

## 注意事项

1. `sync.Pool` 的对象可能被 GC 清理，因此它更像“缓存”，不保证命中率与容量。
2. gRPC 连接本身通常是长连接并带内部连接管理；生产环境是否需要“连接池”需要结合并发模型、负载均衡与连接数上限综合评估。
