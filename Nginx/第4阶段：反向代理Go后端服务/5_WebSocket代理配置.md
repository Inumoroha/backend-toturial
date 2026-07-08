# 5. WebSocket 代理配置

本节目标：理解 WebSocket 为什么需要特殊代理头，并能写出 Nginx WebSocket 代理配置。

---

## 一、WebSocket 和普通 HTTP 的区别

WebSocket 开始时也是 HTTP 请求，但之后会升级协议：

```text
HTTP -> WebSocket
```

请求头里会有：

```text
Upgrade: websocket
Connection: Upgrade
```

Nginx 需要把这些头正确转发给 Go。

---

## 二、map 配置

在 `http` 上下文：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

含义：

- 如果客户端有 Upgrade，就设置 Connection 为 upgrade。
- 如果没有，就 close。

---

## 三、WebSocket location

```nginx
server {
    listen 80;
    server_name api.local;

    location /ws/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }
}
```

关键点：

- `proxy_http_version 1.1`
- `Upgrade`
- `Connection`
- 更长的 `proxy_read_timeout`

---

## 四、为什么要更长超时

WebSocket 是长连接。如果 `proxy_read_timeout` 太短，连接可能空闲一会儿就被 Nginx 断开。

可以设置：

```nginx
proxy_read_timeout 3600s;
```

但应用层也应该有心跳机制。

---

## 五、常见问题

### 1. 握手失败

检查是否传了：

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

### 2. 连接一会儿就断

检查：

- `proxy_read_timeout`。
- Go WebSocket 心跳。
- 客户端心跳。
- 中间代理或负载均衡超时。

### 3. map 写错上下文

`map` 要写在 `http` 上下文，不能写在 `server` 或 `location` 里。

---

## 六、本节练习

1. 在 `http` 中添加 `map`。
2. 配置 `/ws/` location。
3. 检查 `nginx -t`。
4. 如果你的 Go 项目有 WebSocket，验证握手。
5. 故意去掉 Upgrade 头，观察失败。

---

## 七、本节复盘

请确认你能回答：

1. WebSocket 为什么需要 Upgrade？
2. `map $http_upgrade $connection_upgrade` 有什么作用？
3. WebSocket 为什么需要 HTTP/1.1？
4. 长连接为什么要关注 `proxy_read_timeout`？
5. `map` 应该写在哪个上下文？

