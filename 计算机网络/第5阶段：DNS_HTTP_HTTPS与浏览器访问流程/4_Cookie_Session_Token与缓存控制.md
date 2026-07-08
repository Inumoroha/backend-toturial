# 4. Cookie、Session、Token 与缓存控制

本节目标：理解 HTTP 无状态、Cookie、Session、Token 的关系，并掌握基础缓存控制。

---

## 一、HTTP 无状态

HTTP 协议本身不会记住上一次请求是谁发的。

所以登录态需要额外机制：

```text
Cookie
Session
Token
Authorization Header
```

---

## 二、Cookie

服务端设置：

```http
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

浏览器后续请求自动携带：

```http
Cookie: session_id=abc123
```

常见属性：

- `HttpOnly`：禁止 JS 读取。
- `Secure`：只在 HTTPS 发送。
- `SameSite`：降低 CSRF 风险。
- `Max-Age` / `Expires`：过期时间。

---

## 三、Session

Session 通常是服务端保存的会话状态。

流程：

```text
用户登录。
服务端生成 session_id。
session_id 写入 Cookie。
服务端用 session_id 查用户状态。
```

Session 可以存：

- 内存。
- Redis。
- 数据库。

多实例服务通常不应该只存本机内存。

---

## 四、Token

Token 是认证凭据，可以放在：

```http
Authorization: Bearer xxx
```

也可以放在 Cookie。

不要混淆：

```text
Cookie 是浏览器存储和发送机制。
Token 是认证内容。
```

---

## 五、缓存控制

常见响应头：

```http
Cache-Control: max-age=60
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

强缓存：

```text
缓存没过期，浏览器直接使用本地缓存。
```

协商缓存：

```text
浏览器问服务端资源是否变化，未变化返回 304。
```

敏感接口：

```http
Cache-Control: no-store
```

静态资源：

```http
Cache-Control: public, max-age=31536000, immutable
```

---

## 补充实验：用 curl 观察 Cookie 往返

写一个最小登录态示例：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
        http.SetCookie(w, &http.Cookie{
            Name:     "session_id",
            Value:    "abc123",
            Path:     "/",
            HttpOnly: true,
            SameSite: http.SameSiteLaxMode,
        })
        fmt.Fprintln(w, "logged in")
    })

    http.HandleFunc("/me", func(w http.ResponseWriter, r *http.Request) {
        c, err := r.Cookie("session_id")
        if err != nil {
            http.Error(w, "not logged in", http.StatusUnauthorized)
            return
        }
        fmt.Fprintln(w, "session:", c.Value)
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

第一次登录并保存 Cookie：

```bash
curl -i -c cookie.txt http://127.0.0.1:8080/login
```

带 Cookie 访问：

```bash
curl -i -b cookie.txt http://127.0.0.1:8080/me
```

观察：

```text
响应里的 Set-Cookie。
curl 保存的 cookie.txt。
下一次请求里的 Cookie。
```

这能帮助你理解：Cookie 是浏览器或客户端负责保存并自动带回的机制，Session 是服务端根据 Cookie 查到的登录状态。

---

## 补充实践：不同接口的缓存策略

可以按接口类型选择：

```text
登录、个人信息、订单：Cache-Control: no-store
接口列表短缓存：Cache-Control: private, max-age=30
公共静态资源：Cache-Control: public, max-age=31536000, immutable
需要协商缓存的资源：ETag + If-None-Match
```

敏感接口不要只写：

```http
Cache-Control: no-cache
```

`no-cache` 的意思是使用缓存前必须向服务端确认，不等于“不存储”。真正不希望缓存落盘时应使用：

```http
Cache-Control: no-store
```

这在登录态、支付、个人隐私接口里很重要。

---

## 六、常见问题

### 1. Cookie 等于 Session 吗？

不等于。Cookie 在客户端，Session 通常在服务端。

### 2. Token 一定要放 LocalStorage 吗？

不一定，也可以放 Cookie。选择要结合 XSS、CSRF、防护策略。

### 3. 用户隐私接口能缓存吗？

通常不应该缓存，至少要非常谨慎设置缓存头。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 解释 HTTP 无状态。
- 区分 Cookie、Session、Token。
- 解释 Cookie 常见安全属性。
- 解释强缓存和协商缓存。
- 根据接口类型选择基本缓存策略。
