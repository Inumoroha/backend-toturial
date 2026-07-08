# 3. Token 鉴权边界

本节目标：理解简单 Bearer Token 鉴权在学习阶段怎么写，以及真实项目中还需要补哪些能力。

---

## 一、学习阶段的简单鉴权

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer dev-token" {
			writeError(w, http.StatusUnauthorized, "unauthorized", "missing or invalid token")
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

这适合学习中间件模型，但不是完整认证系统。

---

## 二、真实 Token 需要校验什么

真实项目通常需要检查：

- Token 是否存在。
- Token 格式是否正确。
- 签名是否有效。
- 是否过期。
- 是否被撤销。
- 用户是否存在。
- 用户是否有权限访问资源。

如果使用 JWT，还要谨慎处理算法、密钥、过期时间和刷新机制。

---

## 三、401 和 403

`401 Unauthorized`：

```text
你还没有通过认证，或者认证凭证无效。
```

`403 Forbidden`：

```text
你已经认证了，但没有权限访问这个资源。
```

例子：

```text
没有 Token -> 401
Token 错误 -> 401
普通用户访问管理员接口 -> 403
```

---

## 四、不要在日志中打印完整 Token

不推荐：

```go
log.Printf("auth=%s", r.Header.Get("Authorization"))
```

Token 属于敏感信息。日志系统通常多人可见，也可能被采集到第三方平台。

可以打印是否存在、用户 ID、Token 前后少量字符，但要谨慎。

---

## 五、本节检查点

请确认你能回答：

- 学习阶段固定 Token 有什么局限？
- 真实 Token 通常要校验哪些内容？
- 401 和 403 怎么区分？
- 为什么日志中不能打印完整 Token？

