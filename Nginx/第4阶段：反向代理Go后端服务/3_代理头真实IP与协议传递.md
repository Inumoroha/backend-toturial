# 3. 代理头、真实 IP 与协议传递

本节目标：理解 Nginx 代理 Go 服务时为什么要设置请求头，以及 Go 服务如何正确读取客户端真实信息。

---

## 一、为什么需要代理头

当请求链路是：

```text
Client -> Nginx -> Go
```

Go 服务直接看到的客户端通常是 Nginx，而不是用户浏览器。

如果你在 Go 中读取：

```go
r.RemoteAddr
```

很可能看到：

```text
127.0.0.1:xxxxx
```

这只是 Nginx 到 Go 的连接地址。

---

## 二、推荐基础代理头

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

解释：

- `Host`：保留用户访问的域名。
- `X-Real-IP`：Nginx 看到的客户端 IP。
- `X-Forwarded-For`：代理链路 IP 列表。
- `X-Forwarded-Proto`：用户访问 Nginx 时使用的协议。

---

## 三、Go 中观察这些头

访问：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/headers
```

如果前一节的 Go 服务还在运行，应该能看到：

```json
{
  "remote_addr": "127.0.0.1:xxxxx",
  "host": "api.local",
  "x_real_ip": "127.0.0.1",
  "x_forwarded_for": "127.0.0.1",
  "x_forwarded_proto": "http"
}
```

这说明 Nginx 已经把代理信息传给 Go。

---

## 四、HTTPS 场景下的协议

如果用户访问：

```text
https://api.example.com
```

Nginx 到 Go 仍可能是：

```text
http://127.0.0.1:8080
```

这时 Go 服务如果要知道用户原始协议，就要看：

```text
X-Forwarded-Proto: https
```

Nginx HTTPS server 中通常写：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

或者：

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

---

## 五、X-Forwarded-For 不一定可信

客户端可以伪造：

```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://api.local/api/headers
```

如果 Nginx 使用：

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Go 可能看到：

```text
1.2.3.4, 127.0.0.1
```

所以不要在安全敏感逻辑里无条件信任请求头。

真实 IP 的可信性来自网络边界：

- Go 服务不直接暴露公网。
- 只有可信 Nginx 能访问 Go。
- 多层代理时配置 `set_real_ip_from`。

---

## 六、Go 服务应该怎么做

简单内网场景：

```go
func ClientIP(r *http.Request) string {
	if ip := r.Header.Get("X-Real-IP"); ip != "" {
		return ip
	}
	return r.RemoteAddr
}
```

复杂生产场景要明确可信代理列表，不要随便相信任何客户端传来的头。

---

## 七、常见用途

Go 服务会用这些头做：

- 访问日志。
- 审计。
- 限流。
- 生成回调 URL。
- OAuth 跳转地址。
- 判断是否 HTTPS。

如果代理头缺失，可能出现：

- 日志里全是 `127.0.0.1`。
- 生成的 URL 是 http 而不是 https。
- 限流按 Nginx IP 限，所有用户互相影响。

---

## 八、本节练习

1. 配置四个基础代理头。
2. 访问 `/api/headers`。
3. 伪造 `X-Forwarded-For` 请求头。
4. 观察 Go 收到的变化。
5. 思考你的 Go 服务是否能被绕过 Nginx 直接访问。

---

## 九、本节复盘

请确认你能回答：

1. Go 的 `RemoteAddr` 为什么不一定是真实客户端 IP？
2. `X-Real-IP` 和 `X-Forwarded-For` 有什么区别？
3. 为什么 `X-Forwarded-For` 可以被伪造？
4. HTTPS 终止在 Nginx 时，Go 怎么知道原始协议？
5. 为什么 Go 服务不应该直接暴露公网？

