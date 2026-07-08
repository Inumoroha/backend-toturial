# 3. 缓存、gzip、权限与 MIME 排查

本节目标：掌握静态资源上线最常见的四类问题：缓存、压缩、权限和 MIME 类型。

---

## 一、静态资源缓存

给 `/assets/` 配置缓存：

```nginx
location ^~ /assets/ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    try_files $uri =404;
}
```

验证：

```bash
curl -I http://static.local/assets/style.css
```

关注：

```text
Cache-Control
Expires
```

建议：

- 带 hash 的 JS/CSS 可以长缓存。
- `index.html` 不要长缓存。

---

## 二、index.html 不长缓存

```nginx
location = /index.html {
    add_header Cache-Control "no-cache";
}
```

原因：HTML 负责引用最新的 JS/CSS。如果 HTML 被长缓存，用户可能一直加载旧资源。

---

## 三、开启 gzip

在 `http` 上下文：

```nginx
gzip on;
gzip_min_length 1024;
gzip_comp_level 5;
gzip_types text/plain text/css application/json application/javascript;
```

验证：

```bash
curl -H "Accept-Encoding: gzip" -I http://static.local/assets/style.css
```

关注：

```text
Content-Encoding: gzip
```

注意：图片、视频通常已经压缩，不需要 gzip。

---

## 四、MIME 类型

主配置中应有：

```nginx
include /etc/nginx/mime.types;
default_type application/octet-stream;
```

检查 CSS：

```bash
curl -I http://static.local/assets/style.css
```

期望：

```text
Content-Type: text/css
```

如果 CSS 或 JS MIME 错误，浏览器可能拒绝加载。

---

## 五、权限排查

403 常见原因是 Nginx 用户没有权限读取文件。

查看 worker 用户：

```bash
ps -ef | grep "nginx: worker"
```

检查路径权限：

```bash
namei -l /home/your-user/nginx-lab/static/index.html
```

注意：不只是文件要可读，上级目录也要可进入。

---

## 六、404 排查

按路径公式手算：

```text
root: 真实路径 = root + URI
alias: 真实路径 = alias + 去掉 location 前缀后的 URI
```

然后确认文件是否存在：

```bash
ls -lah /真实/文件/路径
```

---

## 七、本节练习

1. 给 `/assets/` 配置 30 天缓存。
2. 给 `index.html` 配置 `no-cache`。
3. 开启 gzip。
4. 用 `curl -I` 检查 MIME。
5. 用 `namei -l` 检查权限。
6. 故意写错路径，复现 404 并修复。

---

## 八、本节复盘

请确认你能回答：

1. 为什么 assets 可以长缓存而 index.html 不适合？
2. gzip 适合压缩哪些资源？
3. MIME 类型错误会导致什么问题？
4. 403 和 404 排查方向有什么区别？
5. `namei -l` 在权限排查中有什么用？

