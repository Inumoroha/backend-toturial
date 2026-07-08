# 3. HTTP 方法、状态码、Header 与后端语义

本节目标：掌握常见 HTTP 方法、状态码和 Header 的后端语义，避免接口设计和错误处理混乱。

---

## 一、常见方法

| 方法 | 常见语义 |
| --- | --- |
| GET | 查询资源 |
| POST | 创建资源或提交动作 |
| PUT | 整体替换 |
| PATCH | 部分更新 |
| DELETE | 删除资源 |

方法只是约定，后端仍要正确实现业务语义。

---

## 二、幂等性

幂等表示同一个请求执行一次和多次，最终效果一致。

通常：

- GET 应该幂等。
- PUT 通常幂等。
- DELETE 通常幂等。
- POST 不一定幂等。

写操作如果要支持重试，应该设计幂等键。

---

## 三、常见状态码

| 状态码 | 含义 | 后端场景 |
| --- | --- | --- |
| 200 | 成功 | 查询成功 |
| 201 | 已创建 | 创建资源成功 |
| 204 | 无响应体成功 | 删除成功 |
| 400 | 请求错误 | 参数格式不对 |
| 401 | 未认证 | 没登录 |
| 403 | 无权限 | 登录但无权限 |
| 404 | 不存在 | 资源或路由不存在 |
| 409 | 冲突 | 唯一冲突、版本冲突 |
| 429 | 请求过多 | 限流 |
| 500 | 内部错误 | 程序异常 |
| 502 | 网关连接上游失败 | upstream 不可用 |
| 503 | 服务不可用 | 过载或维护 |
| 504 | 网关等待上游超时 | 上游慢 |

---

## 四、常见 Header

| Header | 作用 |
| --- | --- |
| Content-Type | 请求或响应体格式 |
| Authorization | 认证凭据 |
| Cookie | 浏览器携带 Cookie |
| Set-Cookie | 服务端设置 Cookie |
| User-Agent | 客户端信息 |
| X-Request-Id | 请求追踪 |
| X-Forwarded-For | 代理传递客户端 IP |

---

## 五、Go 中设置状态码和 Header

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
```

注意：

```text
WriteHeader 只能有效调用一次。
如果先 Write body，Go 会默认发送 200。
```

---

## 补充实践：为一个用户资源设计状态码

以用户接口为例，推荐这样设计：

```text
GET    /users/123       查询用户
POST   /users           创建用户
PATCH  /users/123       修改用户部分字段
DELETE /users/123       删除用户
```

对应状态码：

```text
查询成功：200
创建成功：201
删除成功且无响应体：204
JSON 格式错误：400
未登录：401
无权限删除别人账号：403
用户不存在：404
邮箱唯一冲突：409
请求太频繁：429
数据库异常：500
```

不要把所有失败都返回 `200`：

```json
{"code":500,"message":"failed"}
```

这样会让网关、监控、负载均衡、客户端 SDK 都很难正确判断请求是否成功。

---

## 补充代码：统一 JSON 错误响应

```go
func writeJSON(w http.ResponseWriter, status int, value any) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(value)
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{
        "error": message,
    })
}
```

使用：

```go
if r.Method != http.MethodPost {
    writeError(w, http.StatusMethodNotAllowed, "method not allowed")
    return
}
```

注意 `405 Method Not Allowed` 适合“路径存在，但方法不支持”。如果路径根本不存在，才是 `404`。

---

## 六、常见问题

### 1. 参数错误用 400 还是 500？

客户端传错参数通常用 400，不应该用 500。

### 2. 未登录和无权限有什么区别？

未登录是 401，登录了但没权限是 403。

### 3. 502 和 504 区别是什么？

502 通常是网关无法成功连接或获取上游响应。504 是连接上了但等待上游超时。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 根据接口语义选择 HTTP 方法。
- 解释常见状态码。
- 正确区分 400、401、403、404、409、429、500、502、503、504。
- 在 Go handler 中设置 Header 和状态码。
