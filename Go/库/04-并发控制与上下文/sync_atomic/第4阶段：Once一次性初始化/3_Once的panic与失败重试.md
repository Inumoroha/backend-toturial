# 3. Once 的 panic 与失败重试

本节目标：理解 `Once` 遇到 panic 或初始化失败时的行为，避免把它用在不适合的重试场景中。

---

## 一、panic 后会怎样

如果 `once.Do` 中的函数 panic，这次调用也会被认为已经执行过。

示例：

```go
var once sync.Once

func Init() {
    once.Do(func() {
        panic("init failed")
    })
}
```

第一次调用 panic 后，第二次再调用 `Init`，初始化函数不会再次执行。

这点非常重要。

---

## 二、返回错误怎么办

常见写法：

```go
var (
    once sync.Once
    cfg  *Config
    err  error
)

func GetConfig() (*Config, error) {
    once.Do(func() {
        cfg, err = LoadConfig()
    })
    return cfg, err
}
```

如果 `LoadConfig` 第一次失败，后续调用会一直返回同一个错误，不会重试。

这适合：

```text
初始化失败就是不可恢复错误；
希望所有调用方看到同一个失败结果。
```

不适合：

```text
配置服务短暂不可用；
网络抖动后希望重新初始化；
初始化依赖可能稍后恢复。
```

---

## 三、需要重试时怎么办

如果需要失败后重试，可以不用 `Once`，改用 `Mutex` 控制状态：

```go
type Loader struct {
    mu     sync.Mutex
    loaded bool
    cfg    *Config
}

func (l *Loader) Get() (*Config, error) {
    l.mu.Lock()
    defer l.mu.Unlock()

    if l.loaded {
        return l.cfg, nil
    }

    cfg, err := LoadConfig()
    if err != nil {
        return nil, err
    }

    l.cfg = cfg
    l.loaded = true
    return cfg, nil
}
```

这段代码的语义是：

```text
只有成功才标记 loaded；
失败时下次还可以重试。
```

---

## 四、不要重置 Once

不要尝试通过重新赋值来“重置”已经使用的 `Once`，尤其是在并发访问中。

如果业务需要可重置初始化，应该显式设计状态机，而不是把 `Once` 当成可重置开关。

---

## 五、本节练习

1. 写一个 `once.Do` 中 panic 的例子，观察第二次是否执行。
2. 写一个返回错误的初始化函数，观察错误是否会被缓存。
3. 用 `Mutex` 实现失败可重试的 loader。
4. 总结 `Once` 和 `Mutex loader` 的适用差异。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 说明 `Once` 中 panic 后不会重试。
- 说明初始化返回错误时的缓存行为。
- 在需要失败重试时选择 `Mutex` 或其他状态管理方案。
