# 2. Buffer 复用与状态清理

本节目标：通过复用 `bytes.Buffer` 理解 `sync.Pool` 最常见的工程用法。

---

## 一、为什么复用 Buffer

在高并发 HTTP 服务中，经常需要临时拼接响应、日志或编码数据。

如果每次都创建新的 buffer：

```go
var buf bytes.Buffer
```

在高频调用下可能产生较多分配，增加 GC 压力。

`sync.Pool` 可以复用这些临时对象。

---

## 二、正确复用方式

```go
var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func BuildMessage(name string) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufferPool.Put(buf)

    buf.WriteString("hello, ")
    buf.WriteString(name)
    return buf.String()
}
```

关键点：

- `Get` 后立即 `Reset`。
- 使用结束后 `Put`。
- `Put` 后不再使用。

---

## 三、脏数据问题

如果忘记 `Reset`：

```go
func BuildMessage(name string) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)

    buf.WriteString("hello, ")
    buf.WriteString(name)
    return buf.String()
}
```

下一次取到同一个 buffer 时，里面可能还残留上次内容。

这类 bug 很隐蔽，尤其在日志、响应拼接、加密前数据处理里很危险。

---

## 四、敏感数据要清理

如果对象中包含：

- token。
- 密码。
- 用户隐私。
- 请求 body。

放回池前要确保清理干净。

对于 `[]byte`，可以考虑清零：

```go
for i := range b {
    b[i] = 0
}
```

是否需要清零要根据安全要求和性能成本权衡。

---

## 五、不要复用过大的对象

如果某次请求让 buffer 扩容到很大，直接放回池可能导致后续长期占用大内存。

可以设置上限：

```go
if buf.Cap() <= 64*1024 {
    bufferPool.Put(buf)
}
```

---

## 六、本节练习

1. 用 `sync.Pool` 复用 `bytes.Buffer`。
2. 故意忘记 `Reset`，写测试观察脏数据。
3. 增加最大容量限制。
4. 思考哪些业务对象不应该放入池。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 正确复用 buffer。
- 识别脏数据风险。
- 针对大对象设置放回策略。
- 知道敏感数据需要额外清理。
