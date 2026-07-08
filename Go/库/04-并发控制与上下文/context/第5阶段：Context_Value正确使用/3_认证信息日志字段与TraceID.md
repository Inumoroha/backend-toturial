# 3. 认证信息、日志字段与 Trace ID

本节目标：学习 `context.Value` 在真实后端项目中的三个典型场景。

它们分别是：

- 当前认证用户。
- 日志字段。
- 链路追踪 trace id。

---

## 一、认证信息

认证中间件通常会做：

```text
读取 Authorization header。
校验 token。
解析用户身份。
把认证结果放入 context。
```

定义类型：

```go
type AuthUser struct {
	ID    int64
	Email string
	Role  string
}
```

封装：

```go
type key string

const authUserKey key = "auth_user"

func WithAuthUser(ctx context.Context, user AuthUser) context.Context {
	return context.WithValue(ctx, authUserKey, user)
}

func AuthUserFromContext(ctx context.Context) (AuthUser, bool) {
	v, ok := ctx.Value(authUserKey).(AuthUser)
	return v, ok
}
```

---

## 二、日志字段

日志中常用字段：

```text
request_id
trace_id
user_id
path
method
```

其中 request id、trace id、当前用户身份适合来自 context。

示例：

```go
func logWithContext(ctx context.Context, msg string) {
	requestID, _ := RequestIDFromContext(ctx)
	traceID, _ := TraceIDFromContext(ctx)

	fmt.Printf("request_id=%s trace_id=%s msg=%s\n", requestID, traceID, msg)
}
```

真实项目里一般会使用结构化日志库，例如 zap、zerolog、slog。

---

## 三、Trace ID

Trace ID 用来串联跨服务调用。

一次请求可能经过：

```text
api-service -> user-service -> order-service -> payment-service
```

如果每个服务都带着同一个 trace id，排查问题会容易很多。

在真实项目中，OpenTelemetry 会帮你处理很多上下文传播细节。

但学习阶段可以先理解：

```text
trace id 是请求级横切信息，适合跟着 context 走。
```

---

## 四、不要把认证用户和业务 userID 混为一谈

当前认证用户：

```text
谁正在发起请求。
```

业务参数 userID：

```text
要查询或操作哪个用户。
```

比如管理员查询用户 1001：

```text
认证用户：管理员 1
业务参数：用户 1001
```

认证用户可以放 context。

业务参数应该放函数参数：

```go
GetUser(ctx, userID)
```

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 用 context 传递已认证用户。
- 区分认证用户和业务 userID。
- 用 context 中的 request id / trace id 生成日志字段。
- 理解链路追踪为什么依赖 context 传播。

---

## 六、认证中间件完整示例

```go
func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token := r.Header.Get("Authorization")
		if token == "" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		user := AuthUser{
			ID:    1001,
			Email: "tom@example.com",
			Role:  "user",
		}

		ctx := WithAuthUser(r.Context(), user)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

学习阶段可以假装 token 校验成功。

真实项目需要校验 JWT、Session 或调用认证服务。

---

## 七、日志函数示例

```go
func logInfo(ctx context.Context, msg string) {
	requestID, _ := RequestIDFromContext(ctx)
	traceID, _ := TraceIDFromContext(ctx)
	user, _ := AuthUserFromContext(ctx)

	fmt.Printf(
		"level=info request_id=%s trace_id=%s user_id=%d msg=%s\n",
		requestID,
		traceID,
		user.ID,
		msg,
	)
}
```

真实项目建议用结构化日志库，不要手拼字符串。

---

## 八、Trace ID 和 Request ID 的区别

简单理解：

```text
request id：标识一次服务入口请求。
trace id：标识一次跨服务调用链。
```

在单体应用里，它们可能看起来差不多。

在微服务里，一个 trace id 可能贯穿多个服务。

---

## 九、常见错误

### 1. 把 token 原文到处传

认证中间件应该解析 token，把需要的信息放入 context，而不是让所有业务层都解析 token。

### 2. 把权限判断全塞到 context 工具里

context 工具只负责读写，权限判断应该在业务层或专门鉴权组件里。

### 3. 混淆 current user 和 target user

管理员操作其他用户时尤其容易出错。

---

## 十、本节练习

实现：

```text
GET /me
GET /users/1001
```

要求：

- `/me` 返回 context 中的当前认证用户。
- `/users/1001` 的 1001 是业务参数，不从 context 取。
- 日志同时打印 request id 和 current user id。
