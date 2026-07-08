# 03 Nginx 基础：指令上下文、include 与进程模型

## 本节目标

理解三个经常被忽略但很重要的基础：

- 指令必须写在正确的上下文。
- 配置文件通过 `include` 组合成最终配置。
- Nginx 的 master/worker 进程如何工作。

## 一、什么是上下文

Nginx 配置不是随便写的。不同指令只能出现在指定层级。

常见上下文：

```nginx
main
events {
}
http {
    server {
        location / {
        }
    }
}
```

例如 `server` 必须写在 `http` 里，不能写在 `events` 里。

## 二、上下文错误示例

错误：

```nginx
events {
    server {
        listen 80;
    }
}
```

检查：

```bash
sudo nginx -t
```

你会看到类似：

```text
"server" directive is not allowed here
```

这个错误的意思不是 server 写错了，而是写错位置了。

## 三、常见指令位置

| 指令 | 常见位置 |
| --- | --- |
| `worker_processes` | main |
| `events` | main |
| `http` | main |
| `server` | http |
| `location` | server |
| `upstream` | http |
| `log_format` | http |
| `access_log` | http、server、location |
| `proxy_pass` | location |
| `limit_req_zone` | http |
| `limit_req` | server、location |

如果不确定，就查官方文档中的 `Context`。

## 四、include 如何工作

主配置通常包含：

```nginx
include /etc/nginx/conf.d/*.conf;
```

这表示 Nginx 会加载 `conf.d` 下所有 `.conf` 文件。

排查配置是否生效时，用：

```bash
sudo nginx -T
```

然后搜索你的域名：

```bash
sudo nginx -T | grep api.local
```

如果搜不到，说明你的配置没有被加载。

## 五、配置拆分建议

学习阶段可以这样拆：

```text
/etc/nginx/nginx.conf
/etc/nginx/conf.d/
  00-log-format.conf
  10-upstream-go-api.conf
  20-api.local.conf
  30-static.local.conf
```

数字前缀能帮助你看出加载顺序和职责。

注意：很多配置不依赖文件顺序，但 `map`、`log_format`、`upstream` 等通常应放在被使用之前，或者至少放在 `http` 上下文中。

## 六、master 与 worker

查看进程：

```bash
ps -ef | grep nginx
```

你会看到：

```text
nginx: master process
nginx: worker process
```

master 负责：

- 读取配置。
- 管理 worker。
- 接收 reload 信号。

worker 负责：

- 处理客户端连接。
- 读取请求。
- 转发 upstream。
- 返回响应。

## 七、reload 发生了什么

执行：

```bash
sudo systemctl reload nginx
```

大致过程：

1. master 检查新配置。
2. 如果配置合法，启动新的 worker。
3. 旧 worker 处理完已有连接后退出。
4. 新请求由新 worker 处理。

这就是为什么 Nginx 支持较平滑的配置重载。

## 八、本节练习

1. 故意把 `server` 写到错误位置，观察报错。
2. 用 `nginx -T` 找到自己写的 server。
3. 拆出一个 `10-upstream.conf` 和一个 `20-api.conf`。
4. 查看 master 和 worker 进程。
5. reload 后再看 worker 进程变化。

## 九、你应该掌握

学完本节，你应该知道配置写在哪里、为什么没有生效，以及 reload 为什么比直接 kill 进程更适合生产环境。

