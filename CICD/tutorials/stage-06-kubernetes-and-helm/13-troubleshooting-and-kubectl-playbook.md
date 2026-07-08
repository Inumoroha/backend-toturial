# 13：kubectl 排障手册

## 1. 本节目标

Kubernetes 排障要靠状态、事件和日志。

这一节给你一套常用 playbook。

## 2. 总览命令

```bash
kubectl -n go-cicd-lab get all
kubectl -n go-cicd-lab get pods -o wide
kubectl -n go-cicd-lab get events --sort-by=.metadata.creationTimestamp
```

## 3. Pod 不 Running

查看：

```bash
kubectl -n go-cicd-lab describe pod <pod>
```

重点看：

- Events。
- Image。
- Node。
- Conditions。
- Restart Count。

常见状态：

- `ImagePullBackOff`
- `CrashLoopBackOff`
- `Pending`
- `CreateContainerConfigError`
- `Running` 但 not ready

## 4. ImagePullBackOff

可能原因：

- 镜像 tag 不存在。
- 镜像仓库私有但没有 imagePullSecret。
- imagePullSecret 名字错。
- registry token 过期。
- 节点无法访问 registry。

排查：

```bash
kubectl -n go-cicd-lab describe pod <pod>
kubectl -n go-cicd-lab get secret
```

## 5. CrashLoopBackOff

容器反复崩溃。

查看日志：

```bash
kubectl -n go-cicd-lab logs <pod>
kubectl -n go-cicd-lab logs <pod> --previous
```

常见原因：

- 应用 panic。
- 必需环境变量缺失。
- 数据库连接失败导致启动退出。
- 命令或 args 错误。
- 端口或配置错误。

## 6. Pending

Pod 一直 Pending。

查看 describe：

```bash
kubectl -n go-cicd-lab describe pod <pod>
```

常见原因：

- 资源 requests 太大。
- 没有可用 Node。
- PVC 无法绑定。
- nodeSelector/tolerations 不匹配。

## 7. Readiness 失败

现象：

```text
Pod Running，但 READY 0/1
```

排查：

```bash
kubectl -n go-cicd-lab describe pod <pod>
kubectl -n go-cicd-lab logs <pod>
```

常见原因：

- `/readyz` 路径错误。
- 应用监听端口错误。
- 数据库不可达。
- readiness timeout 太短。

## 8. Service 不通

查看 Service：

```bash
kubectl -n go-cicd-lab get svc
kubectl -n go-cicd-lab describe svc go-cicd-lab
kubectl -n go-cicd-lab get endpoints go-cicd-lab
```

如果 endpoints 为空，通常是：

- Service selector 和 Pod labels 不匹配。
- Pod not ready。

## 9. Helm 升级失败

查看：

```bash
helm status go-cicd-lab -n go-cicd-lab
helm history go-cicd-lab -n go-cicd-lab
```

渲染：

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab -f deploy/helm/go-cicd-lab/values-staging.yaml
```

常见原因：

- 模板语法错。
- values 类型错。
- immutable field 修改。
- Secret/ConfigMap 不存在。
- probes 导致 `--wait` 超时。

## 10. 临时调试 Pod

如果需要在集群内测试 DNS/网络：

```bash
kubectl -n go-cicd-lab run debug --rm -it --image=busybox:1.36 -- sh
```

在里面：

```sh
nslookup go-cicd-lab
wget -qO- http://go-cicd-lab/healthz
```

生产集群调试 Pod 要遵守权限和审计规则。

## 11. 小练习

故意制造：

1. 错误镜像 tag。
2. 错误 Service selector。
3. 错误 readiness path。
4. 资源 request 过大。

每次记录：

- `kubectl get pods` 状态。
- `describe` 中关键事件。
- 修复方式。

## 12. 本节小结

你现在应该理解：

- Kubernetes 排障先看状态，再看 describe/events/logs。
- ImagePullBackOff、CrashLoopBackOff、Pending、not ready 分别指向不同问题。
- Service 不通常先看 endpoints。
- Helm 失败先 template，再 status/history。

