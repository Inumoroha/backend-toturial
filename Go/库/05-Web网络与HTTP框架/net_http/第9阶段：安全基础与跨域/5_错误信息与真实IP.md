# 5. 错误信息与真实 IP

本节目标：理解错误响应不能泄露内部信息，以及反向代理后如何看待客户端 IP。

---

## 一、不要暴露内部错误

不推荐：

```json
{
  "error": "sql: no rows in result set"
}
```

更推荐：

```json
{
  "error": {
    "code": "not_found",
    "message": "resource not found"
  }
}
```

内部错误写日志，对外响应保持稳定和克制。

---

## 二、panic 信息不要返回给客户端

recovery 中间件里不要这样：

```go
writeError(w, 500, "panic", fmt.Sprint(rec))
```

应该：

```go
log.Printf("panic: %v", rec)
writeError(w, 500, "internal_error", "internal server error")
```

---

## 三、RemoteAddr 不一定是真实用户 IP

```go
r.RemoteAddr
```

它表示直接连接到 Go 服务的对端地址。

如果你的服务在 Nginx、负载均衡、网关后面，`RemoteAddr` 可能是代理地址，不是真实用户地址。

---

## 四、X-Forwarded-For 要谨慎

代理常用 Header：

```text
X-Forwarded-For: client, proxy1, proxy2
X-Real-IP: client
```

但客户端也可以伪造这些 Header。

只有当请求来自你信任的反向代理时，才应该信任这些 Header。

---

## 五、安全检查清单

一个基础 Go HTTP 服务至少检查：

- 是否配置 Server 超时。
- 是否限制请求体大小。
- JSON 是否统一错误响应。
- 是否避免暴露内部错误。
- Cookie 是否设置 `HttpOnly`、`Secure`、`SameSite`。
- CORS 是否使用白名单。
- 鉴权失败是否返回正确状态码。
- 日志中是否避免打印敏感信息。
- 是否理解反向代理后的真实 IP 问题。

---

## 六、本节检查点

请确认你能回答：

- 为什么内部错误不能原样返回？
- panic 信息应该写到哪里？
- `RemoteAddr` 在代理后有什么问题？
- 为什么不能无条件信任 `X-Forwarded-For`？

