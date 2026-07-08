# 1. limit_req 请求限流

本节目标：使用 Nginx 对 API 做基础请求限流，并理解限流区域、速率、突发值和 429 状态码。

---

## 一、为什么要限流

限流可以保护 Go 服务，避免被突发流量打垮。

适合 Nginx 做的限流：

- 单 IP 粗粒度限流。
- 登录接口保护。
- 短信验证码接口保护。
- 上传接口保护。

不适合只靠 Nginx 做的限流：

- 按用户 ID 限流。
- 按租户限流。
- 按 API Key 限流。
- 复杂业务风控。

---

## 二、定义限流区域

在 `http` 上下文中：

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

解释：

- `$binary_remote_addr`：按客户端 IP 作为限流 key。
- `zone=api_limit:10m`：创建名为 `api_limit` 的共享内存区域。
- `rate=10r/s`：平均每秒 10 个请求。

---

## 三、在 location 中使用

```nginx
server {
    listen 80;
    server_name api.local;

    limit_req_status 429;

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

解释：

- `burst=20`：允许短时间突发。
- `nodelay`：突发请求不排队等待，超过就拒绝。
- `limit_req_status 429`：被限流时返回 429。

---

## 四、测试限流

```bash
for i in {1..100}; do curl -s -o /dev/null -w "%{http_code}\n" http://api.local/api/ping; done
```

你可能看到：

```text
200
200
429
429
```

具体比例和执行速度有关。

---

## 五、查看日志

error log 中可能出现限流信息：

```bash
sudo tail -f /var/log/nginx/error.log
```

access log 中会看到 429。

---

## 六、登录接口单独限流

```nginx
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=2r/s;

location /api/auth/ {
    limit_req zone=login_limit burst=5 nodelay;
    proxy_pass http://127.0.0.1:8080;
}
```

登录、验证码接口应该比普通 API 更严格。

---

## 七、本节练习

1. 给 `/api/` 配置 `10r/s`。
2. 设置限流状态码为 429。
3. 用循环 curl 触发限流。
4. 单独给 `/api/auth/` 配置更严格限流。
5. 思考哪些限流应该放 Go 服务中。

---

## 八、本节复盘

请确认你能回答：

1. `limit_req_zone` 为什么要写在 http 上下文？
2. `zone=api_limit:10m` 表示什么？
3. `rate=10r/s` 是什么意思？
4. `burst` 和 `nodelay` 分别控制什么？
5. 为什么业务级限流不能只靠 Nginx？

