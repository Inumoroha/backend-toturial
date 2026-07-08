# 01 Nginx 基础：安装、启动与配置检查

## 本节目标

你要掌握 Nginx 最基本的工作流：

1. 修改配置。
2. 检查配置。
3. 重新加载配置。
4. 访问验证。
5. 查看日志。

这个流程以后会重复很多次。

## 一、确认 Nginx 安装成功

```bash
nginx -v
sudo systemctl status nginx
curl http://127.0.0.1
```

如果服务没有启动：

```bash
sudo systemctl start nginx
```

如果希望开机自启：

```bash
sudo systemctl enable nginx
```

## 二、Nginx 常用命令

### 检查配置

```bash
sudo nginx -t
```

输出类似：

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

这说明语法没问题。注意：语法正确不代表业务逻辑正确，比如路径写错、后端端口写错，`nginx -t` 不一定能发现。

### 查看完整配置

```bash
sudo nginx -T
```

它会把主配置和 include 进来的配置全部打印出来。排查“为什么配置没生效”时很有用。

### 平滑重载

```bash
sudo systemctl reload nginx
```

或者：

```bash
sudo nginx -s reload
```

推荐学习阶段使用：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

先检查，再重载。

## 三、写第一个 server 配置

在 `/etc/nginx/conf.d/hello.conf` 中写入：

```nginx
server {
    listen 80;
    server_name hello.local;

    location / {
        return 200 "hello nginx\n";
    }
}
```

配置 hosts：

```text
127.0.0.1 hello.local
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

验证：

```bash
curl http://hello.local
```

应返回：

```text
hello nginx
```

## 四、理解 server_name

Nginx 可以在同一个 80 端口上服务多个域名：

```nginx
server {
    listen 80;
    server_name api.local;

    location / {
        return 200 "api\n";
    }
}

server {
    listen 80;
    server_name static.local;

    location / {
        return 200 "static\n";
    }
}
```

Nginx 会根据请求头里的 `Host` 选择对应的 `server`。

测试：

```bash
curl -H "Host: api.local" http://127.0.0.1
curl -H "Host: static.local" http://127.0.0.1
```

## 五、常见错误

### 端口被占用

现象：

```text
bind() to 0.0.0.0:80 failed
```

排查：

```bash
sudo lsof -i :80
sudo ss -lntp
```

### 配置少了分号

Nginx 指令大多以 `;` 结尾：

```nginx
return 200 "hello"
```

这是错误的，应该是：

```nginx
return 200 "hello";
```

### 配置文件没有被 include

如果你把配置放在了一个 Nginx 不会加载的位置，reload 后不会生效。

用下面命令确认：

```bash
sudo nginx -T | grep hello.local
```

## 六、本节练习

1. 创建 `hello.local`，返回 `hello nginx`。
2. 创建 `api.local` 和 `static.local` 两个 server。
3. 故意删掉一个分号，执行 `nginx -t` 观察错误。
4. 恢复配置并 reload。
5. 用 `curl -H "Host: xxx"` 测试不同 server。

## 七、你应该掌握

学完本节，你应该能熟练完成：

- 新增一个 Nginx server。
- 检查并重载配置。
- 用 Host 头测试虚拟主机。
- 通过错误信息定位基础配置问题。

