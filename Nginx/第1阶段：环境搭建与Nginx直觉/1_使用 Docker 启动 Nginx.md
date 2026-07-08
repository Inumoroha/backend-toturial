# 1. 使用 Docker 启动 Nginx

本节目标：使用 Docker 在本地启动一个 Nginx，理解容器端口映射、默认页面、日志查看、停止删除和端口冲突处理。

推荐初学阶段先使用 Docker 启动 Nginx。原因不是 Docker 比系统安装更“高级”，而是它更适合反复练习。你可以很快启动、删除、换配置、重来一遍。

---

## 一、前置条件

开始之前，请确认：

- 已安装 Docker Desktop 或 Docker Engine。
- Docker 正在运行。
- 终端中可以执行 `docker --version`。
- 本机 `8080` 端口没有被其他服务占用。

检查 Docker：

```bash
docker --version
```

如果你想使用本机 `80` 端口，也可以，但学习阶段推荐先使用 `8080`，避免和系统已有服务冲突。

---

## 二、最小启动命令

PowerShell 写法：

```bash
docker run --name my-nginx `
  -p 8080:80 `
  -d nginx:stable
```

Linux/macOS 写法：

```bash
docker run --name my-nginx \
  -p 8080:80 \
  -d nginx:stable
```

参数解释：

- `docker run`：创建并启动一个新容器。
- `--name my-nginx`：给容器起名，方便后面管理。
- `-p 8080:80`：把本机 `8080` 端口映射到容器内 `80` 端口。
- `-d`：后台运行容器。
- `nginx:stable`：使用 Nginx 稳定版镜像。

注意端口映射：

```text
8080:80
左边 8080：你电脑上的端口。
右边 80：容器里面 Nginx 监听的端口。
```

所以你访问的是：

```text
http://localhost:8080
```

而不是：

```text
http://localhost:80
```

---

## 三、验证是否启动成功

查看正在运行的容器：

```bash
docker ps
```

如果能看到 `my-nginx`，说明容器正在运行。

访问 Nginx：

```bash
curl http://localhost:8080
```

你应该能看到 Nginx 默认欢迎页的 HTML 内容。

也可以用浏览器打开：

```text
http://localhost:8080
```

---

## 四、查看容器日志

```bash
docker logs my-nginx
```

再发一次请求：

```bash
curl http://localhost:8080
```

然后继续查看日志：

```bash
docker logs my-nginx
```

你会看到访问记录。这个记录来自 Nginx access log。

实时查看：

```bash
docker logs -f my-nginx
```

保持这个窗口不动，再打开另一个终端请求：

```bash
curl http://localhost:8080
```

你会看到日志实时出现。

---

## 五、进入容器查看配置文件

进入容器：

```bash
docker exec -it my-nginx sh
```

查看配置目录：

```bash
ls -lah /etc/nginx
```

常见文件：

```text
/etc/nginx/nginx.conf
/etc/nginx/conf.d/default.conf
```

查看默认 server 配置：

```bash
cat /etc/nginx/conf.d/default.conf
```

你会看到类似：

```nginx
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

这里说明：

- Nginx 在容器内监听 `80`。
- 默认静态文件目录是 `/usr/share/nginx/html`。
- 访问 `/` 会返回这个目录下的 `index.html`。

退出容器：

```bash
exit
```

---

## 六、停止、启动、删除容器

停止：

```bash
docker stop my-nginx
```

再次访问：

```bash
curl http://localhost:8080
```

你会发现连接失败，因为 Nginx 容器已经停止。

重新启动：

```bash
docker start my-nginx
```

删除容器前需要先停止：

```bash
docker stop my-nginx
docker rm my-nginx
```

查看所有容器，包括停止的：

```bash
docker ps -a
```

---

## 七、端口冲突处理

如果启动时报错：

```text
Bind for 0.0.0.0:8080 failed: port is already allocated
```

说明本机 `8080` 端口被占用了。

解决方式一：换端口。

```bash
docker run --name my-nginx `
  -p 8081:80 `
  -d nginx:stable
```

访问：

```bash
curl http://localhost:8081
```

解决方式二：找到占用端口的进程。

Windows PowerShell：

```powershell
netstat -ano | findstr :8080
```

Linux：

```bash
ss -lntp | grep 8080
```

---

## 八、使用自定义静态页面

创建学习目录：

```bash
mkdir nginx-docker-lab
cd nginx-docker-lab
mkdir html
```

创建 `html/index.html`：

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nginx Docker Lab</title>
</head>
<body>
  <h1>Hello Nginx</h1>
  <p>这是通过 Docker 挂载进去的页面。</p>
</body>
</html>
```

删除旧容器：

```bash
docker stop my-nginx
docker rm my-nginx
```

PowerShell 启动：

```bash
docker run --name my-nginx `
  -p 8080:80 `
  -v ${PWD}/html:/usr/share/nginx/html:ro `
  -d nginx:stable
```

Linux/macOS 启动：

```bash
docker run --name my-nginx \
  -p 8080:80 \
  -v "$PWD/html:/usr/share/nginx/html:ro" \
  -d nginx:stable
```

访问：

```bash
curl http://localhost:8080
```

你应该能看到自己写的 HTML。

参数解释：

- `-v`：挂载目录。
- `${PWD}/html`：本机当前目录下的 `html`。
- `/usr/share/nginx/html`：容器内 Nginx 默认静态目录。
- `:ro`：只读挂载，容器内不能修改这个目录。

---

## 九、本节常见问题

### 1. 容器启动后马上退出

查看日志：

```bash
docker logs my-nginx
```

常见原因：

- 挂载了错误配置。
- 配置语法错误。
- 端口被占用。

### 2. 页面还是默认页，不是自己的 index.html

检查：

- 是否挂载到了 `/usr/share/nginx/html`。
- 本机 `html/index.html` 是否存在。
- 是否访问了正确端口。
- 旧容器是否没有删除，仍在运行旧配置。

### 3. 修改 HTML 后浏览器没变化

先用 curl 验证：

```bash
curl http://localhost:8080
```

如果 curl 已经变了，浏览器可能缓存了页面，刷新或清缓存即可。

---

## 十、本节练习

1. 使用 Docker 启动 Nginx。
2. 用 `curl` 访问默认页面。
3. 查看 `docker logs`。
4. 进入容器查看 `/etc/nginx/conf.d/default.conf`。
5. 挂载自己的 `index.html`。
6. 故意占用或重复使用 `8080`，观察端口冲突错误。
7. 换成 `8081` 端口重新启动。

---

## 十一、本节复盘

请确认你能回答：

1. `-p 8080:80` 的左右两边分别代表什么？
2. Nginx 容器内默认静态目录在哪里？
3. 如何查看 Nginx 容器日志？
4. 容器启动失败时第一步看什么？
5. 为什么学习阶段推荐先用 `8080` 而不是 `80`？

