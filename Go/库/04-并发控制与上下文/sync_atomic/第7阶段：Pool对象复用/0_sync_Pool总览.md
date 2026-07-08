# 0. sync.Pool 总览

本阶段目标：理解 `sync.Pool` 的对象复用机制，知道它适合减少临时对象分配和 GC 压力，但不能作为可靠缓存。

`sync.Pool` 解决的问题是：

```text
高频创建的临时对象能否复用，以减少内存分配和 GC 压力？
```

---

## 一、基本写法

```go
var pool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

buf := pool.Get().(*bytes.Buffer)
buf.Reset()
defer pool.Put(buf)
```

---

## 二、适合场景

适合复用：

- `bytes.Buffer`
- 临时 `[]byte`
- 编码解码结构体。
- 请求处理中的临时大对象。

不适合：

- 业务缓存。
- 连接池。
- 必须可靠保存的对象。
- 带用户敏感数据且未清理的对象。

---

## 三、本阶段文件安排

1. `1_Get_Put_New基础.md`
2. `2_Buffer复用与状态清理.md`
3. `3_Pool与GC及性能测试.md`

---

## 四、关键限制

`Pool` 中的对象可能在任意 GC 后被清理。

所以不能依赖：

```text
我 Put 进去的对象，未来一定能 Get 出来。
```

`Pool` 只是一种优化，不应该影响业务正确性。

---

## 五、本阶段达标标准

学完本阶段后，你应该能做到：

- 使用 `Get`、`Put`、`New`。
- 放回对象前清理状态。
- 写 benchmark 比较分配情况。
- 知道 `Pool` 不能当业务缓存或连接池。
