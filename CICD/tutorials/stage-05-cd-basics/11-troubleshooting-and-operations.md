# 11：部署排障与运行维护

## 1. 本节目标

CD 的一半工作是排障。

这一节学习部署失败时如何定位问题。

## 2. 先定位失败位置

看到部署失败，先判断：

```text
workflow 没触发？
approval 卡住？
SSH 失败？
registry pull 失败？
compose up 失败？
容器启动失败？
health check 失败？
业务冒烟测试失败？
```

不要直接猜。

## 3. GitHub Actions 层

检查：

- workflow 是否触发。
- job 是否等待 environment approval。
- environment secrets 是否配置。
- SSH key 是否正确。
- host/user/app_dir 是否正确。
- concurrency 是否让 job 排队。

常见错误：

```text
Permission denied (publickey)
Host key verification failed
No such file or directory
```

## 4. SSH 层

本地验证：

```bash
ssh -i ./go-cicd-lab-deploy deploy@your-server-ip
```

服务器上检查：

```bash
whoami
pwd
docker version
docker compose version
```

如果 deploy 用户不能执行 Docker：

```bash
groups
```

确认在 `docker` 组中，并重新登录。

## 5. Registry 层

服务器拉取失败：

```bash
docker pull ghcr.io/your-name/go-cicd-lab:sha-abc123
```

检查：

- image name 是否小写。
- tag 是否存在。
- 镜像是否私有。
- 服务器是否登录 registry。
- token 是否过期。
- token 是否有 pull 权限。

## 6. Compose 层

查看配置：

```bash
docker compose config
```

拉取：

```bash
docker compose pull api
```

启动：

```bash
docker compose up -d
```

状态：

```bash
docker compose ps
```

日志：

```bash
docker compose logs --tail=200 api
```

## 7. 应用层

容器启动但 health check 失败：

检查：

- 服务是否监听正确端口。
- `HTTP_ADDR` 是否正确。
- `DATABASE_URL` 是否正确。
- 数据库是否 ready。
- 迁移是否执行。
- 应用是否 panic。
- `/healthz` 和 `/readyz` 逻辑是否区分清楚。

## 8. 数据库层

PostgreSQL 容器状态：

```bash
docker compose ps postgres
docker compose logs --tail=100 postgres
```

进入数据库：

```bash
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

注意：生产排障不要随意修改数据。

## 9. 资源层

查看磁盘：

```bash
df -h
```

查看内存：

```bash
free -h
```

查看容器资源：

```bash
docker stats
```

清理旧镜像前先确认回滚需要：

```bash
docker images
docker image prune
```

不要随便删生产正在使用或可能回滚的镜像。

## 10. 部署后维护

至少定期检查：

- 磁盘空间。
- 容器状态。
- 应用日志。
- 镜像仓库 token 是否过期。
- 备份是否可恢复。
- 旧镜像保留策略。
- `.current_image` 是否准确。
- 部署文档是否更新。

## 11. 小练习

制造并排查：

1. 错误镜像 tag。
2. 错误 `DATABASE_URL`。
3. 停止 PostgreSQL 后部署。
4. 修改 SSH key 导致部署失败。

每次记录：

- 失败位置。
- 关键日志。
- 修复命令。
- 如何预防。

## 12. 本节小结

你现在应该理解：

- CD 排障要分层：Actions、SSH、registry、Compose、应用、数据库、资源。
- `docker compose ps/logs/config` 是单机部署排障核心命令。
- 生产排障要谨慎处理数据和镜像清理。
- 维护工作同样属于交付质量的一部分。

