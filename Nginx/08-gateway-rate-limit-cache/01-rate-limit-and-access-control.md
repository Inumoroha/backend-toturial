# 01 网关能力：限流与访问控制

## 本节目标

学习 Nginx 作为轻量网关时的基础能力：

- 请求限流。
- 并发连接限制。
- IP 黑白名单。
- Basic Auth。

这些能力适合做粗粒度保护，复杂业务规则仍然应该在 Go 服务中实现。

## 一、请求限流 limit_req

在 `http` 块中定义限流区域：

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

含义：

- `$binary_remote_addr`：按客户端 IP 限流。
- `zone=api_limit:10m`：共享内存区域名称和大小。
- `rate=10r/s`：每秒 10 个请求。

在 location 中使用：

```nginx
location /api/ {
    limit_req zone=api_limit burst=20 nodelay;

    proxy_pass http://127.0.0.1:8080;
}
```

含义：

- `burst=20`：允许短时间突发排队或通过。
- `nodelay`：突发请求不延迟，符合条件就立即处理，超过则拒绝。

## 二、限流状态码

可以设置被限流时返回的状态码：

```nginx
limit_req_status 429;
```

`429 Too Many Requests` 比 `503` 更适合表达限流。

## 三、连接限制 limit_conn

在 `http` 块定义：

```nginx
limit_conn_zone $binary_remote_addr zone=addr_conn:10m;
```

在 location 中使用：

```nginx
location /api/ {
    limit_conn addr_conn 20;
    proxy_pass http://127.0.0.1:8080;
}
```

表示单个 IP 最多 20 个并发连接。

## 四、IP 黑白名单

只允许内网访问：

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass http://127.0.0.1:8080;
}
```

拒绝某个 IP：

```nginx
deny 203.0.113.10;
allow all;
```

## 五、Basic Auth

给测试环境或后台入口加密码：

```nginx
location /admin/ {
    auth_basic "Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://127.0.0.1:8080;
}
```

创建密码文件：

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

## 六、Nginx 限流与 Go 限流的边界

适合放 Nginx：

- 单 IP 粗粒度限流。
- 防止突发流量打爆后端。
- 给登录、验证码、上传接口加基础保护。

适合放 Go：

- 按用户 ID 限流。
- 按租户限流。
- 按 API Key 限流。
- 动态限流策略。
- 与业务状态相关的限流。

## 七、完整示例

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=addr_conn:10m;

server {
    listen 80;
    server_name api.local;

    limit_req_status 429;

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        limit_conn addr_conn 20;

        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 八、本节练习

1. 配置单 IP 每秒 10 请求。
2. 使用压测工具或循环 curl 触发限流。
3. 把限流状态码改为 429。
4. 给 `/admin/` 加 Basic Auth。
5. 思考登录接口应该在 Nginx 和 Go 各做什么保护。

## 九、你应该掌握

学完本节，你应该能为 Go API 添加基础流量保护，并能说明 Nginx 限流的边界。

