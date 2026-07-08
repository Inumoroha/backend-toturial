# 01 深入理解：location、rewrite 与变量

## 本节目标

进一步理解 Nginx 如何匹配请求路径、重写 URI，以及使用变量。你不需要一开始就写很复杂的规则，但必须能看懂项目里的配置。

## 一、location 匹配复习

常见类型：

```nginx
location = /health {}
location /api/ {}
location ^~ /assets/ {}
location ~ \.jpg$ {}
location / {}
```

建议优先使用：

- 精确匹配处理健康检查。
- 前缀匹配处理 API 和静态资源。
- 兜底 `/` 处理默认路径。

正则 location 功能强，但复杂度也高。

## 二、location 示例

```nginx
server {
    listen 80;
    server_name match.local;

    location = /health {
        return 200 "exact health\n";
    }

    location ^~ /assets/ {
        return 200 "assets\n";
    }

    location /api/ {
        return 200 "api\n";
    }

    location ~ \.txt$ {
        return 200 "txt file\n";
    }

    location / {
        return 200 "fallback\n";
    }
}
```

练习访问：

```bash
curl http://match.local/health
curl http://match.local/assets/a.txt
curl http://match.local/api/users
curl http://match.local/readme.txt
curl http://match.local/anything
```

## 三、rewrite 基础

`rewrite` 可以修改请求 URI。

```nginx
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 last;
}
```

常见 flag：

- `last`：重新走 location 匹配。
- `break`：停止当前 rewrite，继续在当前 location 处理。
- `redirect`：返回 302。
- `permanent`：返回 301。

## 四、rewrite 与 return 的选择

如果只是跳转，优先使用 `return`：

```nginx
return 301 https://$host$request_uri;
```

它更清晰，也更容易维护。

复杂路径改写才考虑 `rewrite`。

## 五、常用变量

```nginx
$host
$remote_addr
$request_uri
$uri
$args
$scheme
$request_method
$http_user_agent
$http_authorization
$upstream_addr
$upstream_response_time
```

常见用途：

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
return 301 https://$host$request_uri;
```

## 六、$uri 与 $request_uri

- `$request_uri`：原始请求 URI，包含查询参数。
- `$uri`：Nginx 处理后的规范化 URI，不包含查询参数。

示例：

```text
请求: /api/users?id=1
$request_uri: /api/users?id=1
$uri: /api/users
```

## 七、本节练习

1. 写多个 location，验证匹配结果。
2. 用 `return` 做 HTTP 到 HTTPS 跳转。
3. 用 `rewrite` 把 `/old/a` 改写到 `/new/a`。
4. 在日志中记录 `$uri` 和 `$request_uri`。
5. 解释一次请求最终匹配到了哪个 location。

## 八、你应该掌握

学完本节，你应该能看懂项目中常见的路径匹配、重定向和 URI 改写配置。

