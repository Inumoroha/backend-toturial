# 02 静态资源服务：缓存、gzip 与错误页

## 本节目标

掌握静态资源服务中的三个常见生产配置：

- 浏览器缓存。
- gzip 压缩。
- 自定义错误页。

这些配置能减少带宽消耗，提高加载速度，也能让错误页面更友好。

## 一、浏览器缓存

静态资源通常可以缓存，例如图片、CSS、JS。

```nginx
server {
    listen 80;
    server_name static.local;

    location /assets/ {
        root /home/your-user/nginx-lab/static;
        expires 7d;
        add_header Cache-Control "public";
    }
}
```

验证：

```bash
curl -I http://static.local/assets/style.css
```

观察响应头里是否有：

```text
Cache-Control
Expires
```

## 二、缓存策略建议

### 带 hash 的文件

例如：

```text
app.8f3a91.js
style.1c2d3e.css
```

可以设置较长缓存：

```nginx
location /assets/ {
    root /home/your-user/nginx-lab/static;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

### HTML 文件

HTML 通常不要设置很长缓存，因为它负责引用最新的 JS、CSS。

```nginx
location = /index.html {
    root /home/your-user/nginx-lab/static;
    add_header Cache-Control "no-cache";
}
```

## 三、开启 gzip

gzip 可以压缩文本资源。

```nginx
http {
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        application/json
        application/javascript
        application/xml;
}
```

验证：

```bash
curl -H "Accept-Encoding: gzip" -I http://static.local/assets/style.css
```

观察：

```text
Content-Encoding: gzip
```

注意：很小的文件压缩收益不明显，图片、视频通常不需要 gzip。

## 四、自定义错误页

创建错误页：

```bash
mkdir -p ~/nginx-lab/static/errors
```

`~/nginx-lab/static/errors/404.html`：

```html
<!doctype html>
<html>
<body>
  <h1>404 Not Found</h1>
  <p>页面不存在。</p>
</body>
</html>
```

Nginx 配置：

```nginx
server {
    listen 80;
    server_name static.local;

    root /home/your-user/nginx-lab/static;

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /errors/404.html;

    location /errors/ {
        internal;
    }
}
```

`internal` 表示这个路径只能由 Nginx 内部跳转访问，用户不能直接访问。

## 五、静态资源常见排障

### 404

检查：

- URL 是否正确。
- `root` 或 `alias` 路径是否正确。
- 文件是否真的存在。
- Nginx 运行用户是否有权限读取。
- `try_files` 是否把请求导向了其他路径。

### 403

常见原因：

- 文件或目录权限不足。
- 访问目录但没有 index 文件。
- 目录列表功能没有开启。

检查权限：

```bash
namei -l /home/your-user/nginx-lab/static/index.html
```

## 六、完整示例

```nginx
server {
    listen 80;
    server_name static.local;

    root /home/your-user/nginx-lab/static;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /assets/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    error_page 404 /errors/404.html;

    location /errors/ {
        internal;
    }
}
```

## 七、本节练习

1. 给 `/assets/` 配置 7 天缓存。
2. 给 `index.html` 配置 `no-cache`。
3. 开启 gzip 并用 `curl` 验证。
4. 配置自定义 404 页面。
5. 故意制造权限问题，观察 403。

## 八、你应该掌握

学完本节，你应该能独立部署一个前端静态项目，并能处理缓存、压缩、404、403 等常见问题。

