# Linux、Docker 网络与故障排查

Prometheus 采集失败时，问题通常不在 PromQL，而在进程、端口、路径、网络、权限或响应格式。先掌握从底层到应用层的排查顺序。

## 1. 从进程和端口开始

```bash
ps aux | grep order-service
ss -lntp | grep 8080
lsof -iTCP:8080 -sTCP:LISTEN
curl -v http://127.0.0.1:8080/healthz
```

Windows PowerShell 可用：

```powershell
Get-Process | Where-Object { $_.ProcessName -match "order|go" }
Get-NetTCPConnection -LocalPort 8080
curl.exe -v http://127.0.0.1:8080/healthz
```

先问四个问题：

1. 进程是否存在？
2. 是否监听了预期端口？
3. 是否只监听在 `127.0.0.1`？
4. 本机 curl 能否拿到预期状态码？

## 2. 检查 DNS 和路径

```bash
getent hosts service-name
nslookup service-name
dig +short service-name
curl -v --connect-timeout 3 http://service-name:8080/metrics
curl -v http://service-name:8080/metrics
```

连接失败、连接超时、HTTP 404、HTTP 500 和指标解析错误是不同问题：

| 现象 | 优先检查 |
| --- | --- |
| connection refused | 进程、监听端口、容器端口映射 |
| timeout | 路由、防火墙、NetworkPolicy、服务是否卡住 |
| 404 | `metrics_path`、反向代理路径、路由注册 |
| 401/403 | 认证、Authorization、RBAC 或代理策略 |
| 500 | 应用自身的指标处理器或依赖 |
| parse error | exposition format、Help/Type、重复样本 |

## 3. Docker 网络实验

启动两个容器并观察网络：

```bash
docker network create prom-lab
docker run -d --name demo-web --network prom-lab nginx
docker run --rm --network prom-lab curlimages/curl \
  curl -v http://demo-web:80/
docker inspect demo-web
docker network inspect prom-lab
```

容器内的 `localhost` 指向当前容器，不是宿主机，也不是另一个容器。这是最常见的 Prometheus target 配置错误之一。

在 Compose 中，服务名可以作为 DNS 名称：

```yaml
services:
  prometheus:
    networks: [lab]
  app:
    networks: [lab]
networks:
  lab:
```

此时 Prometheus 应访问 `app:8080`，而不是 `localhost:8080`。

## 4. 一条可复用的排查链

当“Prometheus 抓不到服务”时按顺序执行：

```bash
# 1. 查看目标
curl http://localhost:9090/api/v1/targets

# 2. 从 Prometheus 容器内部访问
docker exec -it prometheus sh
wget -qO- http://app:8080/metrics

# 3. 检查服务端实际暴露的路径
curl -i http://localhost:8080/metrics

# 4. 查看容器日志
docker logs --tail=200 prometheus
docker logs --tail=200 app
```

不要在没有证据时直接把 `scrape_interval` 调短或重启所有容器。重启可能暂时隐藏连接泄漏、启动顺序和 DNS 缓存问题。

## 5. 练习：制造并定位四种故障

依次制造：

1. 停止 app 容器：观察 connection refused。
2. 把 metrics path 改成 `/metric`：观察 404。
3. 修改服务端返回无效文本：观察 parse error。
4. 加入 10 秒延迟：观察 scrape timeout。

每次故障都记录：

- target 页面显示的错误。
- curl 的错误。
- 服务日志。
- 哪一层首先发现故障。
- 修复动作和恢复时间。

## 6. 阶段验收

给自己一个陌生 target，只提供 job 名称和地址。你应该能在 15 分钟内判断它属于哪一种故障，并提供一条可以复现问题的命令，而不是只说“网络有问题”。

