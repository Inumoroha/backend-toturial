# 3. location 匹配规则详解

本节目标：理解 Nginx 如何根据 URI 选择 `location`。这是学习静态资源、反向代理、限流、缓存和排障的基础。

很多 Nginx 问题表面上是 404、502、配置不生效，本质上是请求没有进入你以为的那个 `location`。

---

## 一、URI 是什么

请求：

```text
GET /api/users?id=1 HTTP/1.1
Host: api.local
```

这里：

```text
/api/users?id=1 是 request_uri。
/api/users 是 uri。
id=1 是 args。
```

Nginx 的 `location` 主要根据 URI 路径部分匹配，也就是 `/api/users`。

---

## 二、准备实验配置

创建：

```nginx
server {
    listen 80;
    server_name match.local;

    location = /health {
        return 200 "exact health\n";
    }

    location ^~ /assets/ {
        return 200 "assets prefix\n";
    }

    location /api/ {
        return 200 "api prefix\n";
    }

    location ~ \.txt$ {
        return 200 "txt regex\n";
    }

    location / {
        return 200 "fallback\n";
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

测试时可以用：

```bash
curl -H "Host: match.local" http://127.0.0.1/health
```

---

## 三、精确匹配 =

```nginx
location = /health {
    return 200 "exact health\n";
}
```

只匹配：

```text
/health
```

不匹配：

```text
/health/
/health/check
```

测试：

```bash
curl -H "Host: match.local" http://127.0.0.1/health
curl -H "Host: match.local" http://127.0.0.1/health/
```

适合用于健康检查：

```nginx
location = /health {
    return 200 "ok\n";
}
```

---

## 四、普通前缀匹配

```nginx
location /api/ {
    return 200 "api prefix\n";
}
```

匹配：

```text
/api/
/api/users
/api/orders/1
```

不匹配：

```text
/apix
/api
```

注意 `/api` 和 `/api/` 是不同路径。生产中最好统一风格。

---

## 五、^~ 前缀匹配

```nginx
location ^~ /assets/ {
    return 200 "assets prefix\n";
}
```

含义：如果这个前缀匹配到了，就不再继续检查正则 location。

适合静态资源：

```nginx
location ^~ /assets/ {
    root /var/www/app;
}
```

原因是静态资源路径通常很明确，不希望被后面的正则意外抢走。

---

## 六、正则匹配

```nginx
location ~ \.txt$ {
    return 200 "txt regex\n";
}
```

`~` 表示区分大小写正则。

测试：

```bash
curl -H "Host: match.local" http://127.0.0.1/readme.txt
```

正则功能强，但也更容易让配置难以理解。初学阶段建议少用。

---

## 七、兜底 location /

```nginx
location / {
    return 200 "fallback\n";
}
```

几乎所有路径都会匹配 `/`，所以它常作为兜底。

常见用途：

- 返回前端 `index.html`。
- 代理到默认 Go 服务。
- 返回 404。

---

## 八、简化匹配顺序

初学阶段可以这样记：

1. 先找 `=` 精确匹配。
2. 再找最长前缀匹配。
3. 如果命中 `^~`，停止检查正则。
4. 否则可能继续检查正则。
5. 最后用 `/` 兜底。

不要一开始就写复杂组合。真实项目里，清晰比炫技更重要。

---

## 九、常见项目结构

前后端分离项目常见配置：

```nginx
server {
    listen 80;
    server_name app.local;

    location = /health {
        return 200 "ok\n";
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }

    location ^~ /assets/ {
        root /var/www/app;
        expires 30d;
    }

    location / {
        root /var/www/app;
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 十、本节练习

1. 配置 `match.local`。
2. 测试 `/health` 和 `/health/`。
3. 测试 `/api/users`。
4. 测试 `/assets/app.txt`，观察是否进入 `^~ /assets/`。
5. 测试 `/readme.txt`。
6. 删除 `^~` 再测试一次，观察变化。

---

## 十一、本节复盘

请确认你能回答：

1. `location = /health` 和 `location /health` 有什么区别？
2. `^~` 的作用是什么？
3. 为什么 `/api` 不一定匹配 `/api/`？
4. 为什么正则 location 要谨慎使用？
5. 如何判断请求进入了哪个 location？

