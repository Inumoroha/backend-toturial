# 0. Context 基础认知总览

本阶段目标：理解 `context.Context` 是什么、能做什么、不能做什么，并掌握最基础的 API。

这一阶段不要急着写复杂项目。先把下面几个问题搞清楚：

- `context.Context` 为什么是接口？
- `Background()` 和 `TODO()` 有什么区别？
- `Done()`、`Err()`、`Deadline()`、`Value()` 分别代表什么？
- 为什么 `ctx context.Context` 通常放在函数第一个参数？
- 为什么不能传 `nil` context？

---

## 一、Context 的核心定位

Go 官方文档对 `context` 的定位可以概括为：

```text
在 API 边界之间传递截止时间、取消信号和请求级值。
```

也就是说，它不是业务对象，而是请求链路的控制信息。

常见函数签名：

```go
func DoSomething(ctx context.Context, input Input) error
```

这里 `ctx` 放在第一个参数，是 Go 社区约定。

这样调用链会很清楚：

```go
handler(ctx)
service.Do(ctx)
repo.Find(ctx)
db.QueryContext(ctx)
```

---

## 二、本阶段文档安排

建议按下面顺序学习：

1. `1_Context接口的四个方法.md`
2. `2_Background与TODO.md`
3. `3_Done_Err_Deadline_Value.md`
4. `4_第一个Context版任务.md`

学完后，你应该能看懂大多数代码里的 `ctx` 是怎么来的、传到哪里去、什么时候生效。

---

## 三、本阶段统一示例

本阶段会反复使用一个模拟任务：

```go
func slowWork(ctx context.Context) error {
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

这个函数表达了 context 最常见的使用方式：

```text
要么任务完成。
要么 context 被取消。
```

---

## 四、学习重点

你需要特别注意两点。

### 1. Context 只是发信号

`context` 不会强行停止你的代码。

如果函数内部完全不检查 `ctx.Done()`，那取消信号就没有效果。

### 2. Context 应该向下传递

不要在每一层都重新创建 `context.Background()`。

如果底层代码重新创建根 context，就会丢失上游的取消和超时信息。

---

## 五、本阶段达标标准

学完本阶段后，你应该能够做到：

- 说出 `Context` 接口的四个方法。
- 理解 `Background()` 和 `TODO()` 的区别。
- 使用 `ctx.Done()` 监听取消。
- 使用 `ctx.Err()` 判断取消原因。
- 知道 `Deadline()` 用来查看截止时间。
- 知道 `Value()` 只适合请求级元数据。
- 写出一个支持 context 取消的函数。

---

## 六、本阶段核心心智模型

可以把 context 理解成一条请求链路上的“控制信封”。

这个信封里不是业务数据主体，而是控制信息：

```text
这个请求什么时候结束？
这个请求是否已经取消？
这个请求有没有请求级标识？
```

函数接收 ctx，就表示它愿意尊重这些控制信息。

---

## 七、Context 的三个常见来源

### 1. 程序入口

```go
ctx := context.Background()
```

常见于 `main`、初始化、测试。

### 2. HTTP 请求

```go
ctx := r.Context()
```

常见于 Web 服务。

### 3. 父 context 派生

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
```

常见于下游调用、子任务、超时控制。

---

## 八、不要把 Context 学成模板

很多人会背：

```go
ctx, cancel := context.WithTimeout(...)
defer cancel()
```

但不知道为什么。

本阶段要建立的问题意识：

```text
这个 ctx 的父节点是谁？
它什么时候取消？
谁会监听它？
取消后错误是什么？
```

能回答这些问题，才是真正理解。

---

## 九、本阶段额外练习

写一个函数：

```go
func Do(ctx context.Context, name string, cost time.Duration) error
```

要求：

- 打印任务开始。
- cost 时间后成功。
- ctx 取消时返回 `ctx.Err()`。
- main 中分别使用 `Background`、`WithCancel`、`WithTimeout` 调用。

观察三种调用方式的差异。
