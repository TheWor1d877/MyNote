在高并发场景下，频繁地 Dial 和 Close TCP 连接会带来巨大的性能开销（三次握手、四次挥手、内核资源分配/回收）。连接池和心跳机制就是为了解决这个问题。

目的：复用已建立的 TCP 连接，避免重复创建/销毁的开销。

## 设计关键点
1. 线程安全： 必须使用sync.Mutex或者chaannel来保证并发安全
2. 声明周期管理: 所有连接都可能会因为网络问题或者对端服务崩溃导致失效，所以连接池需要检测并且剔除怀连接
3. 空闲超时：引入心跳机制，长时间不用的连接应该自动关闭然后释放资源

## 示例
```go
// 一个简化版的连接池结构
type ConnPool struct {
    mu      sync.Mutex
    conns   []net.Conn
    maxSize int
    // ... 其他字段如 dialer, idleTimeout 等
}

func (p *ConnPool) Get() (net.Conn, error) {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    if len(p.conns) > 0 {
        // 从池中取出一个连接
        conn := p.conns[len(p.conns)-1]
        p.conns = p.conns[:len(p.conns)-1]
        return conn, nil
    }
    // 池中无可用连接，且未达到上限，则新建
    if len(p.conns) < p.maxSize {
        return p.dial()
    }
    // 达到上限，可以等待或返回错误
    return nil, errors.New("pool exhausted")
}

func (p *ConnPool) Put(conn net.Conn) {
    p.mu.Lock()
    defer p.mu.Unlock()
    // 在放回前，可以先 Ping 一下检查连接是否还活着
    if p.isAlive(conn) {
        p.conns = append(p.conns, conn)
    } else {
        conn.Close() // 关闭坏连接
    }
}
```