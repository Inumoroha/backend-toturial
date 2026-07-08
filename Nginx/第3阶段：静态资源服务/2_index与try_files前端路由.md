# 2. index 与 try_files 前端路由

本节目标：理解 `index` 和 `try_files`，并掌握前端单页应用常见的 Nginx 配置。

---

## 一、index 是什么

当用户访问目录：

```text
/
/docs/
```

Nginx 需要知道返回哪个默认文件。

配置：

```nginx
location / {
    root /home/your-user/nginx-lab/static;
    index index.html;
}
```

访问：

```text
/
```

实际查找：

```text
/home/your-user/nginx-lab/static/index.html
```

---

## 二、try_files 是什么

`try_files` 表示按顺序尝试文件。

```nginx
location / {
    root /home/your-user/nginx-lab/static;
    try_files $uri $uri/ =404;
}
```

含义：

```text
先找 $uri 对应的文件。
再找 $uri/ 对应的目录。
都找不到就返回 404。
```

请求：

```text
/assets/style.css
```

尝试：

```text
/home/your-user/nginx-lab/static/assets/style.css
/home/your-user/nginx-lab/static/assets/style.css/
404
```

---

## 三、普通静态站点配置

```nginx
server {
    listen 80;
    server_name static.local;

    root /home/your-user/nginx-lab/static;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

适合：

- 文档站。
- 普通 HTML 页面。
- 文件真实存在才访问。

---

## 四、前端 SPA 为什么需要特殊配置

Vue、React、Svelte 等前端单页应用可能有路径：

```text
/users/1
/settings/profile
```

这些路径在磁盘上并不存在真实文件。

如果直接刷新 `/users/1`，Nginx 会找：

```text
/home/your-user/nginx-lab/static/users/1
```

找不到就 404。

但前端应用希望所有非静态资源路径都返回：

```text
index.html
```

让前端路由自己处理。

---

## 五、SPA 配置

```nginx
server {
    listen 80;
    server_name app.local;

    root /home/your-user/nginx-lab/static;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

含义：

```text
真实文件存在 -> 返回真实文件。
真实目录存在 -> 返回目录 index。
都不存在 -> 返回 /index.html。
```

---

## 六、API 与 SPA 共存

前后端分离项目常见：

```nginx
server {
    listen 80;
    server_name app.local;

    root /home/your-user/nginx-lab/static;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

注意顺序不是唯一关键，匹配规则才是关键：

- `/api/` 更具体，会进入 API 代理。
- `/` 兜底处理前端页面。

---

## 七、常见错误

### 1. API 被前端 index.html 吃掉

现象：

```text
curl /api/ping 返回 HTML
```

说明 `/api/` 没有正确匹配代理，落到了 `/` 的 `try_files`。

检查：

```bash
sudo nginx -T | grep -A 30 "location /api/"
```

### 2. 前端刷新 404

说明没有配置：

```nginx
try_files $uri $uri/ /index.html;
```

### 3. 静态资源也返回 index.html

说明资源路径真的不存在，或者构建产物路径配置错了。

用：

```bash
curl -I http://app.local/assets/app.js
```

检查 `Content-Type` 和状态码。

---

## 八、本节练习

1. 配置普通静态站点，找不到返回 404。
2. 改成 SPA 配置，找不到返回 index.html。
3. 增加 `/api/` 代理。
4. 故意删掉 `/api/` location，观察 API 返回 HTML。
5. 用 `curl -I` 检查静态资源响应头。

---

## 九、本节复盘

请确认你能回答：

1. `index` 的作用是什么？
2. `try_files $uri $uri/ =404` 怎么工作？
3. SPA 为什么要 fallback 到 `/index.html`？
4. API 和 SPA 共存时为什么要单独写 `/api/`？
5. API 返回 HTML 时通常是什么问题？

