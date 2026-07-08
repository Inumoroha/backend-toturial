# 03：.dockerignore、本地构建运行与排查

## 1. 本节目标

这一节学习本地镜像工作流：

```text
写 Dockerfile
-> 写 .dockerignore
-> docker build
-> docker run
-> docker logs
-> docker inspect
-> docker image rm
```

本地跑通后，再放进 CI。

## 2. .dockerignore 是什么

Docker build 时会把构建上下文发送给 Docker daemon 或 BuildKit。

`.dockerignore` 用来排除不需要进入构建上下文的文件。

如果不写，可能把这些都传进去：

- `.git/`
- `bin/`
- `coverage.out`
- `.env`
- 本地日志。
- 临时文件。
- IDE 配置。

这会导致：

- 构建变慢。
- 缓存更容易失效。
- 密钥误进构建上下文。

## 3. 推荐 .dockerignore

项目根目录创建：

```text
.dockerignore
```

内容：

```dockerignore
.git
.github
bin
coverage.out
*.test
*.out
.env
.env.*
!.env.example
*.pem
*.key
node_modules
tmp
dist
```

是否忽略 `.github` 取决于你是否需要把 workflow 文件放进镜像。通常不需要。

## 4. 构建镜像

基础命令：

```bash
docker build -t go-cicd-lab:local .
```

含义：

- `-t go-cicd-lab:local`：给镜像打 tag。
- `.`：使用当前目录作为构建上下文。

查看镜像：

```bash
docker images go-cicd-lab
```

## 5. 查看构建日志

如果构建失败，先看失败步骤：

```text
COPY go.mod go.sum ./
RUN go mod download
RUN go build ...
```

常见失败：

- `go.sum` 不存在。
- module path 和 `-ldflags` 不一致。
- `./cmd/server` 目录不存在。
- CGO 依赖导致静态构建失败。
- `.dockerignore` 误排除了必要文件。

## 6. 运行容器

如果服务监听 `8080`：

```bash
docker run --rm -p 8080:8080 go-cicd-lab:local
```

后台运行：

```bash
docker run -d --name go-cicd-lab -p 8080:8080 go-cicd-lab:local
```

查看日志：

```bash
docker logs go-cicd-lab
```

停止并删除：

```bash
docker rm -f go-cicd-lab
```

## 7. 传环境变量

```bash
docker run --rm \
  -e APP_ENV=local \
  -e HTTP_ADDR=:8080 \
  -p 8080:8080 \
  go-cicd-lab:local
```

不要在镜像中写死环境变量。

本地可以使用 `--env-file`：

```bash
docker run --rm --env-file .env -p 8080:8080 go-cicd-lab:local
```

注意：`.env` 不能提交到 Git，也不能 COPY 进镜像。

## 8. 测试健康检查

容器启动后：

```bash
curl -f http://localhost:8080/healthz
```

如果 PowerShell 没有合适的 `curl` 行为，可以用：

```powershell
Invoke-WebRequest http://localhost:8080/healthz
```

## 9. 查看镜像详情

```bash
docker inspect go-cicd-lab:local
```

查看镜像历史：

```bash
docker history go-cicd-lab:local
```

查看容器状态：

```bash
docker ps
docker ps -a
```

## 10. distroless 镜像怎么排查

distroless 没有 shell，所以这个通常不能用：

```bash
docker exec -it go-cicd-lab sh
```

排查方式：

- 看 `docker logs`。
- 本地用非 distroless debug 镜像临时排查。
- 增加结构化日志。
- 提供 `/healthz`、`/readyz`、`/version`。
- 使用 `docker inspect` 查看配置。

生产镜像不应该为了方便 exec shell 而长期保留大量调试工具。

## 11. 清理镜像

删除指定镜像：

```bash
docker image rm go-cicd-lab:local
```

清理悬空镜像：

```bash
docker image prune
```

谨慎清理所有未使用资源：

```bash
docker system prune
```

执行清理前确认不会删掉你需要的本地镜像和容器。

## 12. 小练习

完成：

1. 创建 `.dockerignore`。
2. 构建镜像：

```bash
docker build -t go-cicd-lab:local .
```

3. 运行容器：

```bash
docker run --rm -p 8080:8080 go-cicd-lab:local
```

4. 新开终端测试：

```bash
curl -f http://localhost:8080/healthz
```

5. 查看镜像历史：

```bash
docker history go-cicd-lab:local
```

## 13. 本节小结

你现在应该理解：

- `.dockerignore` 可以减少构建上下文和泄密风险。
- 本地先用 `docker build` 和 `docker run` 验证镜像。
- 容器日志和健康检查是排查启动问题的第一入口。
- distroless 镜像更安全，但调试方式不同。

