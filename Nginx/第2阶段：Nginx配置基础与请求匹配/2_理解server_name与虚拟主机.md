# 2. 理解 server_name 与虚拟主机

本节目标：理解 Nginx 如何在同一个端口上服务多个域名，并掌握用 `curl -H "Host: ..."` 测试虚拟主机。

---

## 一、什么是虚拟主机

一台服务器可以同时服务多个域名：

```text
api.local     -> Go API
static.local  -> 静态页面
admin.local   -> 后台服务
```

它们都可以监听同一个 `80` 端口。Nginx 通过请求头里的 `Host` 判断应该进入哪个 `server`。

---

## 二、准备三个 server

创建配置：

```nginx
server {
    listen 80;
    server_name api.local;

    location / {
        return 200 "api server\n";
    }
}

server {
    listen 80;
    server_name static.local;

    location / {
        return 200 "static server\n";
    }
}

server {
    listen 80;
    server_name admin.local;

    location / {
        return 200 "admin server\n";
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 三、方式一：配置 hosts 测试

编辑 hosts：

```bash
sudo vim /etc/hosts
```

加入：

```text
127.0.0.1 api.local
127.0.0.1 static.local
127.0.0.1 admin.local
```

测试：

```bash
curl http://api.local
curl http://static.local
curl http://admin.local
```

---

## 四、方式二：不改 hosts，直接指定 Host

```bash
curl -H "Host: api.local" http://127.0.0.1
curl -H "Host: static.local" http://127.0.0.1
curl -H "Host: admin.local" http://127.0.0.1
```

这种方式在排查线上问题时很有用。

例如你想测试某台机器上的 Nginx 配置，但 DNS 还没切过去，就可以用 Host 头直接测。

---

## 五、默认 server

如果请求 Host 没有匹配任何 `server_name`，Nginx 会使用默认 server。

可以显式声明：

```nginx
server {
    listen 80 default_server;
    server_name _;

    return 404 "unknown host\n";
}
```

测试：

```bash
curl -H "Host: unknown.local" http://127.0.0.1
```

---

## 六、server_name 支持多个域名

```nginx
server {
    listen 80;
    server_name example.com www.example.com api.example.com;

    location / {
        return 200 "matched\n";
    }
}
```

也支持通配和正则，但学习阶段先少用。明确域名更容易维护。

---

## 七、常见问题

### 1. 明明写了 api.local，但访问到别的 server

检查：

```bash
curl -v http://api.local
```

看请求 Host 是否正确。

再看：

```bash
sudo nginx -T | grep -A 20 api.local
```

确认配置是否加载。

### 2. 浏览器访问不对，curl 对

可能是浏览器缓存、hosts 配置不一致、代理软件影响。

先以 curl 为准排查：

```bash
curl -H "Host: api.local" http://127.0.0.1
```

### 3. Docker 中访问域名失败

Docker 容器内的 `localhost` 指容器自己，不是宿主机。容器网络会在部署阶段详细讲。

---

## 八、本节练习

1. 配置 `api.local`、`static.local`、`admin.local`。
2. 用 hosts 测试。
3. 不改 hosts，用 `curl -H "Host: ..."` 测试。
4. 配置 `default_server`。
5. 请求未知 Host，观察默认 server 响应。

---

## 九、本节复盘

请确认你能回答：

1. 同一个 80 端口为什么能服务多个域名？
2. `server_name` 匹配的是请求中的什么信息？
3. 不改 hosts 如何测试某个 server？
4. `default_server` 有什么作用？
5. 线上灰度切 DNS 前，如何直接测试某台机器？

