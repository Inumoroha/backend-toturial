# 3. curl 与浏览器调试

本节目标：学会用浏览器和 `curl` 调试 HTTP 接口。后端工程师需要能独立验证接口，而不是完全依赖前端或图形化工具。

---

## 一、浏览器适合调试什么

浏览器最适合快速调试 GET 请求。

例如服务启动在本机 `8080` 端口时，可以在地址栏访问：

```text
http://localhost:8080/hello
http://localhost:8080/health
http://localhost:8080/search?q=go
```

浏览器的优点：

- 访问方便。
- 能直接看到响应内容。
- 开发者工具可以看请求和响应。

浏览器的限制：

- 不方便发送 POST、PATCH、DELETE。
- 不方便自定义 Header。
- 不方便精确查看状态码和原始响应。

所以后端开发不能只依赖浏览器。

---

## 二、curl 的基本用法

查看响应体：

```bash
curl http://localhost:8080/hello
```

查看响应头和状态码：

```bash
curl -i http://localhost:8080/hello
```

只查看响应头：

```bash
curl -I http://localhost:8080/hello
```

`-i` 对学习 HTTP 很有帮助，因为它能让你看到：

```http
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: Sun, 05 Jul 2026 09:00:00 GMT
Content-Length: 18

hello, net/http
```

这样你就能确认状态码、Header、Body 是否符合预期。

---

## 三、发送 Query 参数

Query 参数直接写在 URL 后：

```bash
curl -i "http://localhost:8080/search?q=go&page=1"
```

在 PowerShell 中建议给 URL 加引号，因为 `&` 等符号可能被终端解释。

服务端读取时会用：

```go
q := r.URL.Query().Get("q")
page := r.URL.Query().Get("page")
```

---

## 四、发送 JSON 请求

创建资源时经常发送 JSON：

```bash
curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -d "{\"title\":\"learn net/http\"}"
```

Linux/macOS 写法：

```bash
curl -i -X POST http://localhost:8080/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"learn net/http"}'
```

参数说明：

- `-X POST`：指定请求方法。
- `-H`：设置 Header。
- `-d`：设置请求体。

注意：使用 `-d` 时，curl 默认会发送 POST。如果你明确写 `-X POST`，可读性更好。

---

## 五、发送 Token

很多接口需要鉴权，常见方式是 Bearer Token：

```bash
curl -i http://localhost:8080/profile `
  -H "Authorization: Bearer dev-token"
```

服务端读取：

```go
auth := r.Header.Get("Authorization")
```

后面学习中间件时，我们会用这个 Header 做一个简单鉴权示例。

---

## 六、查看重定向

短链接服务会用到重定向。

只查看重定向响应：

```bash
curl -i http://localhost:8080/abc123
```

自动跟随重定向：

```bash
curl -L -i http://localhost:8080/abc123
```

`-L` 表示 follow redirect。

重定向响应通常长这样：

```http
HTTP/1.1 302 Found
Location: https://example.com
```

---

## 七、常见排查方式

如果接口不符合预期，按这个顺序看：

1. 服务是否启动成功。
2. 请求 URL 是否正确。
3. Method 是否正确。
4. Header 是否正确。
5. Body 是否真的发出去了。
6. 服务端日志是否打印了请求。
7. 响应状态码是否符合预期。
8. 响应 Body 是否包含错误信息。

后端排错要尽量把问题拆开。不要只说“接口不通”，而是要具体判断：

```text
是连接不上？
是路由没匹配？
是方法不对？
是 JSON 解析失败？
是业务校验失败？
是服务端 panic？
```

---

## 八、本节练习

先不用写 Go 代码，可以找一个公开 HTTP 服务练习：

```bash
curl -i https://httpbin.org/get
curl -i "https://httpbin.org/get?name=alice"
curl -i -X POST https://httpbin.org/post -H "Content-Type: application/json" -d "{\"name\":\"alice\"}"
```

观察返回内容中的：

- URL。
- Header。
- JSON Body。
- 状态码。

---

## 九、本节检查点

请确认你能独立完成：

- 用浏览器访问 GET URL。
- 用 `curl -i` 查看状态码和 Header。
- 用 `curl -X POST -H -d` 发送 JSON。
- 用 `curl -L` 跟随重定向。
- 根据状态码判断请求大概失败在哪里。

