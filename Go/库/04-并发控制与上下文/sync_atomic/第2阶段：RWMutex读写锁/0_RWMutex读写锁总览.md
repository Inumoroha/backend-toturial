# 0. RWMutex 读写锁总览

本阶段目标：掌握 `sync.RWMutex` 的使用场景，理解读锁和写锁的区别，并能判断什么时候它比 `Mutex` 更合适。

`RWMutex` 适合读多写少的共享状态。

它的核心规则是：

```text
多个读者可以同时持有读锁；
写者需要独占锁；
写锁和读锁不能同时持有。
```

---

## 一、本阶段要解决的问题

学完本阶段后，你应该能回答：

- `RLock` 和 `Lock` 的区别是什么？
- 为什么读操作也需要锁？
- `RWMutex` 一定比 `Mutex` 快吗？
- 什么是读多写少？
- 为什么不能从读锁直接升级成写锁？
- 如何用 benchmark 验证选择？

---

## 二、本阶段文件安排

1. `1_读锁与写锁模型.md`
2. `2_读多写少缓存.md`
3. `3_RWMutex常见误用.md`
4. `4_Mutex与RWMutex性能对比.md`

---

## 三、基本写法

```go
var mu sync.RWMutex

mu.RLock()
// read shared state
mu.RUnlock()

mu.Lock()
// write shared state
mu.Unlock()
```

工程代码里通常写成：

```go
func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    value, ok := c.items[key]
    return value, ok
}
```

写操作仍然使用 `Lock`：

```go
func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[key] = value
}
```

---

## 四、适合场景

适合：

- 读请求远多于写请求。
- 读取临界区相对稳定。
- 写操作不频繁。
- 共享状态结构比较清晰。

不一定适合：

- 写很多。
- 临界区很短。
- 读写比例接近。
- 锁竞争不明显。

---

## 五、本阶段达标标准

学完本阶段后，你应该能做到：

- 用 `RWMutex` 实现读多写少缓存。
- 知道读取共享 map 也需要锁。
- 能解释 `RWMutex` 不一定比 `Mutex` 快。
- 能写 benchmark 比较读多写少、读写均衡、写多读少场景。
