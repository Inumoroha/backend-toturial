# 07：冒烟测试与部署验证

## 1. 本节目标

部署命令执行完不代表服务可用。

这一节学习如何验证部署：

- `/healthz`。
- `/readyz`。
- `/version`。
- 核心业务接口冒烟测试。
- 日志和容器状态。

## 2. 三类检查

### 进程健康

```text
GET /healthz
```

表示服务进程还活着。

### 依赖就绪

```text
GET /readyz
```

表示服务可以接流量，通常需要检查：

- 数据库连接。
- Redis 连接。
- 必要配置。

### 版本确认

```text
GET /version
```

返回：

```json
{
  "version": "v0.1.0",
  "commit": "8f3a2c1",
  "date": "2026-07-04T10:00:00Z"
}
```

它能确认部署的确实是目标版本。

## 3. 冒烟测试是什么

冒烟测试是部署后最小业务验证。

它不追求覆盖所有功能，只确认核心路径没有明显坏掉。

例如 todo API：

```text
GET /healthz
GET /readyz
GET /version
POST /todos
GET /todos
```

如果冒烟测试失败，部署应该失败。

## 4. smoke-test.sh 示例

创建：

```text
scripts/smoke-test.sh
```

内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${BASE_URL:?BASE_URL is required}"

curl -fsS "$BASE_URL/healthz" >/dev/null
curl -fsS "$BASE_URL/readyz" >/dev/null
curl -fsS "$BASE_URL/version" >/dev/null

echo "Smoke tests passed for $BASE_URL"
```

如果有认证接口，可以给测试环境配置专用 smoke test token。

不要使用生产管理员账号做自动测试。

## 5. 在 deploy.sh 中调用

简单方式：

```bash
curl -fsS "$HEALTH_URL" >/dev/null
curl -fsS "$READY_URL" >/dev/null
```

更完整：

```bash
BASE_URL="$BASE_URL" ./scripts/smoke-test.sh
```

如果脚本在服务器上，需要确保它已复制到 app 目录。

## 6. GitHub Actions 中做外部验证

部署 job 后可以增加 step：

```yaml
- name: Smoke test
  run: |
    curl -fsS "${{ secrets.HEALTH_URL }}"
    curl -fsS "${{ secrets.READY_URL }}"
```

但是注意网络：

- GitHub hosted runner 必须能访问你的服务 URL。
- 如果服务只在内网，外部 runner 访问不到。
- 内网服务可以在服务器本机 deploy.sh 中验证，或使用 self-hosted runner。

## 7. 等待服务变 ready

服务启动需要时间。

不要只 sleep 固定时间：

```bash
sleep 30
```

更好的方式是循环检查：

```bash
deadline=$((SECONDS + 90))
until curl -fsS "$READY_URL" >/dev/null; do
  if [ "$SECONDS" -ge "$deadline" ]; then
    exit 1
  fi
  sleep 3
done
```

这样服务快时不用等，服务慢时也有明确超时。

## 8. 部署验证失败时输出什么

失败时至少输出：

```bash
docker compose ps
docker compose logs --tail=100 api
cat .current_image || true
cat .previous_image || true
```

这能帮助你判断：

- 容器是否运行。
- 是否 crash loop。
- 当前镜像是什么。
- 上一个镜像是什么。
- 应用日志中有什么错误。

## 9. 小练习

1. 给 Go 服务添加 `/version`。
2. 创建 `scripts/smoke-test.sh`。
3. 在本地或服务器执行：

```bash
BASE_URL=http://127.0.0.1:8080 ./scripts/smoke-test.sh
```

4. 故意让 `/readyz` 返回失败，观察部署脚本是否失败。

## 10. 本节小结

你现在应该理解：

- 部署成功必须经过验证。
- `/healthz`、`/readyz`、`/version` 分别解决不同问题。
- 冒烟测试覆盖最小核心路径。
- 等待 ready 应该用循环和超时。
- 验证失败时要输出足够日志。

