# 2. atomic.Bool 与状态开关

本节目标：使用 `atomic.Bool` 表达并发安全的开关状态，例如服务是否关闭、功能是否启用。

---

## 一、基本用法

```go
type Server struct {
    closed atomic.Bool
}

func (s *Server) Close() {
    s.closed.Store(true)
}

func (s *Server) IsClosed() bool {
    return s.closed.Load()
}
```

多个 goroutine 可以安全读取和设置状态。

---

## 二、CompareAndSwap

如果只允许第一次关闭执行清理逻辑：

```go
func (s *Server) Close() {
    if !s.closed.CompareAndSwap(false, true) {
        return
    }

    // do close once
}
```

`CompareAndSwap(false, true)` 表示：

```text
如果当前值是 false，就改成 true 并返回 true；
否则不修改并返回 false。
```

这适合实现简单的一次性状态迁移。

---

## 三、与 sync.Once 的区别

`sync.Once` 表达：

```text
某段函数只执行一次。
```

`atomic.Bool + CompareAndSwap` 表达：

```text
状态从 false 变成 true 的那个 goroutine 执行后续逻辑。
```

如果只是关闭一次资源，二者都可能适用。

如果你还需要读取当前是否关闭，`atomic.Bool` 更直接。

---

## 四、不要构造复杂状态机

如果状态不只是 true/false，而是：

```text
starting
running
stopping
stopped
failed
```

并且状态变化伴随多个字段更新，建议使用 `Mutex` 保护状态机，而不是堆很多 atomic。

---

## 五、本节练习

1. 用 `atomic.Bool` 实现服务关闭标志。
2. 使用 `CompareAndSwap` 保证关闭逻辑只执行一次。
3. 与 `sync.Once` 实现对比。
4. 设计一个多状态服务，判断是否还适合 atomic。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 使用 `atomic.Bool` 读写并发状态。
- 使用 `CompareAndSwap` 实现简单状态迁移。
- 判断复杂状态机是否应该改用锁。
