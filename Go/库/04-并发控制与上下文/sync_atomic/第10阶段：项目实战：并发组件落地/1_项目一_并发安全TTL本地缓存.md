# 1. 项目一：并发安全 TTL 本地缓存

本项目目标：实现一个支持 TTL、后台清理、并发读写的本地缓存。

这个项目会综合使用：

- `sync.RWMutex`
- goroutine
- `time.Ticker`
- `context`
- `sync.WaitGroup`

---

## 一、需求分析

缓存需要支持：

```text
Set(key, value, ttl)
Get(key) (value, ok)
Delete(key)
Len()
Close()
后台定时清理过期 key
```

要求：

- 多 goroutine 并发调用安全。
- `Get` 不能返回已过期数据。
- `Close` 后后台清理 goroutine 能退出。
- 测试必须通过 `go test -race`。

---

## 二、数据结构设计

```go
type Item struct {
    Value     string
    ExpiresAt time.Time
}

type Cache struct {
    mu     sync.RWMutex
    items  map[string]Item
    stop   chan struct{}
    wg     sync.WaitGroup
    closed bool
}
```

字段说明：

- `items` 是共享状态。
- `mu` 保护 `items` 和 `closed`。
- `stop` 用于通知后台清理退出。
- `wg` 用于等待后台 goroutine 退出。

---

## 三、Set

写操作使用写锁：

```go
func (c *Cache) Set(key, value string, ttl time.Duration) bool {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.closed {
        return false
    }

    c.items[key] = Item{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
    return true
}
```

这里返回 `false` 表示缓存已关闭。

---

## 四、Get

`Get` 有一个微妙点：如果发现过期，需要删除。

简单正确版本可以直接使用写锁：

```go
func (c *Cache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    item, ok := c.items[key]
    if !ok {
        return "", false
    }

    if time.Now().After(item.ExpiresAt) {
        delete(c.items, key)
        return "", false
    }

    return item.Value, true
}
```

虽然读性能不如 `RLock`，但逻辑清楚。等 benchmark 证明有必要，再优化成读锁加二次检查。

---

## 五、后台清理

```go
func (c *Cache) startCleaner(interval time.Duration) {
    c.wg.Add(1)
    go func() {
        defer c.wg.Done()

        ticker := time.NewTicker(interval)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                c.DeleteExpired()
            case <-c.stop:
                return
            }
        }
    }()
}
```

后台 goroutine 必须有退出路径，否则测试和服务关闭都会泄漏 goroutine。

---

## 六、Close

关闭要考虑重复调用。

可以用 `Mutex` 保护 `closed`：

```go
func (c *Cache) Close() {
    c.mu.Lock()
    if c.closed {
        c.mu.Unlock()
        return
    }
    c.closed = true
    close(c.stop)
    c.mu.Unlock()

    c.wg.Wait()
}
```

注意：不要在持锁期间 `wg.Wait()`，否则后台清理如果也需要锁，可能死锁。

---

## 七、测试清单

必须测试：

- `Set` 后能 `Get`。
- TTL 过期后 `Get` 返回 false。
- `Delete` 后不可读。
- 并发 `Set/Get/Delete` 无 race。
- `Close` 后后台 goroutine 退出。
- 重复 `Close` 不 panic。

运行：

```bash
go test -race ./...
```

---

## 八、进阶优化

完成简单正确版本后，再考虑：

- `Get` 使用 `RLock` 快路径。
- 分片锁降低竞争。
- 泛型 value。
- 回调清理函数。
- 最大容量限制。

不要第一版就做复杂。

---

## 九、项目达标标准

完成后你应该能说清：

- 哪些字段是共享状态。
- 为什么 `Get` 可能需要写锁。
- 后台 goroutine 如何退出。
- 为什么 `Close` 不能在持锁期间等待。
- race 测试覆盖了哪些路径。
