# 0. Context Value 正确使用总览

本阶段目标：掌握 `context.WithValue` 的正确边界，避免把 context 当成万能 map。

`context.Value` 是 `context` 中最容易被滥用的能力。

它确实有用，但只适合请求级元数据。

---

## 一、本阶段文档安排

建议按下面顺序学习：

1. `1_WithValue适合传什么.md`
2. `2_类型安全的Key与封装函数.md`
3. `3_认证信息日志字段与TraceID.md`
4. `4_常见滥用案例.md`
5. `5_Context_Value综合练习.md`

---

## 二、Value 的正确定位

适合放入 context：

- request id。
- trace id。
- span context。
- 已解析的用户身份。
- logger 需要的请求级字段。
- 租户 id 等横切元数据。

不适合放入 context：

- 函数必需业务参数。
- 数据库连接。
- service 对象。
- repository 对象。
- 配置项。
- 大对象。
- 可变对象。

---

## 三、判断标准

判断一个值是否适合放入 context，可以问：

```text
它是否属于本次请求的横切元数据？
它是否不影响函数的核心业务签名？
它是否需要被很多层共享，但不是业务必需参数？
```

如果答案是“是”，可以考虑 context。

如果一个函数没有这个值就无法完成业务，应该放在参数里。

---

## 四、本阶段达标标准

学完本阶段后，你应该能够做到：

- 说出 `context.Value` 适合和不适合传什么。
- 使用自定义 key 类型避免冲突。
- 封装 `WithXxx` 和 `XxxFromContext`。
- 通过 context 传 request id、trace id、认证信息。
- 识别把业务参数塞进 context 的坏味道。

---

## 五、本阶段的核心判断

学 `Value` 时，最重要的不是记 API，而是判断边界。

你可以用一句话判断：

```text
如果这个值是业务函数完成任务必需的参数，就不要放 context。
```

例如：

```go
GetUser(ctx, userID)
```

这里 userID 是业务参数。

而 request id 是日志、追踪需要的横切信息：

```go
WithRequestID(ctx, requestID)
```

这才适合 context。

---

## 六、本阶段统一练习

做一个简单接口：

```text
GET /me
```

中间件负责：

- 设置 request id。
- 模拟认证用户。

handler 负责：

- 从 context 读取认证用户。
- 从 context 读取 request id 打日志。
- 返回当前用户信息。

这个练习能覆盖 `Value` 的正确用法。

---

## 七、常见错误预告

你要特别避免：

- string key 到处散落。
- 直接 `ctx.Value(key).(Type)`。
- 把 userID、orderID 这类业务参数放 context。
- 把 db、repo、client 放 context。
- 把 map 当作共享数据塞 context。

---

## 八、本阶段完成标准

你应该能写出：

```go
func WithRequestID(ctx context.Context, requestID string) context.Context
func RequestIDFromContext(ctx context.Context) (string, bool)
```

并解释为什么 key 类型不导出。

---

## 九、Value 学习的两个层次

第一层是会写：

```go
ctx = context.WithValue(ctx, key, value)
value := ctx.Value(key)
```

这一层很简单。

第二层是会判断：

```text
这个值该不该放 context？
这个 key 会不会冲突？
读取不到时怎么办？
这个 value 会不会被并发修改？
```

真正的工程能力在第二层。

---

## 十、和函数参数的边界

可以用下面的对比记忆：

```text
函数没有它就不能完成业务：函数参数。
很多层都可能需要，但不是业务主体：context value。
应用启动后长期存在的依赖：结构体字段。
```

例如：

```text
userID：函数参数。
requestID：context value。
db pool：结构体字段。
```

这个分类在代码 review 中非常有用。

---

## 十一、本阶段实践成果

学完本阶段后，建议你沉淀一个小包：

```text
internal/requestctx
  context.go
```

里面只放：

```text
RequestID
TraceID
AuthUser
TenantID
```

不要把业务参数、数据库连接、服务对象放进去。

这个边界越早建立，后面的 service 和 repository 代码越清爽。
