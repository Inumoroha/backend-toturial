# 03 网关能力：生产配置模式与边界判断

## 本节目标

把限流、CORS、缓存、上传限制这些零散配置，整理成生产中更清晰的配置模式。

## 一、按路径拆分策略

不要把所有 API 都放在一个 location 里。更清晰的方式是按风险和行为拆分：

```nginx
location /api/public/ {
    # 公开只读接口，可以考虑缓存
}

location /api/auth/ {
    # 登录、注册、验证码，应该更严格限流
}

location /api/upload {
    # 上传接口，单独设置 body size 和超时
}

location /api/admin/ {
    # 后台接口，可能需要 IP 限制或 Basic Auth
}

location /api/ {
    # 普通 API 默认策略
}
```

这样配置更长，但更容易排查和维护。

## 二、登录接口限流示例

```nginx
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=2r/s;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=20r/s;

server {
    listen 80;
    server_name api.local;

    limit_req_status 429;

    location /api/auth/ {
        limit_req zone=login_limit burst=5 nodelay;
        proxy_pass http://127.0.0.1:8080;
    }

    location /api/ {
        limit_req zone=api_limit burst=40 nodelay;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

登录、短信、验证码接口通常比普通 API 更需要保护。

## 三、CORS 更稳妥写法

如果只允许固定域名：

```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com" $http_origin;
    "https://admin.example.com" $http_origin;
}

server {
    location /api/ {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

        if ($request_method = OPTIONS) {
            return 204;
        }

        proxy_pass http://127.0.0.1:8080;
    }
}
```

`map` 要放在 `http` 上下文。这样比直接写 `*` 更安全。

## 四、缓存必须考虑用户身份

错误示例：

```nginx
location /api/user/profile {
    proxy_cache api_cache;
    proxy_pass http://127.0.0.1:8080;
}
```

这可能把 A 用户的个人信息缓存后返回给 B 用户。

谨慎原则：

- 默认不要缓存带 Authorization 的请求。
- 默认不要缓存 Cookie 相关请求。
- 默认不要缓存用户个人数据。

示例：

```nginx
proxy_no_cache $http_authorization;
proxy_cache_bypass $http_authorization;
```

## 五、缓存 key 要明确

```nginx
proxy_cache_key "$scheme$request_method$host$request_uri";
```

如果响应会受语言、租户、设备影响，缓存 key 也要包含对应维度。例如：

```nginx
proxy_cache_key "$scheme$request_method$host$request_uri$http_accept_language";
```

## 六、上传接口的特殊配置

上传接口往往需要：

```nginx
location /api/upload {
    client_max_body_size 100m;
    proxy_request_buffering on;
    proxy_read_timeout 120s;
    proxy_pass http://127.0.0.1:8080;
}
```

注意：

- `client_max_body_size` 控制最大请求体。
- 大文件上传可能需要更长超时。
- Go 服务也要限制文件大小，不能只依赖 Nginx。

## 七、网关配置审查清单

上线前检查：

- 登录和验证码接口是否限流。
- 上传接口是否有大小限制。
- 管理后台是否有访问控制。
- CORS 是否只允许可信域名。
- 缓存是否避开了用户私有数据。
- Nginx 与 Go 是否都做了必要校验。
- 限流状态码是否明确，例如 429。

## 八、本节练习

1. 把 `/api/auth/` 和 `/api/` 配置成不同限流。
2. 使用 `map` 配置允许域名列表。
3. 给带 Authorization 的请求禁用代理缓存。
4. 给上传接口设置更大 body size 和超时。
5. 写一份你自己项目的网关策略表。

## 九、你应该掌握

学完本节，你应该能按接口类型设计 Nginx 网关策略，而不是把所有 API 都套一个模板。

