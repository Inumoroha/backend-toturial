# 02. 使用 Docker 启动 RabbitMQ

## 1. 为什么用 Docker

RabbitMQ 也可以直接安装在 Windows 上，但学习阶段更推荐 Docker：

- 启动快。
- 清理方便。
- 不污染本机环境。
- 后续可以和 Go 项目、数据库一起用 Docker Compose 编排。

本教程使用 RabbitMQ 官方社区 Docker 镜像的管理版：

```text
rabbitmq:4-management
```

其中：

- `4` 表示 RabbitMQ 4.x 系列。
- `management` 表示镜像内启用了 Management Plugin。

## 2. 启动 RabbitMQ 容器

在 PowerShell 执行：

```powershell
docker run -d --hostname rabbitmq-dev --name rabbitmq-dev -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=go_learner -e RABBITMQ_DEFAULT_PASS=go_learner_pwd rabbitmq:4-management
```

参数说明：

| 参数 | 作用 |
| --- | --- |
| `-d` | 后台运行容器 |
| `--hostname rabbitmq-dev` | 设置容器主机名 |
| `--name rabbitmq-dev` | 设置容器名称 |
| `-p 5672:5672` | 映射 AMQP 端口 |
| `-p 15672:15672` | 映射管理后台端口 |
| `RABBITMQ_DEFAULT_USER` | 设置默认用户 |
| `RABBITMQ_DEFAULT_PASS` | 设置默认密码 |
| `rabbitmq:4-management` | 使用 RabbitMQ 4.x 管理版镜像 |

## 3. 检查容器是否启动

执行：

```powershell
docker ps
```

你应该能看到类似信息：

```text
NAMES          IMAGE                   PORTS
rabbitmq-dev   rabbitmq:4-management   0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
```

查看启动日志：

```powershell
docker logs -f rabbitmq-dev
```

看到类似下面的内容时，说明 RabbitMQ 已经启动完成：

```text
Server startup complete
```

按 `Ctrl+C` 可以退出日志查看，不会停止容器。

## 4. 访问管理后台

浏览器打开：

```text
http://localhost:15672
```

登录账号：

```text
Username: go_learner
Password: go_learner_pwd
```

如果能看到 RabbitMQ Overview 页面，说明本地环境已经跑通。

## 5. 使用命令检查 RabbitMQ 状态

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl status
```

这个命令会进入容器执行 `rabbitmqctl status`，用于查看 RabbitMQ 节点状态。

你也可以查看当前队列：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues
```

刚启动时可能没有业务队列，这是正常的。

查看 exchange：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_exchanges name type durable auto_delete
```

RabbitMQ 默认会有一些内置 exchange，例如：

```text
amq.direct
amq.fanout
amq.topic
amq.headers
```

## 6. 停止和删除容器

停止容器：

```powershell
docker stop rabbitmq-dev
```

再次启动已停止的容器：

```powershell
docker start rabbitmq-dev
```

删除容器：

```powershell
docker rm rabbitmq-dev
```

如果容器正在运行，需要先停止再删除：

```powershell
docker stop rabbitmq-dev
docker rm rabbitmq-dev
```

## 7. 容器已存在怎么办

如果启动时报错：

```text
Conflict. The container name "/rabbitmq-dev" is already in use
```

说明已经存在同名容器。

先查看：

```powershell
docker ps -a
```

如果你想继续用原来的容器：

```powershell
docker start rabbitmq-dev
```

如果你想重新创建：

```powershell
docker stop rabbitmq-dev
docker rm rabbitmq-dev
```

然后重新执行启动命令。

## 8. 可选：使用 Docker Compose

如果你更喜欢用 Compose，可以后续在项目根目录创建 `docker-compose.yml`：

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    container_name: rabbitmq-dev
    hostname: rabbitmq-dev
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: go_learner
      RABBITMQ_DEFAULT_PASS: go_learner_pwd
```

启动：

```powershell
docker compose up -d
```

停止：

```powershell
docker compose down
```

第 1 阶段先使用 `docker run` 即可，简单直接。

## 9. 本节小结

你已经完成了本地 RabbitMQ 环境启动。

请确认你能做到：

- 执行 `docker ps` 看到 `rabbitmq-dev`。
- 打开 `http://localhost:15672`。
- 使用 `go_learner / go_learner_pwd` 登录。
- 执行 `rabbitmqctl status` 查看状态。

下一节开始认识 Management UI。

