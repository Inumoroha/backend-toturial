# 4. FileServer 静态文件服务

本节目标：学会使用 `http.FileServer` 提供静态目录访问。

---

## 一、最小静态文件服务

目录：

```text
public/
  index.html
  app.css
```

代码：

```go
fs := http.FileServer(http.Dir("./public"))
mux.Handle("GET /static/", http.StripPrefix("/static/", fs))
```

访问：

```text
http://localhost:8080/static/index.html
```

---

## 二、StripPrefix 的作用

请求路径是：

```text
/static/index.html
```

但文件真实路径是：

```text
./public/index.html
```

如果不去掉 `/static/` 前缀，FileServer 会去找：

```text
./public/static/index.html
```

所以需要：

```go
http.StripPrefix("/static/", fs)
```

---

## 三、静态文件适合放什么

适合：

- 图片。
- CSS。
- JavaScript。
- 简单 HTML。
- 下载附件的公开资源。

不适合：

- 需要复杂鉴权的私有文件。
- 用户上传但未经校验的文件。
- 敏感配置。

---

## 四、目录列表问题

默认 FileServer 在某些情况下可能展示目录列表。生产中要谨慎公开目录，避免把不该暴露的文件放进去。

更稳妥的做法是：

- 静态目录只放可公开文件。
- 上传文件和静态资源分开。
- 私有文件通过 Handler 校验权限后再返回。

---

## 五、本节检查点

请确认你能回答：

- `http.FileServer` 做什么？
- `http.StripPrefix` 为什么常和 FileServer 一起用？
- 静态目录里不应该放什么？
- 私有文件为什么不适合直接用 FileServer 暴露？

