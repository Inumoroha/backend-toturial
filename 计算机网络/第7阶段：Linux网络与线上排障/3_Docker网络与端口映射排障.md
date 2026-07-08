# 3. Docker 网络与端口映射排障

本节目标：排查“容器服务启动了但宿主机访问不到”的常见问题。

---

## 一、排查顺序

```text
容器是否运行？
容器内服务是否监听？
是否监听 0.0.0.0？
是否做了端口映射？
宿主机端口是否监听？
防火墙是否放行？
```

---

## 二、常用命令

查看容器：

```bash
docker ps
```

查看日志：

```bash
docker logs 容器ID
```

查看端口映射：

```bash
docker port 容器ID
```

进入容器：

```bash
docker exec -it 容器ID sh
```

容器内查看端口：

```bash
ss -lnt
```

---

## 三、监听地址问题

容器内错误写法：

```go
http.ListenAndServe("127.0.0.1:8080", nil)
```

推荐：

```go
http.ListenAndServe(":8080", nil)
```

或者：

```go
http.ListenAndServe("0.0.0.0:8080", nil)
```

---

## 四、端口映射问题

```bash
docker run -p 8081:8080 app
```

访问宿主机：

```text
宿主机IP:8081
```

左边是宿主机端口，右边是容器端口。

---

## 补充实验：复现监听 127.0.0.1 导致宿主机访问失败

容器内如果服务只监听：

```go
http.ListenAndServe("127.0.0.1:8080", nil)
```

它只接受容器自己网络命名空间里的本地访问。即使你写了：

```bash
docker run -p 8081:8080 app
```

宿主机访问也可能失败。

正确监听：

```go
http.ListenAndServe("0.0.0.0:8080", nil)
```

或者：

```go
http.ListenAndServe(":8080", nil)
```

验证方法：

```bash
docker exec -it 容器ID sh
ss -lntp
```

如果看到：

```text
127.0.0.1:8080
```

说明只监听容器内 localhost。

如果看到：

```text
0.0.0.0:8080
```

说明容器内所有网卡都能接收。

---

## 补充排障：容器端口映射四步确认

第一步，确认容器还在运行：

```bash
docker ps
```

第二步，确认端口映射：

```bash
docker port 容器ID
```

第三步，在容器内访问服务：

```bash
docker exec -it 容器ID sh
wget -O- http://127.0.0.1:8080/healthz
```

第四步，在宿主机访问映射端口：

```bash
curl -i http://127.0.0.1:8081/healthz
```

判断：

```text
容器内失败：服务本身没起来或监听错。
容器内成功、宿主机失败：端口映射、防火墙、Docker 网络问题。
宿主机成功、外部失败：云安全组或宿主机防火墙问题。
```

---

## 补充理解：容器访问宿主机服务

容器内的：

```text
127.0.0.1
```

指的是容器自己，不是宿主机。

如果容器要访问宿主机上的服务，常见方式：

```text
Docker Desktop：host.docker.internal
Linux：使用 docker0 网关地址，例如 172.17.0.1
自定义 bridge：查看 docker network inspect
```

查看默认网关：

```bash
docker exec -it 容器ID sh
ip route
```

如果容器内访问 `localhost:5432` 失败，但宿主机 PostgreSQL 正常，先确认你访问的是不是错误的 localhost。

---

## 补充现场判断：先分清访问方向

Docker 网络问题一定要先分清方向：

```text
宿主机访问容器。
容器访问宿主机。
容器访问容器。
容器访问外网。
外部机器访问宿主机映射端口。
```

每个方向的排查点不同。比如宿主机访问容器重点看 `-p` 和监听地址；容器访问宿主机重点看 `host.docker.internal` 或网关地址；外部机器访问宿主机重点看云安全组和宿主机防火墙。

---

## 五、常见问题

### 1. EXPOSE 是否等于发布端口？

不是。发布端口需要 `-p`。

### 2. 容器内 localhost 是谁？

是容器自己，不是宿主机。

### 3. docker ps 看不到端口映射怎么办？

说明启动容器时可能没有加 `-p`。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 排查 Docker 端口映射。
- 解释容器内 localhost。
- 确认容器内服务监听地址。
- 解释 `8081:8080` 的含义。
