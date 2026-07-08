# 2. CORS 与 OPTIONS 预检请求

本节目标：理解浏览器跨域请求为什么会失败，掌握用 Nginx 添加 CORS 响应头和处理 OPTIONS 预检请求的方法。

---

## 一、什么是 CORS

CORS 是浏览器的跨域访问控制机制。

例如前端页面来自：

```text
https://app.example.com
```

请求 API：

```text
https://api.example.com
```

浏览器认为这是跨域请求。服务端必须明确返回允许跨域的响应头，否则浏览器会拦截响应。

注意：CORS 是浏览器限制。`curl` 不受 CORS 影响。

---

## 二、最小 CORS 配置

```nginx
location /api/ {
    add_header Access-Control-Allow-Origin "https://app.example.com" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

    if ($request_method = OPTIONS) {
        return 204;
    }

    proxy_pass http://127.0.0.1:8080;
}
```

解释：

- `Access-Control-Allow-Origin`：允许哪个前端域名访问。
- `Access-Control-Allow-Methods`：允许哪些 HTTP 方法。
- `Access-Control-Allow-Headers`：允许前端携带哪些请求头。
- `OPTIONS`：浏览器预检请求。

---

## 三、为什么有 OPTIONS 预检

当前端发送一些“非简单请求”时，浏览器会先发 OPTIONS：

```http
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

如果服务端允许，浏览器才会继续发送真正的 POST 请求。

---

## 四、不要随便使用星号

不推荐在有认证的接口中直接：

```nginx
add_header Access-Control-Allow-Origin "*" always;
```

原因：

- 涉及 Cookie、Authorization 时应限制可信来源。
- `*` 会扩大攻击面。
- 生产环境通常需要白名单。

---

## 五、使用 map 做白名单

`map` 写在 `http` 上下文：

```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com" $http_origin;
    "https://admin.example.com" $http_origin;
}
```

location 中：

```nginx
location /api/ {
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

    if ($request_method = OPTIONS) {
        return 204;
    }

    proxy_pass http://127.0.0.1:8080;
}
```

这样只有白名单中的 Origin 会被原样返回。

---

## 六、用 curl 模拟预检

```bash
curl -i -X OPTIONS http://api.local/api/ping \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type, Authorization"
```

期望：

```text
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
```

---

## 七、CORS 放 Nginx 还是 Go

适合放 Nginx：

- 所有 API 使用统一跨域策略。
- 前端域名固定。
- 想在入口层统一处理 OPTIONS。

适合放 Go：

- 不同接口跨域策略不同。
- 跨域规则依赖租户、用户或业务配置。
- 需要更复杂的 Origin 校验逻辑。

---

## 八、本节练习

1. 给 `/api/` 添加 CORS 响应头。
2. 用 curl 模拟 OPTIONS 预检。
3. 把允许域名从 `*` 改成固定域名。
4. 使用 `map` 支持两个前端域名。
5. 思考你的项目 CORS 应该放 Nginx 还是 Go。

---

## 九、本节复盘

请确认你能回答：

1. CORS 是浏览器限制还是服务端限制？
2. OPTIONS 预检请求什么时候出现？
3. 为什么生产中不建议随便使用 `*`？
4. `map $http_origin $cors_origin` 有什么作用？
5. CORS 放 Nginx 和放 Go 的区别是什么？

