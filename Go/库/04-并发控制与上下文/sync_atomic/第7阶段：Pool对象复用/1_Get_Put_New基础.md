# 1. Get、Put、New 基础

本节目标：掌握 `sync.Pool` 的基本 API 和对象生命周期。

---

## 一、New 函数

```go
var pool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 1024)
    },
}
```

当池中没有可用对象时，`Get` 会调用 `New` 创建对象。

如果没有设置 `New`，`Get` 可能返回 `nil`。

---

## 二、Get

```go
buf := pool.Get().([]byte)
```

`Get` 从池中取出一个对象。你不能假设它是刚刚被你放进去的对象。

---

## 三、Put

```go
pool.Put(buf)
```

`Put` 把对象放回池中，供后续复用。

注意：放回前要清理状态。

---

## 四、最小示例

```go
var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func Encode(s string) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufferPool.Put(buf)

    buf.WriteString("prefix:")
    buf.WriteString(s)
    return buf.String()
}
```

---

## 五、不要 Put 后继续使用

错误示例：

```go
buf := pool.Get().(*bytes.Buffer)
pool.Put(buf)
buf.WriteString("still use")
```

对象放回池后，其他 goroutine 可能取走并修改它。Put 后继续使用会导致数据竞争或脏数据。

---

## 六、本节练习

1. 创建一个 `bytes.Buffer` 池。
2. 实现一个字符串拼接函数。
3. 忘记 `Reset`，观察脏数据。
4. Put 后继续使用对象，用 `go test -race` 观察风险。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 正确使用 `New`、`Get`、`Put`。
- 知道对象放回池后不能再使用。
- 知道放回前要重置状态。
