# 02-Docker 与端口网络问题排查

## 学习目标

理解 Docker bridge 网络、端口映射、容器内外访问差异，并能排查“容器服务启动了但访问不到”的问题。

## 一、Docker 默认网络

Docker 默认使用 bridge 网络。容器会获得一个私有 IP，通常在类似 `172.17.0.0/16` 的网段。

查看网络：

```bash
docker network ls
docker network inspect bridge
```

容器内查看：

```bash
ip addr
ip route
```

## 二、端口映射

容器内服务监听 `8080`，宿主机不一定能直接访问。需要端口映射：

```bash
docker run -p 8080:8080 your-image
```

含义：

```text
宿主机 8080 -> 容器 8080
```

也可以写成：

```bash
docker run -p 8081:8080 your-image
```

含义：

```text
宿主机 8081 -> 容器 8080
```

## 三、容器内监听地址

如果 Go 服务在容器内监听：

```go
http.ListenAndServe("127.0.0.1:8080", nil)
```

即使做了 `-p 8080:8080`，宿主机也可能访问不到。

容器内服务应该监听：

```go
http.ListenAndServe("0.0.0.0:8080", nil)
```

或：

```go
http.ListenAndServe(":8080", nil)
```

## 四、实验：构建 Go HTTP 容器

`main.go`：

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello from container")
    })
    http.ListenAndServe(":8080", nil)
}
```

`Dockerfile`：

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.20
WORKDIR /app
COPY --from=build /app/server .
EXPOSE 8080
CMD ["./server"]
```

构建运行：

```bash
docker build -t go-net-demo .
docker run --rm -p 8080:8080 go-net-demo
```

访问：

```bash
curl http://127.0.0.1:8080
```

## 五、排查步骤

如果访问不到：

1. 容器是否运行？

   ```bash
   docker ps
   ```

2. 端口是否映射？

   ```bash
   docker port 容器ID
   ```

3. 容器内服务是否监听？

   ```bash
   docker exec -it 容器ID sh
   ss -lnt
   ```

4. 服务是否监听 `0.0.0.0`？

   ```bash
   ss -lntp
   ```

5. 宿主机端口是否监听？

   ```bash
   ss -lntp | grep 8080
   ```

## 六、容器互联

创建自定义网络：

```bash
docker network create app-net
```

启动两个容器在同一网络：

```bash
docker run -d --name svc-a --network app-net go-net-demo
docker run --rm -it --network app-net alpine sh
```

在 alpine 容器中访问：

```sh
wget -qO- http://svc-a:8080
```

Docker 自定义网络支持通过容器名解析。

## 七、练习题

1. `-p 8081:8080` 表示什么？
2. 为什么容器内监听 `127.0.0.1` 会导致外部访问失败？
3. `EXPOSE 8080` 是否等于发布端口？
4. 如何进入容器确认服务是否监听？

## 八、验收标准

你能独立把一个 Go HTTP 服务打包成 Docker 镜像，并能排查容器端口映射和监听地址问题。

