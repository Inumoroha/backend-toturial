# 2. Cookie 安全属性

本节目标：理解 Cookie 常见安全属性，并能在 `net/http` 中正确设置。

---

## 一、设置 Cookie

```go
http.SetCookie(w, &http.Cookie{
	Name:     "session_id",
	Value:    "abc123",
	Path:     "/",
	HttpOnly: true,
	Secure:   true,
	SameSite: http.SameSiteLaxMode,
})
```

---

## 二、HttpOnly

```go
HttpOnly: true
```

作用：禁止浏览器 JavaScript 读取 Cookie。

这可以降低 XSS 攻击后 Cookie 被脚本直接偷走的风险。

只要是 session 类 Cookie，通常都应该设置 `HttpOnly`。

---

## 三、Secure

```go
Secure: true
```

作用：只允许浏览器在 HTTPS 请求中携带 Cookie。

生产环境应该使用 HTTPS，并给敏感 Cookie 设置 Secure。

本地 HTTP 开发环境如果设置了 `Secure: true`，浏览器可能不会发送 Cookie，这是常见调试坑。

---

## 四、SameSite

常见值：

```go
SameSite: http.SameSiteLaxMode
SameSite: http.SameSiteStrictMode
SameSite: http.SameSiteNoneMode
```

作用：控制跨站请求时是否携带 Cookie，能帮助缓解 CSRF 风险。

常见选择：

- 普通 Web 应用：`Lax` 是常见起点。
- 高敏感场景：考虑 `Strict`。
- 跨站必须携带 Cookie：`None`，并且通常需要 `Secure`。

---

## 五、MaxAge 和 Expires

设置有效期：

```go
MaxAge: 3600
```

删除 Cookie：

```go
http.SetCookie(w, &http.Cookie{
	Name:   "session_id",
	Value:  "",
	Path:   "/",
	MaxAge: -1,
})
```

删除时 `Path` 要和设置时保持一致，否则可能删不掉。

---

## 六、本节检查点

请确认你能回答：

- `HttpOnly` 防什么？
- `Secure` 在本地开发时有什么坑？
- `SameSite` 大致解决什么问题？
- 删除 Cookie 时为什么要注意 Path？

