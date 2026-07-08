# 2. proxy_pass 基础与路径规则

本节目标：掌握 Nginx 反向代理 Go 服务的基础配置，并重点理解 `proxy_pass` 后面带不带 `/` 的区别。

这是 Nginx 代理 Go 服务时最容易踩坑的一节。

---

## 一、最小反向代理配置

假设 Go 服务已经运行：

```bash
curl http://127.0.0.1:8080/api/ping
```

返回正常后，配置 Nginx：

```nginx
server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

测试：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/ping
```

如果返回 Go 的 JSON，说明代理成功。

---

## 二、请求链路

```text
curl http://api.local/api/ping
  |
  v
Nginx :80
  |
  | proxy_pass
  v
Go :8080 /api/ping
```

Nginx 对客户端来说是服务端，对 Go 来说又是客户端。

---

## 三、proxy_pass 不带 URI

配置：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

请求：

```text
/api/ping
```

转发给 Go：

```text
/api/ping
```

也就是原始 URI 基本保留。

这种写法最适合初学阶段：

```text
Nginx 的 /api/ 对应 Go 的 /api/
```

---

## 四、proxy_pass 带 /

配置：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

请求：

```text
/api/ping
```

转发给 Go：

```text
/ping
```

因为 `location /api/` 匹配到的前缀被替换成了 `proxy_pass` 后面的 `/`。

如果 Go 只注册了 `/api/ping`，这时就会 404。

---

## 五、路径规则对比表

| location | proxy_pass | 客户端请求 | Go 收到 |
| --- | --- | --- | --- |
| `/api/` | `http://127.0.0.1:8080` | `/api/ping` | `/api/ping` |
| `/api/` | `http://127.0.0.1:8080/` | `/api/ping` | `/ping` |
| `/backend/` | `http://127.0.0.1:8080/api/` | `/backend/ping` | `/api/ping` |

学习阶段建议优先使用第一种。

---

## 六、什么时候需要去掉前缀

如果你希望对外暴露：

```text
/api/ping
```

但 Go 服务内部路由是：

```text
/ping
```

可以写：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

这表示 Nginx 帮你去掉 `/api/` 前缀。

但生产中要写清楚，否则后期很容易忘记路径被改了。

---

## 七、验证路径是否正确

在 Go 服务中打印：

```go
log.Printf("path=%s raw_query=%s", r.URL.Path, r.URL.RawQuery)
```

或者临时增加接口返回：

```go
json.NewEncoder(w).Encode(map[string]string{
    "path": r.URL.Path,
})
```

然后通过 Nginx 请求，观察 Go 实际收到的路径。

---

## 八、常见问题

### 1. Nginx 访问 404，但 Go 直连正常

很可能是路径被 `proxy_pass` 改了。

排查：

```bash
curl http://127.0.0.1:8080/api/ping
curl -H "Host: api.local" http://127.0.0.1/api/ping
```

如果第一个成功，第二个 404，就检查 `proxy_pass` 末尾斜杠。

### 2. Nginx 返回 502

这通常不是路径问题，而是 Nginx 连不上 Go。

检查：

```bash
curl http://127.0.0.1:8080/api/ping
sudo tail -n 50 /var/log/nginx/error.log
```

---

## 九、本节练习

1. 配置 `location /api/`，`proxy_pass` 不带 `/`。
2. 请求 `/api/ping`，确认成功。
3. 把 `proxy_pass` 改成带 `/`。
4. 再次请求 `/api/ping`，观察 Go 是否还能匹配。
5. 修改 Go 路由或 Nginx 配置，让请求重新成功。

---

## 十、本节复盘

请确认你能回答：

1. `proxy_pass http://127.0.0.1:8080;` 会保留 URI 吗？
2. `proxy_pass http://127.0.0.1:8080/;` 会如何处理 `/api/` 前缀？
3. Nginx 访问 404 但 Go 直连成功，优先怀疑什么？
4. 为什么初学阶段建议 Nginx 和 Go 都使用 `/api/` 前缀？
5. 如何验证 Go 实际收到的路径？

