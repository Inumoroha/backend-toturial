# 4. HTTP 反向代理项目总览与单上游转发

本节目标：实现一个最小 HTTP 反向代理，把请求从代理转发到一个后端服务。

---

## 一、项目目录

```text
http-reverse-proxy/
  go.mod
  cmd/backend/main.go
  cmd/proxy/main.go
```

---

## 二、测试后端

```go
func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello from backend")
    })
    http.ListenAndServe(":8081", nil)
}
```

---

## 三、单上游代理

```go
target, _ := url.Parse("http://127.0.0.1:8081")
proxy := httputil.NewSingleHostReverseProxy(target)

srv := &http.Server{
    Addr:              ":8080",
    Handler:           proxy,
    ReadHeaderTimeout: 2 * time.Second,
}

srv.ListenAndServe()
```

---

## 四、测试

启动后端：

```bash
go run ./cmd/backend
```

启动代理：

```bash
go run ./cmd/proxy
```

请求：

```bash
curl -v http://127.0.0.1:8080/hello
```

---

## 补充落地步骤：先做单上游闭环

第一版反向代理只做单上游：

```text
客户端 -> proxy:8080 -> backend:9001
```

验收顺序：

```bash
curl -i http://127.0.0.1:9001/healthz
curl -i http://127.0.0.1:8080/healthz
```

如果第一个失败，问题在后端。

如果第一个成功、第二个失败，问题在代理。

实现时先不要做负载均衡、健康检查、限流。先保证：

```text
路径能转发。
Query 能保留。
Header 能传递。
状态码能原样返回。
Body 能正常返回。
```

---

## 补充排障：单上游代理失败看哪里

常见失败：

```text
proxy 返回 502：上游没启动、地址错、端口错。
proxy 返回 404：路径被改错或上游没有该路由。
请求卡住：上游慢或代理没有设置超时。
Header 丢失：Director 没正确设置请求。
```

日志至少要记录：

```text
method
path
upstream
status
duration
error
```

---

## 补充文件职责：单上游代理可以这样拆

建议拆成：

```text
cmd/proxy/main.go       启动入口。
internal/proxy/proxy.go 创建 ReverseProxy。
internal/proxy/log.go   访问日志中间件。
cmd/backend/main.go     测试上游。
```

第一版先不读配置文件，直接在代码里写上游地址。等主链路跑通后，再把上游地址改成环境变量或配置文件。

这样可以减少一开始的变量，让问题更容易定位。

---

## 补充验收命令：单上游必须保留状态码

准备三个测试：

```bash
curl -i http://127.0.0.1:9001/ok
curl -i http://127.0.0.1:9001/not-found
curl -i http://127.0.0.1:8080/not-found
```

代理应该保留上游状态码，而不是把所有响应都改成 `200`。

还要验证 Query：

```bash
curl -i "http://127.0.0.1:8080/search?q=go"
```

上游应该能看到 `q=go`。这说明代理没有丢失 URL 信息。

---

## 五、常见问题

### 1. 反向代理是否只是转发 body？

不是。它还要处理 URL、Host、Header、连接、错误和响应回写。

### 2. 代理到后端是否复用客户端连接？

不是同一条 TCP 连接。客户端到代理、代理到后端是两段连接。

### 3. 单上游版本有什么价值？

它先跑通主链路，后续才能安全加入负载均衡、超时和健康检查。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `httputil.NewSingleHostReverseProxy`。
- 启动后端和代理。
- 通过代理访问后端。
- 解释代理和后端之间是两段 TCP 连接。
