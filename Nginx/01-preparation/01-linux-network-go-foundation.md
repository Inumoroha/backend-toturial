# 01 准备阶段：Linux、网络与 Go HTTP 基础

## 本节目标

学习 Nginx 之前，你需要先具备三个基础：

- 能在 Linux 上操作文件、服务、进程和日志。
- 能理解一次 HTTP 请求从浏览器到服务端的大致路径。
- 能写出一个最小可运行的 Go HTTP 服务。

这些基础不需要一次学到特别深，但必须能用于排查问题。Nginx 很多错误并不是 Nginx 自身的问题，而是端口、权限、DNS、后端服务、网络连接出了问题。

## 一、Linux 必备命令

### 文件与目录

常用命令：

```bash
pwd
ls -lah
cd /etc/nginx
cat nginx.conf
less nginx.conf
cp old.conf new.conf
mv a.conf b.conf
rm test.conf
```

你需要能看懂：

- 当前在哪个目录。
- 配置文件是否存在。
- 文件权限是否允许 Nginx 读取。
- 修改的是不是正确的配置文件。

### 进程与端口

常用命令：

```bash
ps aux | grep nginx
ps aux | grep my-go-app
ss -lntp
lsof -i :80
lsof -i :8080
```

关注点：

- Nginx 是否启动。
- Go 服务是否启动。
- 80、443、8080 这些端口是否被占用。
- 服务监听的是 `127.0.0.1`、`0.0.0.0`，还是某个内网 IP。

### 服务管理

常用命令：

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl status nginx
```

学习重点：

- `restart` 会重启服务。
- `reload` 通常用于平滑加载新配置。
- 修改 Nginx 配置后，先执行 `nginx -t`，再 reload。

## 二、网络基础

### 请求链路

用户访问 `https://api.example.com/users` 时，通常会经历：

1. 浏览器解析域名，得到服务器 IP。
2. 浏览器与服务器建立 TCP 连接。
3. 如果是 HTTPS，浏览器与服务器进行 TLS 握手。
4. 请求进入 Nginx。
5. Nginx 根据 `server_name` 和 `location` 找到匹配配置。
6. Nginx 把请求转发给 Go 服务。
7. Go 服务处理请求并返回响应。
8. Nginx 把响应返回给客户端。

你以后排障时，要不断问自己：请求现在卡在哪一层？

### 常见状态码

- `200`：请求成功。
- `301`、`302`：重定向。
- `400`：客户端请求有问题。
- `401`：未认证。
- `403`：无权限。
- `404`：资源不存在或路由不匹配。
- `413`：请求体太大。
- `499`：客户端主动断开连接，Nginx 常见日志状态码。
- `500`：后端服务内部错误。
- `502`：Nginx 连不上后端，或后端返回了异常响应。
- `503`：服务不可用。
- `504`：Nginx 等后端响应超时。

## 三、用 curl 分析请求

`curl` 是学习 Nginx 必备工具。

```bash
curl http://127.0.0.1
curl -v http://127.0.0.1
curl -I http://127.0.0.1
curl -H "Host: api.local" http://127.0.0.1
curl -X POST http://127.0.0.1/api/users -d '{"name":"tom"}' -H "Content-Type: application/json"
```

常用参数：

- `-v`：查看请求和响应细节。
- `-I`：只看响应头。
- `-H`：添加请求头。
- `-X`：指定请求方法。
- `-d`：发送请求体。

## 四、Go HTTP 最小服务

创建 `main.go`：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "pong")
	})

	http.HandleFunc("/headers", func(w http.ResponseWriter, r *http.Request) {
		for k, values := range r.Header {
			for _, v := range values {
				fmt.Fprintf(w, "%s: %s\n", k, v)
			}
		}
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行：

```bash
go run main.go
```

验证：

```bash
curl http://127.0.0.1:8080/ping
curl http://127.0.0.1:8080/headers
```

如果你能看到 `pong` 和请求头信息，就说明 Go 服务准备好了。

## 五、本节练习

1. 启动 Go 服务并访问 `/ping`。
2. 用 `ss -lntp` 查看 Go 服务监听端口。
3. 用 `curl -v` 查看完整请求过程。
4. 停掉 Go 服务，再次 curl，观察错误。
5. 修改 Go 服务端口为 `8081`，重新运行并验证。

## 六、你应该掌握

学完本节，你应该能回答：

- 怎么判断一个端口是否在监听？
- 怎么判断 Nginx 或 Go 服务是否启动？
- `curl -v` 能帮你看到什么？
- 浏览器访问一个接口时，Nginx 在链路中处于什么位置？

