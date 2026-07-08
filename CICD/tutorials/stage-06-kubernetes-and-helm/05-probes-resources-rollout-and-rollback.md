# 05：Probes、资源、滚动更新与回滚

## 1. 本节目标

这一节让 Deployment 更接近真实生产。

你要学习：

- readinessProbe。
- livenessProbe。
- startupProbe。
- resources requests/limits。
- rollout status。
- rollout undo。

## 2. 三种 Probe

### readinessProbe

判断 Pod 是否可以接流量。

失败时，Service 不会把流量发给该 Pod。

### livenessProbe

判断容器是否需要重启。

失败超过阈值时，kubelet 会重启容器。

### startupProbe

给慢启动应用更多启动时间。

startupProbe 成功前，liveness/readiness 不会接管。

## 3. Go API 推荐 probes

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: http
  failureThreshold: 30
  periodSeconds: 2

readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

经验：

- readiness 可以检查数据库连接。
- liveness 不要依赖太多外部服务，否则数据库短暂抖动可能导致应用被重启。
- startup 适合启动慢的服务。

## 4. resources

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

requests：

- 调度器用来决定 Pod 放到哪个 Node。
- 表示应用正常运行期望资源。

limits：

- 限制容器最多能用多少资源。
- 内存超过 limit 可能被 OOMKilled。

初学不要乱给很小的内存 limit，容易导致看不懂的重启。

## 5. 滚动更新策略

Deployment 默认 RollingUpdate。

可以显式写：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

含义：

- 更新时至少保持当前可用副本数量。
- 最多额外创建 1 个 Pod。

## 6. 查看 rollout

```bash
kubectl -n go-cicd-lab rollout status deployment/go-cicd-lab
kubectl -n go-cicd-lab rollout history deployment/go-cicd-lab
```

查看 ReplicaSet：

```bash
kubectl -n go-cicd-lab get rs
```

## 7. 回滚

回滚到上一个版本：

```bash
kubectl -n go-cicd-lab rollout undo deployment/go-cicd-lab
```

回滚到指定 revision：

```bash
kubectl -n go-cicd-lab rollout undo deployment/go-cicd-lab --to-revision=2
```

注意：这只回滚 Deployment 规格，例如镜像。

它不会回滚数据库迁移，也不会自动回滚外部配置。

## 8. rollout 失败常见原因

- 镜像拉取失败。
- readinessProbe 一直失败。
- 容器启动后崩溃。
- 资源不足无法调度。
- ConfigMap/Secret 不存在。
- 端口配置错误。

排查：

```bash
kubectl -n go-cicd-lab get pods
kubectl -n go-cicd-lab describe pod <pod>
kubectl -n go-cicd-lab logs <pod>
```

## 9. 小练习

1. 给 Deployment 添加 probes。
2. 给 Deployment 添加 resources。
3. 修改镜像 tag，观察 rollout。
4. 故意把 readiness path 改错，观察 rollout 卡住。
5. 用 describe 和 logs 排查。
6. 回滚。

## 10. 本节小结

你现在应该理解：

- readiness 决定是否接流量。
- liveness 决定是否重启。
- startup 保护慢启动。
- requests/limits 影响调度和资源隔离。
- Deployment 支持 rollout history 和 rollback。

