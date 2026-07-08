# 3. Header 和 Cookie

本节目标：学会读取和设置 Header、Cookie。它们是鉴权、内容协商、浏览器状态管理中经常用到的基础能力。

---

## 一、读取请求 Header

读取 Header：

```go
ua := r.Header.Get("User-Agent")
auth := r.Header.Get("Authorization")
```

示例：

```go
mux.HandleFunc("GET /headers", func(w http.ResponseWriter, r *http.Request) {
	userAgent := r.Header.Get("User-Agent")
	authorization := r.Header.Get("Authorization")

	fmt.Fprintf(w, "user_agent=%s\n", userAgent)
	fmt.Fprintf(w, "authorization=%s\n", authorization)
})
```

验证：

```bash
curl -i http://localhost:8080/headers `
  -H "Authorization: Bearer dev-token"
```

---

## 二、设置响应 Header

设置纯文本响应：

```go
w.Header().Set("Content-Type", "text/plain; charset=utf-8")
```

设置 JSON 响应：

```go
w.Header().Set("Content-Type", "application/json")
```

设置自定义 Header：

```go
w.Header().Set("X-Request-ID", "req-123")
```

注意：Header 必须在写状态码或响应体前设置。

---

## 三、Authorization Header

很多 API 使用：

```text
Authorization: Bearer token
```

读取：

```go
auth := r.Header.Get("Authorization")
if auth != "Bearer dev-token" {
	http.Error(w, "unauthorized", http.StatusUnauthorized)
	return
}
```

这只是学习示例。真实项目中，Token 需要校验签名、过期时间、权限范围等。

---

## 四、读取 Cookie

浏览器会自动携带同域 Cookie。

读取：

```go
cookie, err := r.Cookie("session_id")
if err != nil {
	http.Error(w, "missing session", http.StatusUnauthorized)
	return
}

fmt.Fprintln(w, cookie.Value)
```

如果 Cookie 不存在，`r.Cookie` 会返回错误。

---

## 五、设置 Cookie

示例：

```go
http.SetCookie(w, &http.Cookie{
	Name:     "session_id",
	Value:    "abc123",
	Path:     "/",
	HttpOnly: true,
	SameSite: http.SameSiteLaxMode,
})
```

常见属性：

- `Name`：Cookie 名称。
- `Value`：Cookie 值。
- `Path`：Cookie 对哪些路径生效。
- `HttpOnly`：禁止 JavaScript 读取，降低 XSS 风险。
- `Secure`：只在 HTTPS 下发送。
- `SameSite`：控制跨站请求是否携带 Cookie。
- `MaxAge`：有效期。

学习环境可以不设置 `Secure`，生产 HTTPS 环境通常应该设置。

---

## 六、登录示例

简单示例：

```go
mux.HandleFunc("POST /login", func(w http.ResponseWriter, r *http.Request) {
	http.SetCookie(w, &http.Cookie{
		Name:     "session_id",
		Value:    "dev-session",
		Path:     "/",
		HttpOnly: true,
		SameSite: http.SameSiteLaxMode,
	})

	fmt.Fprintln(w, "logged in")
})
```

验证：

```bash
curl -i -X POST http://localhost:8080/login
```

你会看到响应里有：

```http
Set-Cookie: session_id=dev-session; Path=/; HttpOnly; SameSite=Lax
```

---

## 七、带 Cookie 请求

curl 手动携带 Cookie：

```bash
curl -i http://localhost:8080/profile `
  -H "Cookie: session_id=dev-session"
```

服务端读取：

```go
mux.HandleFunc("GET /profile", func(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie("session_id")
	if err != nil || cookie.Value != "dev-session" {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}

	fmt.Fprintln(w, "profile")
})
```

---

## 八、Header 和 Cookie 怎么选

API 服务常见选择：

- 前后端分离、移动端、第三方调用：常用 Authorization Header。
- 浏览器 Web 应用：常用 Cookie + Session。
- 简单内部服务：可以用固定 Header Token，但要注意安全边界。

本教程不是认证系统专项，当前阶段只需要知道它们在 `net/http` 中怎么读写。

---

## 九、本节检查点

请确认你能做到：

- 读取 `Authorization` Header。
- 设置 `Content-Type`。
- 设置自定义响应 Header。
- 设置 Cookie。
- 读取 Cookie。
- 解释 `HttpOnly` 的作用。

