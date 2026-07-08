# 4. CORS 白名单实践

本节目标：把 CORS 从“能跑”写法改成更安全的白名单写法。

---

## 一、不要随便允许所有来源

简单写法：

```go
w.Header().Set("Access-Control-Allow-Origin", "*")
```

对于公开、无凭证 API 可能可以接受，但如果涉及用户身份、Cookie、Authorization，就要谨慎。

更推荐白名单：

```go
allowedOrigins := map[string]bool{
	"http://localhost:5173": true,
	"https://example.com":   true,
}
```

---

## 二、白名单 CORS 中间件

```go
func cors(allowed map[string]bool) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")
			if allowed[origin] {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Vary", "Origin")
				w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PATCH, DELETE, OPTIONS")
				w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
			}

			if r.Method == http.MethodOptions {
				w.WriteHeader(http.StatusNoContent)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

---

## 三、为什么要设置 Vary: Origin

```go
w.Header().Set("Vary", "Origin")
```

这告诉缓存系统：不同 Origin 可能得到不同响应。

如果不设置，代理或 CDN 缓存可能错误复用某个 Origin 的 CORS 响应。

---

## 四、CORS 不是后端鉴权

CORS 是浏览器安全机制。它不能替代服务端鉴权。

攻击者仍然可以用：

- curl。
- 自己写程序。
- Postman。

直接请求你的后端。

所以敏感接口必须在服务端做认证和授权。

---

## 五、本节检查点

请确认你能回答：

- 为什么不建议随便 `Access-Control-Allow-Origin: *`？
- 白名单 CORS 怎么写？
- `Vary: Origin` 有什么意义？
- 为什么 CORS 不能替代鉴权？

