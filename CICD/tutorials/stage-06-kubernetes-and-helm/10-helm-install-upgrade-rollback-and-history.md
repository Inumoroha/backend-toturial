# 10：Helm 安装、升级、回滚与历史

## 1. 本节目标

这一节学习 Helm 的日常命令。

你要掌握：

- `helm lint`
- `helm template`
- `helm upgrade --install`
- `helm list`
- `helm history`
- `helm rollback`
- `helm uninstall`

## 2. lint 和 template

检查 Chart：

```bash
helm lint deploy/helm/go-cicd-lab
```

渲染 YAML：

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab
```

部署前先 template。

很多 Helm 问题在还没碰集群前就能发现。

## 3. 安装或升级

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=sha-8f3a2c1
```

`upgrade --install` 表示：

- release 不存在就安装。
- release 已存在就升级。

## 4. 等待部署完成

推荐加：

```bash
--wait --timeout 3m
```

完整：

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=sha-8f3a2c1 \
  --wait \
  --timeout 3m
```

如果 readinessProbe 一直失败，Helm 会等待到超时并返回失败。

## 5. atomic

```bash
--atomic
```

如果升级失败，Helm 会尝试回滚。

示例：

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=bad-image \
  --wait \
  --timeout 3m \
  --atomic
```

注意：`--atomic` 不能解决数据库迁移回滚问题。

## 6. 查看 release

```bash
helm list -n go-cicd-lab
```

查看状态：

```bash
helm status go-cicd-lab -n go-cicd-lab
```

查看历史：

```bash
helm history go-cicd-lab -n go-cicd-lab
```

## 7. 回滚

回滚到上一个 revision：

```bash
helm rollback go-cicd-lab -n go-cicd-lab
```

回滚到指定 revision：

```bash
helm rollback go-cicd-lab 2 -n go-cicd-lab
```

查看 rollout：

```bash
kubectl -n go-cicd-lab rollout status deployment/go-cicd-lab
```

## 8. 卸载

```bash
helm uninstall go-cicd-lab -n go-cicd-lab
```

注意：

- Helm 删除它管理的资源。
- PVC、Secret 是否删除取决于资源定义和策略。
- 生产环境谨慎卸载。

## 9. dry-run

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=sha-test \
  --dry-run
```

用于查看将要应用的内容。

## 10. 小练习

1. `helm lint`。
2. `helm template`。
3. `helm upgrade --install`。
4. 修改 image tag 再 upgrade。
5. `helm history`。
6. `helm rollback`。
7. 查看 Pod 是否回到旧镜像。

## 11. 本节小结

你现在应该理解：

- Helm release 有 revision 历史。
- `upgrade --install` 是常用部署命令。
- `--wait` 能让部署等待 ready。
- `--atomic` 可以在升级失败时回滚 Kubernetes 资源。
- Helm rollback 不等于数据库回滚。

