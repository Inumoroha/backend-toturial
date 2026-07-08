# 02 网关能力：CORS、代理缓存与上传限制

## 本节目标

学习几个常见网关配置：

- CORS 跨域响应头。
- 代理缓存。
- 上传大小限制。

## 一、CORS 是什么

CORS 是浏览器的跨域访问控制。它不是 Nginx 独有能力，但 Nginx 可以统一添加跨域响应头。

简单示例：

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

注意：CORS 也可以放在 Go 服务中。如果跨域规则和业务强相关，建议放 Go。

## 二、不要随意使用 `*`

```nginx
add_header Access-Control-Allow-Origin "*" always;
```

这对公开 API 可能可以，但如果涉及 Cookie、Authorization、敏感数据，应明确允许的域名。

## 三、上传大小限制

全局限制：

```nginx
server {
    client_max_body_size 10m;
}
```

单独给上传接口放大：

```nginx
location /api/upload {
    client_max_body_size 100m;
    proxy_pass http://127.0.0.1:8080;
}
```

超过限制会返回 `413`。

## 四、代理缓存基础

在 `http` 块定义缓存路径：

```nginx
proxy_cache_path /var/cache/nginx/api_cache
                 levels=1:2
                 keys_zone=api_cache:10m
                 max_size=1g
                 inactive=10m
                 use_temp_path=off;
```

在 location 中使用：

```nginx
location /api/public/ {
    proxy_cache api_cache;
    proxy_cache_valid 200 1m;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    add_header X-Cache-Status $upstream_cache_status always;

    proxy_pass http://127.0.0.1:8080;
}
```

验证：

```bash
curl -I http://api.local/api/public/news
```

观察：

```text
X-Cache-Status: MISS
X-Cache-Status: HIT
```

## 五、哪些接口适合缓存

适合：

- 公共配置。
- 新闻列表。
- 商品详情。
- 不频繁变化的只读接口。

不适合：

- 用户个人数据。
- 订单数据。
- 权限相关响应。
- 强实时接口。
- 带副作用的 POST、PUT、DELETE。

## 六、完整示例

```nginx
proxy_cache_path /var/cache/nginx/api_cache
                 levels=1:2
                 keys_zone=api_cache:10m
                 max_size=1g
                 inactive=10m
                 use_temp_path=off;

server {
    listen 80;
    server_name api.local;

    client_max_body_size 10m;

    location /api/upload {
        client_max_body_size 100m;
        proxy_pass http://127.0.0.1:8080;
    }

    location /api/public/ {
        proxy_cache api_cache;
        proxy_cache_valid 200 1m;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status always;

        proxy_pass http://127.0.0.1:8080;
    }

    location /api/ {
        add_header Access-Control-Allow-Origin "https://app.example.com" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

        if ($request_method = OPTIONS) {
            return 204;
        }

        proxy_pass http://127.0.0.1:8080;
    }
}
```

## 七、本节练习

1. 给 `/api/` 添加 CORS 响应头。
2. 单独给 `/api/upload` 设置 `client_max_body_size 100m`。
3. 给 `/api/public/` 配置 1 分钟缓存。
4. 使用 `X-Cache-Status` 验证 HIT 与 MISS。
5. 判断你的接口哪些可以缓存，哪些绝不能缓存。

## 八、你应该掌握

学完本节，你应该能：

- 处理常见跨域问题。
- 限制或放大上传大小。
- 给只读接口添加代理缓存。
- 理解缓存带来的数据一致性风险。

