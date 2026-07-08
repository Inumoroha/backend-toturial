# 03 静态资源服务：权限、MIME 与路径排障

## 本节目标

静态资源问题最常见的是 404、403、MIME 类型错误和缓存没生效。本节专门训练排障能力。

## 一、Nginx 运行用户

查看 Nginx 使用哪个用户运行 worker：

```bash
ps -ef | grep "nginx: worker"
```

也可以看主配置：

```bash
grep '^user' /etc/nginx/nginx.conf
```

常见用户：

- `www-data`
- `nginx`

静态文件必须允许这个用户读取。

## 二、目录权限为什么重要

读取文件不只需要文件本身有读权限，还需要上级目录有执行权限。

检查完整路径权限：

```bash
namei -l /home/your-user/nginx-lab/static/index.html
```

如果中间某个目录对 Nginx 用户不可进入，就会 403。

## 三、403 排查顺序

1. 请求是否访问的是目录而不是文件。
2. 目录下是否有 `index.html`。
3. `autoindex` 是否关闭。
4. Nginx 用户是否有读取文件权限。
5. Nginx 用户是否有进入上级目录权限。
6. SELinux 是否阻止访问，主要见于 CentOS 系。

临时验证权限时可以把静态目录放到 `/var/www`，避免用户家目录权限干扰。

## 四、MIME 类型

如果 JS 或 CSS 被浏览器拒绝，可能是 MIME 类型不对。

确认 `http` 块中有：

```nginx
include /etc/nginx/mime.types;
default_type application/octet-stream;
```

验证：

```bash
curl -I http://static.local/assets/style.css
```

应该看到类似：

```text
Content-Type: text/css
```

JS 文件应是：

```text
Content-Type: application/javascript
```

## 五、路径排障公式

遇到 404，先按公式算真实文件路径。

### root

```nginx
location /assets/ {
    root /var/www/app;
}
```

请求：

```text
/assets/app.js
```

真实路径：

```text
/var/www/app/assets/app.js
```

### alias

```nginx
location /assets/ {
    alias /var/www/app/static/;
}
```

请求：

```text
/assets/app.js
```

真实路径：

```text
/var/www/app/static/app.js
```

## 六、缓存没生效怎么查

查看响应头：

```bash
curl -I http://static.local/assets/app.js
```

关注：

```text
Cache-Control
Expires
ETag
Last-Modified
```

如果没有缓存头，检查：

- 请求是否真的匹配到 `/assets/`。
- `add_header` 是否写在正确的 location。
- 该状态码是否会应用 `add_header`。
- 是否需要 `always` 参数。

## 七、完整排障练习

1. 把静态目录放到一个 Nginx 无权限访问的位置，复现 403。
2. 使用 `namei -l` 找到权限断点。
3. 故意把 `alias` 末尾 `/` 去掉，观察路径错误。
4. 删除 `mime.types` include，观察 CSS 的 Content-Type。
5. 给 `/assets/` 添加缓存头，用 `curl -I` 验证。

## 八、你应该掌握

学完本节，你应该能独立排查静态资源 404、403、MIME 类型异常和缓存头不生效的问题。

