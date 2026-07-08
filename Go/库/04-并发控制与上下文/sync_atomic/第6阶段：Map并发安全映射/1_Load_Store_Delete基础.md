# 1. Load、Store、Delete 基础

本节目标：掌握 `sync.Map` 最基础的读、写、删除操作。

---

## 一、Store

```go
var m sync.Map

m.Store("user:1", "Alice")
```

`Store` 用于写入或覆盖 key。

---

## 二、Load

```go
value, ok := m.Load("user:1")
if !ok {
    return
}

name := value.(string)
fmt.Println(name)
```

`Load` 返回 `any`，所以通常需要类型断言。

类型断言失败会 panic：

```go
name := value.(string)
```

如果不确定类型，可以写：

```go
name, ok := value.(string)
```

---

## 三、Delete

```go
m.Delete("user:1")
```

删除不存在的 key 不会报错。

---

## 四、类型安全问题

`sync.Map` 的 key 和 value 都是 `any`，这让它很灵活，也带来风险。

例如：

```go
m.Store("age", 18)
value, _ := m.Load("age")
name := value.(string)
```

这会 panic。

业务代码中如果 key/value 类型固定，`map + RWMutex` 或泛型封装通常更好。

---

## 五、最小缓存示例

```go
type UserCache struct {
    m sync.Map
}

func (c *UserCache) Set(id int64, name string) {
    c.m.Store(id, name)
}

func (c *UserCache) Get(id int64) (string, bool) {
    value, ok := c.m.Load(id)
    if !ok {
        return "", false
    }
    name, ok := value.(string)
    return name, ok
}

func (c *UserCache) Delete(id int64) {
    c.m.Delete(id)
}
```

---

## 六、本节练习

1. 用 `sync.Map` 实现用户名称缓存。
2. 写并发读写测试。
3. 故意存入错误类型，观察类型断言风险。
4. 用 `map + RWMutex` 实现同样功能并对比可读性。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 使用 `Store`、`Load`、`Delete`。
- 处理类型断言。
- 认识 `sync.Map` 的类型安全不足。
