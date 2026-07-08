# 20. NetworkPolicy 与服务访问边界

本节目标：学完后你能用 NetworkPolicy 限制服务之间的访问范围，避免所有 Pod 默认互通。

## 简短引入

很多本地集群里，Pod 之间默认可以互相访问。学习阶段这很方便，但真实项目中不应该让任意服务都能连数据库、订单服务或管理后台。NetworkPolicy 可以理解为 Kubernetes 里的网络访问规则。

## 一、为什么需要它

真实后端项目常见访问关系：

- `short-api` 可以访问 PostgreSQL。
- `admin-api` 可以访问权限服务。
- 普通业务服务不应该访问运维管理接口。
- 测试环境服务不应该误连生产数据库。

RBAC 管的是 Kubernetes API 权限，NetworkPolicy 管的是 Pod 网络访问边界。两者解决的问题不同。

```text
最小权限不只包括 API 权限，也包括网络访问范围。能不互通的服务，就不要默认互通。
```

## 二、基本用法

先给应用 Pod 和数据库 Pod 打标签：

```yaml
metadata:
  labels:
    app: short-api
```

```yaml
metadata:
  labels:
    app: postgres
```

只允许 `short-api` 访问 PostgreSQL 的 5432 端口：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-short-api-to-postgres
  namespace: backend-dev
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: short-api
      ports:
        - protocol: TCP
          port: 5432
```

应用：

```bash
kubectl apply -f networkpolicy.yaml
kubectl get networkpolicy -n backend-dev
```

## 三、关键参数/语法/代码结构

- `podSelector`：这条策略保护哪些 Pod。
- `policyTypes`：策略类型，常见是 `Ingress` 和 `Egress`。
- `ingress.from`：允许谁访问被保护的 Pod。
- `ports`：允许访问哪些端口。
- `namespaceSelector`：按 namespace 标签选择来源或目标。

要注意：NetworkPolicy 是否生效取决于集群网络插件。本地或某些轻量环境可能创建成功但不真正拦截流量。

## 四、真实后端场景示例

短链接服务有三个组件：

- `short-api`：对外提供 HTTP API。
- `postgres`：存储短链接。
- `cleanup-job`：定时清理过期数据。

合理访问关系是：

```text
Ingress -> short-api -> postgres
cleanup-job -> postgres
```

不合理关系是：

```text
任意 Pod -> postgres
```

如果清理任务也要访问数据库，可以给 CronJob 的 Pod 加标签：

```yaml
metadata:
  labels:
    app: short-link-cleaner
```

然后在 NetworkPolicy 中加入：

```yaml
from:
  - podSelector:
      matchLabels:
        app: short-api
  - podSelector:
      matchLabels:
        app: short-link-cleaner
```

## 五、注意点

- 不要一开始就在生产环境套一组没验证过的默认拒绝策略。
- 先画清服务调用关系，再写策略。
- 数据库、消息队列、管理后台优先做访问限制。
- 策略变更要可回滚，避免把核心服务之间的流量切断。
- NetworkPolicy 不替代应用鉴权，接口权限仍然要在业务层校验。

## 六、常见错误

- 错误：以为创建 NetworkPolicy 后一定生效。
  需要确认网络插件支持 NetworkPolicy。
- 错误：只写了入站规则，忘了 DNS 或出站访问。
  如果做默认拒绝，应用可能解析不了 Service 名称。
- 错误：用 IP 写死访问规则。
  Pod IP 会变，真实项目更应该用标签选择。

## 七、本节达标标准

- 能解释 RBAC 和 NetworkPolicy 的区别。
- 能写出只允许指定服务访问数据库的策略。
- 能说明 NetworkPolicy 对网络插件的依赖。
- 能在上线策略前先梳理服务调用关系。

