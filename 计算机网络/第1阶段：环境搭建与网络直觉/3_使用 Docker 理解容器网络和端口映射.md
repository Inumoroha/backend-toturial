# 3. 使用 Docker 理解容器网络和端口映射

本节目标：通过 Docker 实验理解容器 IP、宿主机 IP、端口映射、容器内外 `localhost` 的区别。这些概念是后续学习 Nginx、网关、Kubernetes 网络的基础。

很多 Go 后端初学者第一次部署服务时会遇到这个问题：

```text
服务在容器里启动成功了，为什么宿主机访问不到？
```

答案通常不在 Go 代码里，而在监听地址、端口映射和容器网络里。

---

## 一、先理解宿主机和容器

宿主机就是运行 Docker 的机器。

容器是运行在宿主机上的隔离环境。容器有自己的：

- 文件系统。
- 进程空间。
- 网络命名空间。
- IP 地址。
- 路由表。

所以容器内的：

```text
127.0.0.1
```

指的是容器自己，不是宿主机。

这句话非常重要：

```text
容器里的 localhost，不等于宿主机的 localhost。
```

---

## 二、查看 Docker 网络

查看 Docker 网络列表：

```bash
docker network ls
```

你通常会看到：

```text
bridge
host
none
```

默认情况下，普通容器会连接到 `bridge` 网络。

查看 bridge 网络详情：

```bash
docker network inspect bridge
```

关注：

- `Subnet`：Docker bridge 的网段。
- `Gateway`：容器默认网关。
- `Containers`：当前连接到这个网络的容器。

---

## 三、启动一个容器观察网络

启动 Alpine 容器：

```bash
docker run --rm -it alpine sh
```

如果镜像不存在，Docker 会自动拉取。

容器内执行：

```sh
ip addr
ip route
cat /etc/resolv.conf
```

如果 Alpine 默认没有 `ip` 命令，可以先安装：

```sh
apk add --no-cache iproute2
```

观察：

- 容器通常有一个 `eth0`。
- 容器 IP 可能类似 `172.17.0.2`。
- 默认网关可能是 `172.17.0.1`。
- DNS 可能由 Docker 注入。

---

## 四、容器访问外网

容器内执行：

```sh
ping 8.8.8.8
```

再测试 HTTP：

```sh
wget -qO- https://example.com
```

如果 `ping` 不通但 `wget` 能访问，不一定是异常，可能是 ICMP 被限制。

容器访问外网时，通常会经过宿主机做 NAT。容器使用私有 IP 发包，宿主机会把源地址转换成宿主机可访问外网的地址。

---

## 五、端口映射是什么

假设容器内有服务监听：

```text
容器内 8080
```

如果没有端口映射，宿主机通常不能直接通过 `127.0.0.1:8080` 访问它。

端口映射写法：

```bash
docker run -p 8080:8080 ...
```

含义是：

```text
宿主机 8080 -> 容器 8080
```

如果写成：

```bash
docker run -p 8081:8080 ...
```

含义是：

```text
宿主机 8081 -> 容器 8080
```

左边是宿主机端口，右边是容器端口。

---

## 六、实验：运行一个 Nginx 容器

启动 Nginx：

```bash
docker run --rm --name web-demo -p 8080:80 -d nginx:alpine
```

参数说明：

- `--rm`：容器停止后自动删除。
- `--name web-demo`：容器名称。
- `-p 8080:80`：宿主机 8080 映射到容器 80。
- `-d`：后台运行。
- `nginx:alpine`：使用轻量 Nginx 镜像。

访问：

```bash
curl http://127.0.0.1:8080
```

如果看到 Nginx 欢迎页面，说明端口映射成功。

查看端口映射：

```bash
docker port web-demo
```

查看容器日志：

```bash
docker logs web-demo
```

停止容器：

```bash
docker stop web-demo
```

---

## 七、故意制造端口访问失败

不做端口映射启动：

```bash
docker run --rm --name web-demo -d nginx:alpine
```

此时访问：

```bash
curl http://127.0.0.1:8080
```

大概率失败，因为宿主机没有把 `8080` 转发到容器。

查看容器自己的 IP：

```bash
docker inspect web-demo
```

你可能能从宿主机访问容器 IP，但这不是推荐的稳定方式。容器 IP 会变化，项目中应该使用端口映射、Docker 网络名称或服务发现。

停止：

```bash
docker stop web-demo
```

---

## 八、容器内服务监听 127.0.0.1 的坑

如果 Go 服务在容器内监听：

```go
http.ListenAndServe("127.0.0.1:8080", nil)
```

它只监听容器自己的回环地址。

即使你做了：

```bash
docker run -p 8080:8080 ...
```

宿主机也可能访问不到。

容器内服务应该监听：

```go
http.ListenAndServe(":8080", nil)
```

或者：

```go
http.ListenAndServe("0.0.0.0:8080", nil)
```

---

## 九、自定义 Docker 网络

创建网络：

```bash
docker network create app-net
```

启动 Nginx：

```bash
docker run --rm -d --name web-demo --network app-net nginx:alpine
```

启动一个调试容器：

```bash
docker run --rm -it --network app-net alpine sh
```

在调试容器中访问：

```sh
wget -qO- http://web-demo
```

在自定义网络中，Docker 可以用容器名做 DNS 解析。

清理：

```bash
docker stop web-demo
docker network rm app-net
```

---

## 十、Go 后端中的对应关系

当你把 Go 服务放进容器时，要同时确认三件事：

```text
Go 服务监听什么地址和端口？
Docker 是否做了端口映射？
访问方应该访问宿主机端口、容器名，还是服务名？
```

常见错误：

```text
服务监听 127.0.0.1:8080。
Dockerfile 写了 EXPOSE 8080。
运行容器时没有 -p。
然后以为宿主机一定能访问。
```

`EXPOSE` 只是镜像元信息，不等于发布端口。真正发布端口要用 `-p` 或 Docker Compose 的 `ports`。

---

## 十一、常见问题

### 1. EXPOSE 8080 是否等于开放端口？

不是。`EXPOSE` 只是声明容器内服务可能使用的端口，真正映射到宿主机要使用 `-p`。

### 2. 端口映射 `8081:8080` 哪边是宿主机？

左边是宿主机，右边是容器。

```text
宿主机 8081 -> 容器 8080
```

### 3. 容器内访问 localhost 是访问谁？

访问容器自己，不是宿主机。

### 4. 为什么容器 IP 不适合写死？

容器重建后 IP 可能变化。容器之间通信应优先使用自定义网络中的容器名，生产环境使用服务发现或 Kubernetes Service。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 使用 `docker network ls` 和 `docker network inspect` 查看 Docker 网络。
- 在容器内查看 IP、路由、DNS。
- 解释 `-p 8080:80` 的含义。
- 解释容器内 `localhost` 和宿主机 `localhost` 的区别。
- 使用 Nginx 容器完成端口映射访问。
- 解释为什么 Go 服务在容器内应该监听 `0.0.0.0` 或 `:端口`。

完成这些之后，就可以进入下一节：使用 Wireshark 与 tcpdump 做第一次抓包。

