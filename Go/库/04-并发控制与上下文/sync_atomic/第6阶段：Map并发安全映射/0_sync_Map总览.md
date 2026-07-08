# 0. sync.Map 总览

本阶段目标：理解 `sync.Map` 的适用场景和限制，知道它不是普通 map 的万能替代品。

`sync.Map` 是标准库提供的并发安全映射结构。

---

## 一、基本 API

```go
var m sync.Map

m.Store("key", "value")
value, ok := m.Load("key")
m.Delete("key")
actual, loaded := m.LoadOrStore("key", "value")
m.Range(func(key, value any) bool {
    return true
})
```

---

## 二、适合场景

`sync.Map` 适合：

- 读多写少。
- key 相对稳定。
- 多 goroutine 并发访问。
- 不想自己维护锁。
- 缓存类或一次写多次读场景。

不一定适合：

- 写入频繁。
- 需要复杂不变量。
- 需要类型安全。
- 需要一致快照遍历。

---

## 三、本阶段文件安排

1. `1_Load_Store_Delete基础.md`
2. `2_LoadOrStore与并发初始化.md`
3. `3_Range语义与类型封装.md`

---

## 四、与 map + RWMutex 的关系

很多业务代码中，`map + RWMutex` 更清晰：

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}
```

它的优点是：

- 类型明确。
- 不需要类型断言。
- 可以维护复杂业务规则。
- 更容易控制锁范围。

`sync.Map` 适合特定场景，不是默认选择。

---

## 五、本阶段达标标准

学完本阶段后，你应该能做到：

- 使用 `Load`、`Store`、`Delete`、`LoadOrStore`。
- 知道 `Range` 不是一致快照。
- 能用泛型封装类型安全 Map。
- 能判断 `sync.Map` 与 `map + RWMutex` 的取舍。
