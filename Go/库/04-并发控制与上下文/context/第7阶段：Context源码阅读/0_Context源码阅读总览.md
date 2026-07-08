# 0. Context 源码阅读总览

本阶段目标：读懂 Go 标准库 `context` 的核心实现，建立更深一层的理解。

你不需要一开始就记住所有细节。

源码阅读重点是回答：

```text
Context 为什么能取消？
Done channel 什么时候创建？
父 context 如何通知子 context？
WithTimeout 如何自动取消？
WithValue 为什么是链式查找？
```

---

## 一、本阶段文档安排

建议按下面顺序学习：

1. `1_Context接口与emptyCtx.md`
2. `2_cancelCtx取消实现.md`
3. `3_timerCtx超时实现.md`
4. `4_valueCtx链式取值.md`
5. `5_propagateCancel取消传播.md`
6. `6_源码阅读总结.md`

---

## 二、阅读源码的方法

不要从头到尾硬读。

建议按 API 反推：

```text
调用 WithCancel
  -> 返回了什么类型？
  -> cancel 函数做了什么？
  -> Done channel 什么时候关闭？
```

这样读更有目标。

---

## 三、本阶段达标标准

学完本阶段后，你应该能够做到：

- 说出 `emptyCtx`、`cancelCtx`、`timerCtx`、`valueCtx` 的作用。
- 理解 context 的取消传播大致如何实现。
- 理解 `WithTimeout` 和 timer 的关系。
- 理解 `WithValue` 为什么会一层层查找。

---

## 四、源码阅读前的准备

阅读标准库源码前，建议先在本机找到 Go 安装目录：

```powershell
go env GOROOT
```

然后查看：

```text
$GOROOT/src/context/context.go
```

如果使用 VS Code，也可以在代码里按住 Ctrl 点击 `context.WithTimeout` 跳转到源码。

学习阶段不需要逐行背诵源码。更推荐准备一份问题清单，然后带着问题读。

例如：

```text
WithCancel 返回的 cancel 到底是什么？
Done channel 是创建 context 时就创建的吗？
父 context 取消时，子 context 为什么能知道？
WithTimeout 到期后是谁调用 cancel？
WithValue 查不到当前 key 时去哪里找？
```

这样读源码不会迷路。

---

## 五、源码阅读的三个层次

### 1. 看 API 层

先看导出的函数和接口：

```go
Background
TODO
WithCancel
WithTimeout
WithDeadline
WithValue
```

这一层回答：

```text
调用者看到什么？
函数签名是什么？
返回值是什么？
```

### 2. 看结构体层

再看内部类型：

```text
emptyCtx
cancelCtx
timerCtx
valueCtx
```

这一层回答：

```text
每种 context 保存了哪些状态？
每种 context 增加了什么能力？
```

### 3. 看传播层

最后看父子关系和取消传播。

这一层回答：

```text
父 context 取消时，子 context 怎么被取消？
取消时如何避免重复关闭 channel？
取消后如何释放 children 引用？
```

如果第一次读源码，卡在第三层很正常。先把前两层读懂，就已经很有价值。

---

## 六、建议边读边写的小实验

源码阅读不要只看。建议用小实验验证行为。

### 实验一：多次 cancel 是否安全

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()
cancel()
cancel()
fmt.Println(ctx.Err())
```

如果你读过 `cancelCtx`，就能理解为什么多次 cancel 不会 panic。

### 实验二：父取消是否影响子

```go
parent, parentCancel := context.WithCancel(context.Background())
child, childCancel := context.WithCancel(parent)
defer childCancel()

parentCancel()
fmt.Println(child.Err())
```

这能帮助你理解取消传播。

### 实验三：相同 key 的 Value

```go
type key string

const k key = "name"

ctx := context.Background()
ctx = context.WithValue(ctx, k, "old")
ctx = context.WithValue(ctx, k, "new")

fmt.Println(ctx.Value(k))
```

这能帮助你理解 `valueCtx` 的链式查找和外层遮蔽。

---

## 七、常见阅读误区

### 1. 试图一次读懂所有细节

标准库源码虽然不算特别长，但它处理了并发安全、父子传播、timer、兼容自定义 context 等细节。第一次读不懂全部很正常。

### 2. 只看结构体，不看调用路径

只看 `cancelCtx` 字段是不够的。你还需要看：

```text
WithCancel -> withCancel -> propagateCancel -> cancel
```

具体函数名可能随 Go 版本略有变化，但阅读思路不变。

### 3. 忽略并发安全

context 经常跨 goroutine 使用，所以源码里会有锁、原子值或延迟初始化。不要把它当作普通单线程数据结构理解。

---

## 八、本阶段学习建议

源码阶段可以分两轮：

第一轮只追求能画出结构：

```text
emptyCtx
cancelCtx
timerCtx
valueCtx
```

第二轮再追取消传播：

```text
parent 如何记录 child？
cancel 如何关闭 done？
timer 到期后如何触发 cancel？
```

如果能把这些讲给别人听，说明你已经不是只会用 API，而是理解了 context 的工程设计。
