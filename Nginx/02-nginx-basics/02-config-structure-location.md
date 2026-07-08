# 02 Nginx 基础：配置结构与 location 入门

## 本节目标

理解 Nginx 配置文件的层级结构，并掌握最常见的 `location` 用法。

## 一、配置结构

典型 Nginx 配置长这样：

```nginx
user www-data;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;

    server {
        listen 80;
        server_name example.local;

        location / {
            return 200 "hello\n";
        }
    }
}
```

主要层级：

- `main`：最外层，配置 worker、用户等全局信息。
- `events`：连接处理相关配置。
- `http`：HTTP 服务相关配置。
- `server`：一个虚拟主机。
- `location`：一个 URI 匹配规则。

日常开发最常改的是 `server` 和 `location`。

## 二、location 的基本作用

`location` 用来根据请求路径选择处理方式。

```nginx
server {
    listen 80;
    server_name demo.local;

    location / {
        return 200 "home\n";
    }

    location /api/ {
        return 200 "api\n";
    }

    location /admin/ {
        return 200 "admin\n";
    }
}
```

测试：

```bash
curl http://demo.local/
curl http://demo.local/api/users
curl http://demo.local/admin/
```

## 三、常见 location 类型

### 普通前缀匹配

```nginx
location /api/ {
    return 200 "api\n";
}
```

只要路径以 `/api/` 开头，就能匹配。

### 精确匹配

```nginx
location = /health {
    return 200 "ok\n";
}
```

只匹配 `/health`，不匹配 `/health/` 或 `/health/a`。

### 正则匹配

```nginx
location ~ \.php$ {
    return 403;
}
```

`~` 表示区分大小写正则匹配。学习初期少用正则，能用前缀就用前缀。

### 优先前缀匹配

```nginx
location ^~ /assets/ {
    return 200 "assets\n";
}
```

匹配到后，不再继续检查正则 location。

## 四、推荐先记住的匹配顺序

简化理解：

1. 先看有没有 `=` 精确匹配。
2. 再找最长的前缀匹配。
3. 如果有正则，可能继续检查正则。
4. 最后使用普通 `/` 兜底。

学习阶段建议你少写复杂 location，先保持清晰。

## 五、一个常见 API 配置雏形

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

    location / {
        root /var/www/app;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

这个配置表达：

- `/health` 由 Nginx 直接返回。
- `/api/` 转发给 Go 后端。
- 其他路径返回前端静态页面。

这是前后端分离项目里非常常见的结构。

## 六、本节练习

1. 配置 `/`、`/api/`、`/admin/` 三个 location。
2. 添加 `= /health` 精确匹配。
3. 用 `curl` 测试每个路径返回是否符合预期。
4. 故意写两个类似 location，观察哪个生效。
5. 用 `nginx -T` 查看最终加载的配置。

## 七、你应该掌握

学完本节，你应该能看懂：

- Nginx 配置的层级。
- `server` 和 `location` 的职责。
- 请求路径如何匹配到不同处理逻辑。
- 为什么 API、静态页面、健康检查经常写在不同 location 中。

