# 4. return、rewrite 与常用变量

本节目标：掌握 Nginx 中最常见的直接返回、重定向、路径改写和变量使用方式。

这一节不追求复杂 rewrite，而是建立清晰判断：

```text
简单返回或跳转，用 return。
复杂 URI 改写，再考虑 rewrite。
```

---

## 一、return 直接返回

最简单：

```nginx
location /ping {
    return 200 "pong\n";
}
```

测试：

```bash
curl http://example.local/ping
```

返回：

```text
pong
```

健康检查经常这样写：

```nginx
location = /health {
    return 200 "ok\n";
}
```

---

## 二、return 做重定向

HTTP 跳转 HTTPS：

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
```

含义：

- `$host`：请求 Host。
- `$request_uri`：原始 URI，包含查询参数。

请求：

```text
http://example.com/api/users?id=1
```

跳转到：

```text
https://example.com/api/users?id=1
```

---

## 三、常用变量

| 变量 | 含义 |
| --- | --- |
| `$host` | 请求 Host |
| `$remote_addr` | 连接到 Nginx 的客户端 IP |
| `$uri` | 规范化后的 URI，不含查询参数 |
| `$request_uri` | 原始 URI，包含查询参数 |
| `$args` | 查询参数 |
| `$scheme` | 请求协议，http 或 https |
| `$request_method` | 请求方法 |
| `$http_user_agent` | User-Agent 请求头 |
| `$upstream_addr` | 实际 upstream 地址 |
| `$upstream_response_time` | upstream 响应耗时 |

---

## 四、$uri 与 $request_uri 的区别

请求：

```text
/api/users?id=1
```

通常：

```text
$uri = /api/users
$request_uri = /api/users?id=1
$args = id=1
```

做 HTTPS 跳转时，一般用：

```nginx
return 301 https://$host$request_uri;
```

因为你希望保留查询参数。

---

## 五、rewrite 基础

路径改写：

```nginx
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 last;
}
```

请求：

```text
/old/a
```

会改写成：

```text
/new/a
```

`last` 表示用新 URI 重新进行 location 匹配。

---

## 六、rewrite 常见 flag

| flag | 含义 |
| --- | --- |
| `last` | 停止当前 rewrite，重新匹配 location |
| `break` | 停止 rewrite，在当前 location 继续处理 |
| `redirect` | 返回 302 临时重定向 |
| `permanent` | 返回 301 永久重定向 |

学习阶段最常用的是 `return 301`，rewrite 可以后面再深入。

---

## 七、调试变量

可以临时返回变量：

```nginx
location /debug {
    return 200 "host=$host\nuri=$uri\nrequest_uri=$request_uri\nargs=$args\n";
}
```

测试：

```bash
curl "http://example.local/debug?id=1"
```

这是一种很适合学习阶段的调试方式。

---

## 八、本节练习

1. 写 `/ping` 返回 `pong`。
2. 写 HTTP 到 HTTPS 的 301 跳转。
3. 写 `/debug` 返回 `$host`、`$uri`、`$request_uri`。
4. 写 rewrite，把 `/old/a` 改到 `/new/a`。
5. 对比 `return 301` 和 `rewrite permanent`。

---

## 九、本节复盘

请确认你能回答：

1. 为什么简单跳转推荐用 `return`？
2. `$uri` 和 `$request_uri` 有什么区别？
3. HTTP 跳 HTTPS 为什么要保留 `$request_uri`？
4. `rewrite ... last` 表示什么？
5. 如何临时输出变量帮助调试？

