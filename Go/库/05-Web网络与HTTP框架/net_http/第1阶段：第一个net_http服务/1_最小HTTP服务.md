# 1. 最小 HTTP 服务

本节目标：写出第一个能运行、能访问、能返回响应的 `net/http` 服务。

---

## 一、创建练习目录

如果你还没有练习目录，可以在当前教程目录下创建：

```bash
mkdir playground
cd playground
go mod init example.com/net-http-playground
```

新建 `main.go`。

---

## 二、写最小服务

代码：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "hello, net/http")
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行：

```bash
go run .
```

访问：

```bash
curl -i http://localhost:8080/hello
```

你应该看到类似：

```http
HTTP/1.1 200 OK
Date: Sun, 05 Jul 2026 09:00:00 GMT
Content-Length: 16
Content-Type: text/plain; charset=utf-8

hello, net/http
```

---

## 三、代码逐行解释

导入包：

```go
import (
	"fmt"
	"log"
	"net/http"
)
```

- `fmt` 用来写文本响应。
- `log` 用来打印启动日志和错误。
- `net/http` 是本阶段主角。

注册路由：

```go
http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello, net/http")
})
```

意思是：当请求路径匹配 `/hello` 时，执行这个函数。

启动服务：

```go
log.Fatal(http.ListenAndServe(":8080", nil))
```

意思是：监听本机 `8080` 端口，开始处理 HTTP 请求。

第二个参数传 `nil` 时，Go 会使用默认路由器 `http.DefaultServeMux`。初学可以这样写，但项目中更推荐自己创建 `http.NewServeMux()`，后面会讲。

---

## 四、为什么 ListenAndServe 会阻塞

`http.ListenAndServe` 会一直运行，等待客户端请求。

它不会执行一下就结束，因为 HTTP 服务的工作方式是：

```text
启动
-> 监听端口
-> 等待请求
-> 处理请求
-> 继续等待请求
```

所以通常它写在 `main` 函数最后。

如果启动失败，例如端口被占用，它会返回错误。我们用 `log.Fatal` 打印错误并退出程序。

---

## 五、端口被占用怎么办

如果看到类似错误：

```text
listen tcp :8080: bind: address already in use
```

说明 `8080` 端口已经被其他程序占用了。

解决方式：

1. 停掉占用端口的程序。
2. 或者换一个端口，例如：

```go
log.Fatal(http.ListenAndServe(":8081", nil))
```

再访问：

```bash
curl -i http://localhost:8081/hello
```

---

## 六、加一个 health 接口

修改代码：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "hello, net/http")
	})

	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

验证：

```bash
curl -i http://localhost:8080/health
curl -i http://localhost:8080/hello
```

---

## 七、本节常见坑

### 1. 修改代码后忘记重启

`go run .` 启动的是当前编译出的程序。修改代码后，需要停止旧进程，再重新运行。

在终端按：

```text
Ctrl+C
```

然后重新执行：

```bash
go run .
```

### 2. URL 写错

`/hello` 和 `/Hello` 是不同路径。HTTP Path 通常区分大小写。

### 3. 端口写错

代码监听 `:8080`，curl 也要访问 `localhost:8080`。

---

## 八、本节检查点

请确认你能做到：

- 写出最小 `net/http` 服务。
- 启动服务并访问 `/hello`。
- 添加 `/health` 接口。
- 看懂 `ListenAndServe` 的作用。
- 处理端口占用问题。

