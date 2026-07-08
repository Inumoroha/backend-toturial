# 0. Atomic 原子操作总览

本阶段目标：掌握 `sync/atomic` 的常见用法，理解它适合简单共享状态，不适合复杂业务不变量。

`atomic` 解决的问题是：

```text
对单个变量进行并发安全的原子读写或更新。
```

---

## 一、推荐使用 typed atomic

现代 Go 中推荐使用类型化 atomic：

```go
var counter atomic.Int64

counter.Add(1)
value := counter.Load()
counter.Store(0)
```

常见类型：

- `atomic.Int64`
- `atomic.Uint64`
- `atomic.Bool`
- `atomic.Pointer[T]`

---

## 二、本阶段文件安排

1. `1_原子计数器.md`
2. `2_atomic_Bool与状态开关.md`
3. `3_atomic与Mutex取舍.md`

---

## 三、适合场景

适合：

- 请求计数。
- 成功失败数量统计。
- 简单开关。
- 只读配置指针替换。
- 热路径上的简单数值状态。

不适合：

- 多字段一致性。
- 复杂状态机。
- 需要事务式更新。
- 需要维护 map、slice 等复合结构。

---

## 四、本阶段达标标准

学完本阶段后，你应该能做到：

- 用 `atomic.Int64` 实现计数器。
- 用 `atomic.Bool` 实现状态开关。
- 知道不能混用 atomic 访问和普通访问。
- 能判断 atomic 与 Mutex 的取舍。
