# 3. Range 语义与类型封装

本节目标：理解 `sync.Map.Range` 的遍历语义，并用泛型封装改善类型安全。

---

## 一、Range 基本用法

```go
m.Range(func(key, value any) bool {
    fmt.Println(key, value)
    return true
})
```

返回 `true` 表示继续遍历，返回 `false` 表示停止。

---

## 二、Range 不是一致快照

`Range` 遍历期间，其他 goroutine 仍然可以 `Store` 或 `Delete`。

这意味着：

```text
Range 不保证看到遍历开始那一刻的完整快照；
Range 期间的修改是否被看到，不应该作为业务正确性依赖。
```

如果你需要一致快照，应该自己加锁并复制数据，或者使用其他设计。

---

## 三、Range 中不要做太慢的事情

如果遍历函数里做很慢的操作，会拖慢整个遍历过程。

建议：

```text
Range 中收集必要 key/value；
遍历结束后再做慢操作。
```

---

## 四、泛型封装

可以封装一个类型安全 Map：

```go
type TypedMap[K comparable, V any] struct {
    m sync.Map
}

func (tm *TypedMap[K, V]) Store(key K, value V) {
    tm.m.Store(key, value)
}

func (tm *TypedMap[K, V]) Load(key K) (V, bool) {
    value, ok := tm.m.Load(key)
    if !ok {
        var zero V
        return zero, false
    }
    v, ok := value.(V)
    if !ok {
        var zero V
        return zero, false
    }
    return v, true
}
```

这能改善调用侧类型体验，但内部仍然依赖类型断言。

---

## 五、什么时候不用 sync.Map

如果你需要：

- 按多个字段维护不变量。
- 做一致快照。
- 频繁批量更新。
- 强类型和清晰业务方法。

优先考虑：

```text
普通 map + Mutex/RWMutex
```

---

## 六、本节练习

1. 使用 `Range` 打印所有缓存项。
2. 在 `Range` 同时并发写入，观察语义不适合当快照。
3. 实现泛型 `TypedMap`。
4. 对比 `TypedMap` 和 `map + RWMutex` 的可读性。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 使用 `Range` 遍历 `sync.Map`。
- 知道 `Range` 不提供一致快照。
- 能用泛型封装基本类型安全。
- 能判断什么时候不该用 `sync.Map`。
