# 1. root 与 alias 路径映射

本节目标：彻底理解 `root` 和 `alias` 的区别。静态资源 404 的一大半问题，都来自路径拼接理解错误。

---

## 一、准备练习目录

创建目录：

```bash
mkdir -p ~/nginx-lab/static/assets
```

创建首页：

```bash
cat > ~/nginx-lab/static/index.html <<'EOF'
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nginx Static Lab</title>
</head>
<body>
  <h1>Nginx Static Lab</h1>
</body>
</html>
EOF
```

创建 CSS：

```bash
cat > ~/nginx-lab/static/assets/style.css <<'EOF'
body {
  font-family: Arial, sans-serif;
}
EOF
```

如果你不想用 shell 创建，也可以手动创建这些文件。

---

## 二、root 的路径规则

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

请求：

```text
/assets/style.css
```

真实文件路径：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

公式：

```text
真实路径 = root 路径 + 完整 URI
```

---

## 三、alias 的路径规则

配置：

```nginx
server {
    listen 80;
    server_name static.local;

    location /downloads/ {
        alias /home/your-user/nginx-lab/static/assets/;
    }
}
```

请求：

```text
/downloads/style.css
```

真实文件路径：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

公式：

```text
真实路径 = alias 路径 + 去掉 location 前缀后的 URI
```

---

## 四、对比示例

### root

```nginx
location /assets/ {
    root /home/your-user/nginx-lab/static;
}
```

请求：

```text
/assets/style.css
```

路径：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

### alias

```nginx
location /assets/ {
    alias /home/your-user/nginx-lab/static/assets/;
}
```

请求：

```text
/assets/style.css
```

路径：

```text
/home/your-user/nginx-lab/static/assets/style.css
```

结果一样，但思路不同：

- `root` 保留完整 URI。
- `alias` 替换 location 前缀。

---

## 五、alias 末尾斜杠问题

推荐：

```nginx
location /assets/ {
    alias /home/your-user/nginx-lab/static/assets/;
}
```

`location` 和 `alias` 都以 `/` 结尾，最不容易出错。

不推荐初学阶段写各种不对齐的形式。

---

## 六、什么时候用 root，什么时候用 alias

使用 `root`：

```text
URL 路径结构和磁盘目录结构一致。
```

例如：

```text
/assets/app.js -> /var/www/app/assets/app.js
```

使用 `alias`：

```text
URL 路径只是一个对外别名，和磁盘目录不一致。
```

例如：

```text
/downloads/manual.pdf -> /data/files/manual.pdf
```

---

## 七、验证命令

```bash
curl -I http://static.local/
curl -I http://static.local/assets/style.css
curl -I http://static.local/downloads/style.css
```

如果返回 404，先手算真实文件路径，再去服务器上确认文件是否存在。

---

## 八、本节练习

1. 使用 `root` 部署首页。
2. 使用 `root` 访问 `/assets/style.css`。
3. 使用 `alias` 把 `/downloads/` 映射到 assets 目录。
4. 故意把 alias 路径写错，观察 404。
5. 手算每个请求对应的真实文件路径。

---

## 九、本节复盘

请确认你能回答：

1. `root` 的路径拼接公式是什么？
2. `alias` 的路径拼接公式是什么？
3. 为什么 alias 末尾斜杠容易出错？
4. 静态资源 404 时第一步应该做什么？
5. 什么场景更适合用 alias？

