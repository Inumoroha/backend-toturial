# 2. LoadOrStore 与并发初始化

本节目标：掌握 `LoadOrStore` 的语义，理解它在并发初始化、缓存占位中的用途和限制。

---

## 一、LoadOrStore 的语义

```go
actual, loaded := m.LoadOrStore(key, value)
```

含义：

- 如果 key 已存在，返回已有值，`loaded == true`。
- 如果 key 不存在，保存新值，返回新值，`loaded == false`。

---

## 二、并发场景

多个 goroutine 同时执行：

```go
actual, loaded := m.LoadOrStore("k", "v")
```

只有一个 goroutine 会真正存储成功，其他 goroutine 会得到已存在值。

这可以避免简单的“先 Load 再 Store”竞态。

---

## 三、先 Load 再 Store 的问题

错误思路：

```go
if _, ok := m.Load(key); !ok {
    m.Store(key, value)
}
```

并发下多个 goroutine 可能都看到 key 不存在，然后都执行初始化逻辑。

`LoadOrStore` 能把检查和存储合成一个原子步骤。

---

## 四、初始化函数仍然可能重复执行

注意下面写法：

```go
value := expensiveInit()
actual, loaded := m.LoadOrStore(key, value)
```

`expensiveInit` 在 `LoadOrStore` 之前已经执行了。多个 goroutine 仍然可能重复执行昂贵初始化，只是最终只有一个值被存进去。

如果要避免重复执行初始化函数，需要更复杂的 singleflight 模式。

---

## 五、适合后端场景

适合：

- 简单缓存占位。
- key 对应的 value 创建成本不高。
- 只关心最终 map 中只有一个值。

不适合：

- 初始化非常昂贵。
- 初始化有副作用。
- 失败要传播给等待者。

这时可以了解 `golang.org/x/sync/singleflight`。

---

## 六、本节练习

1. 使用 `LoadOrStore` 实现用户缓存。
2. 并发调用同一个 key，统计成功存储次数。
3. 把 value 改成昂贵初始化函数，观察初始化仍可能重复执行。
4. 阅读 singleflight 的用途。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 正确解释 `actual` 和 `loaded`。
- 知道 `LoadOrStore` 避免的是存储竞态，不一定避免初始化重复。
- 能判断是否需要 singleflight。
