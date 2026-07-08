# 4. 接口测试工具与 HTTP 请求直觉

本节目标：学会用接口测试工具观察一次 HTTP 请求和响应。后端开发不是只写代码，还要会验证接口。

---

## 一、一次 HTTP 请求包含什么

一次请求通常包含：

```text
请求方法：GET、POST、PUT、DELETE
请求地址：http://localhost:8080/ping
请求头：Content-Type、Authorization 等
请求体：JSON、表单、文件等
```

一次响应通常包含：

```text
状态码：200、400、401、404、500
响应头：Content-Type 等
响应体：通常是 JSON
```

你写 Gin 接口时，本质上就是读取请求，返回响应。

---

## 二、创建 requests.http

在项目根目录创建：

```text
requests.http
```

写入：

```http
### ping
GET http://localhost:8080/ping

### hello with query
GET http://localhost:8080/hello?name=alice

### not found
GET http://localhost:8080/not-exists
```

如果使用 VS Code REST Client，点击每个请求上方的 `Send Request` 即可发送。

---

## 三、观察成功响应

请求：

```http
GET http://localhost:8080/ping
```

正常状态码：

```text
200 OK
```

响应体：

```json
{
  "message": "pong"
}
```

这说明：

- 服务正在运行。
- 路由 `/ping` 存在。
- handler 正常写回 JSON。

---

## 四、观察 404

请求：

```http
GET http://localhost:8080/not-exists
```

通常返回：

```text
404 Not Found
```

404 的含义是：请求路径没有匹配到任何路由。

排查顺序：

1. 路径是否拼错。
2. API 前缀是否漏了。
3. 服务是否是最新代码。
4. 是否请求到了正确端口。

---

## 五、观察请求日志

标准库默认不会像 Gin 那样打印请求日志。你可以先手动打印：

```go
func pingHandler(w http.ResponseWriter, r *http.Request) {
    log.Printf("%s %s", r.Method, r.URL.Path)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "message": "pong",
    })
}
```

请求后终端会看到：

```text
GET /ping
```

后面使用 Gin 的 `gin.Default()` 时，它会默认帮你打印请求日志。

---

## 六、GET 和 POST 的直觉

`GET` 常用于查询：

```text
GET /users
GET /users/1
```

`POST` 常用于创建：

```text
POST /users
```

`PUT` 常用于整体更新：

```text
PUT /users/1
```

`PATCH` 常用于局部更新：

```text
PATCH /users/1/status
```

`DELETE` 常用于删除：

```text
DELETE /users/1
```

Gin 学习里要不断强化这个直觉，不要把所有接口都写成 `POST`。

---

## 七、本节练习

完成：

- 创建 `requests.http`。
- 写入 `/ping` 请求。
- 写入 `/hello?name=alice` 请求。
- 写入一个故意不存在的请求。
- 分别记录 200 和 404 的现象。

---

## 八、本节验收

你应该能够回答：

- 一次 HTTP 请求由哪些部分组成？
- 一次 HTTP 响应由哪些部分组成？
- 404 通常说明什么？
- GET 和 POST 在语义上有什么区别？
- 为什么接口请求文件值得保存？

