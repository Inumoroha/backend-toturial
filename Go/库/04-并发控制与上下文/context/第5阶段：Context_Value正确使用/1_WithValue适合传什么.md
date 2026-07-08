# 1. WithValue 适合传什么

本节目标：理解 `context.WithValue` 的使用边界。

基本写法：

```go
ctx = context.WithValue(ctx, key, value)
```

读取：

```go
value := ctx.Value(key)
```

看起来很像 map，但不要把它当作普通 map 使用。

---

## 一、适合放 request id

```go
ctx = WithRequestID(ctx, "req-123")
```

request id 是请求级元数据。

它不是业务函数的核心参数，但日志、链路追踪、排障都需要它。

适合放入 context。

---

## 二、适合放 trace id

链路追踪系统需要把 trace 信息传过很多层。

业务函数通常不关心 trace 细节，但日志、监控、下游调用会用到。

这类数据适合放 context。

---

## 三、适合放认证结果

认证中间件解析 token 后，可以把用户身份放入 context：

```go
type AuthUser struct {
	ID    int64
	Role  string
	Email string
}
```

Handler 或 service 可以读取当前用户：

```go
user, ok := AuthUserFromContext(ctx)
```

注意：如果某个业务方法必须明确操作某个 userID，仍然建议把 userID 放在函数参数里。

---

## 四、不适合放业务参数

不推荐：

```go
ctx = context.WithValue(ctx, "user_id", userID)
service.GetUser(ctx)
```

推荐：

```go
service.GetUser(ctx, userID)
```

原因：

- 函数签名不清楚。
- 调用方不知道必须提前放什么 value。
- 类型断言容易失败。
- 重构困难。

---

## 五、不适合放依赖对象

不推荐：

```go
ctx = context.WithValue(ctx, "db", db)
ctx = context.WithValue(ctx, "repo", repo)
```

数据库连接、repository、service 应该通过结构体依赖注入。

例如：

```go
type UserService struct {
	repo *UserRepo
}
```

context 不是依赖注入容器。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 判断一个值是否适合放 context。
- 解释 request id 为什么适合。
- 解释业务参数为什么不适合。
- 解释依赖对象为什么不适合。

---

## 七、判断表

| 值 | 是否适合 context | 原因 |
| --- | --- | --- |
| request id | 适合 | 请求级横切元数据 |
| trace id | 适合 | 跨服务追踪 |
| auth user | 适合 | 中间件解析后下游共享 |
| userID 查询参数 | 不适合 | 业务必需参数 |
| orderID | 不适合 | 业务必需参数 |
| db pool | 不适合 | 应用依赖 |
| logger | 通常不直接放 | 可以从 ctx 提取字段构造 logger |
| config | 不适合 | 应用配置 |

---

## 八、业务参数为什么必须显式

函数签名就是契约。

```go
func GetOrder(ctx context.Context) (*Order, error)
```

调用方看不出必须提前把 orderID 放进 ctx。

而：

```go
func GetOrder(ctx context.Context, orderID int64) (*Order, error)
```

一眼就清楚。

这能减少隐式依赖。

---

## 九、认证用户的边界

认证用户适合放 context，但仍然要小心。

例如：

```go
currentUser, ok := AuthUserFromContext(ctx)
```

表示“当前请求是谁发起的”。

但如果业务是查询某个用户：

```go
GetUser(ctx, targetUserID)
```

targetUserID 仍然应该作为参数。

---

## 十、常见错误

### 1. 用 context 做参数袋

这会让函数依赖变得不可见。

### 2. 放大对象

大对象会跟随 context 链路传递，不利于资源管理。

### 3. 放可变对象

多个 goroutine 可能同时读写，产生数据竞争。

---

## 十一、本节练习

判断下面哪些适合放入 context：

1. request id。
2. 当前登录用户。
3. 要查询的商品 id。
4. 数据库连接池。
5. trace id。
6. 分页参数 page 和 pageSize。

请写出理由。

---

## 十二、答案参考

1. request id：适合。它是请求级元数据。
2. 当前登录用户：适合。它是认证中间件解析出的请求级身份。
3. 要查询的商品 id：不适合。它是业务参数。
4. 数据库连接池：不适合。它是应用依赖。
5. trace id：适合。它是链路追踪元数据。
6. 分页参数：不适合。它是业务查询参数。

如果你判断不确定，就问：

```text
这个函数没有它还能不能表达清楚自己的业务？
```

如果不能，就放函数参数。

---

## 十三、真实项目建议

在项目早期就约定允许进入 context value 的类型。

例如：

```text
允许：request id、trace id、auth user、tenant id。
禁止：业务 id、分页参数、数据库连接、配置、service 对象。
```

这个约定可以减少后期代码风格分裂。
