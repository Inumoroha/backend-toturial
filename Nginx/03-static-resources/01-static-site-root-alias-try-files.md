# 01 静态资源服务：root、alias 与 try_files

## 本节目标

掌握 Nginx 托管静态页面和文件的核心配置。Go 后端工程师也经常需要部署前端构建产物、管理后台、上传文件访问路径或文档站点，所以这一部分很实用。

## 一、准备静态目录

建议创建：

```bash
mkdir -p ~/nginx-lab/static/assets
```

创建 `~/nginx-lab/static/index.html`：

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nginx Static Lab</title>
  <link rel="stylesheet" href="/assets/style.css">
</head>
<body>
  <h1>Nginx Static Lab</h1>
  <p>如果你看到这个页面，说明静态资源配置成功。</p>
</body>
</html>
```

创建 `~/nginx-lab/static/assets/style.css`：

```css
body {
  margin: 40px;
  font-family: Arial, sans-serif;
}
```

## 二、使用 root

配置：

```nginx
server {
    listen 80;
    server_name static.local;

    location / {
        root /home/your-user/nginx-lab/static;
        index index.html;
    }
}
```

访问：

```bash
curl http://static.local/
curl http://static.local/assets/style.css
```

`root` 的路径拼接规则是：

```text
文件路径 = root 路径 + 请求 URI
```

例如：

```text
URI: /assets/style.css
root: /home/your-user/nginx-lab/static
最终文件: /home/your-user/nginx-lab/static/assets/style.css
```

## 三、使用 alias

`alias` 常用于把某个 URL 前缀映射到另一个文件目录。

```nginx
server {
    listen 80;
    server_name static.local;

    location /downloads/ {
        alias /home/your-user/nginx-lab/static/assets/;
    }
}
```

访问：

```bash
curl http://static.local/downloads/style.css
```

`alias` 的路径拼接规则是：

```text
文件路径 = alias 路径 + 去掉 location 前缀后的 URI
```

例如：

```text
URI: /downloads/style.css
location: /downloads/
alias: /home/your-user/nginx-lab/static/assets/
最终文件: /home/your-user/nginx-lab/static/assets/style.css
```

## 四、root 与 alias 的常见坑

### root 示例

```nginx
location /assets/ {
    root /home/your-user/nginx-lab/static;
}
```

请求 `/assets/style.css`，查找：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

### alias 示例

```nginx
location /assets/ {
    alias /home/your-user/nginx-lab/static/assets/;
}
```

请求 `/assets/style.css`，查找：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

二者结果可能一样，但含义不同。学习阶段建议：

- URL 路径和磁盘路径结构一致时，用 `root`。
- URL 路径只是一个映射别名时，用 `alias`。

## 五、使用 try_files

`try_files` 用来按顺序尝试文件是否存在。

```nginx
server {
    listen 80;
    server_name static.local;

    location / {
        root /home/your-user/nginx-lab/static;
        index index.html;
        try_files $uri $uri/ =404;
    }
}
```

含义：

1. 先找请求路径对应的文件。
2. 再找请求路径对应的目录。
3. 都找不到就返回 404。

## 六、前端单页应用配置

Vue、React 等 SPA 前端常需要这样配置：

```nginx
server {
    listen 80;
    server_name app.local;

    location / {
        root /home/your-user/nginx-lab/static;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

这样访问 `/users/123` 时，如果磁盘上没有这个文件，最终会返回 `index.html`，交给前端路由处理。

## 七、本节练习

1. 使用 `root` 部署一个静态页面。
2. 使用 `alias` 暴露 `/downloads/` 路径。
3. 对比 `root` 与 `alias` 的路径拼接结果。
4. 配置 `try_files $uri $uri/ =404`。
5. 配置 SPA 兜底到 `/index.html`。

## 八、你应该掌握

学完本节，你应该能回答：

- `root` 和 `alias` 有什么区别？
- `try_files` 为什么常用于前端项目？
- 静态文件 404 时应该检查哪些路径？

