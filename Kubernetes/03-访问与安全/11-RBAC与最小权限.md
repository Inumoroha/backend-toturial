# 11. RBAC 与最小权限

本节目标：学完后你能理解 Kubernetes 权限控制的基本结构，并为应用或部署账号分配最小权限。

## 简短引入

后端系统里我们不会让普通用户拥有管理员权限。Kubernetes 也一样。RBAC 用来控制谁能对哪些资源做哪些操作。

## 一、为什么需要它

真实项目中有不同身份：

- 开发人员查看测试环境日志。
- CI/CD 账号发布应用。
- 应用 Pod 读取 ConfigMap。
- 运维人员管理集群资源。

如果所有人都用管理员权限，误删资源、泄露 Secret 的风险会非常高。

```text
权限默认应该从小开始，需要什么再加什么，而不是先给 cluster-admin。
```

## 二、基本用法

创建 ServiceAccount：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: short-api
  namespace: backend-dev
```

创建 Role，只允许读取 ConfigMap：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: backend-dev
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
```

绑定权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: short-api-config-reader
  namespace: backend-dev
subjects:
  - kind: ServiceAccount
    name: short-api
    namespace: backend-dev
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

Deployment 使用 ServiceAccount：

```yaml
spec:
  template:
    spec:
      serviceAccountName: short-api
```

## 三、关键参数/语法/代码结构

- ServiceAccount：Pod 使用的身份。
- Role：namespace 内权限规则。
- RoleBinding：把 Role 绑定给用户、组或 ServiceAccount。
- ClusterRole：集群级权限，影响范围更大。
- ClusterRoleBinding：集群级绑定，生产中更要谨慎。

常用动词：

- `get`：读取单个资源。
- `list`：列出资源。
- `watch`：监听资源变化。
- `create`、`update`、`patch`、`delete`：修改类权限，风险更高。

## 四、真实后端场景示例

短链接服务如果只是普通 HTTP API，通常不需要访问 Kubernetes API。它甚至可以使用默认受限 ServiceAccount。

但 CI/CD 部署账号需要更新 Deployment 镜像，可以只给某个 namespace 中 Deployment 的更新权限，而不是集群管理员：

```yaml
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch", "update"]
```

真实团队里，应用账号和部署账号应该分开。应用运行时不应该拥有发布权限。

## 五、注意点

- 不要为了省事给 `cluster-admin`。
- 不要让业务 Pod 拥有读取所有 Secret 的权限。
- CI/CD 凭证要能轮换，离职和泄露时要能快速撤销。
- RBAC 是生产安全的一部分，不是全部，还需要网络、镜像、密钥等控制。

## 六、常见错误

- 错误：应用 Pod 使用默认 ServiceAccount，且默认账号权限过大。
  应该为应用创建专用 ServiceAccount。
- 错误：RoleBinding 绑错 namespace。
  Role 和 RoleBinding 都是 namespace 资源时，要确认在同一 namespace。
- 错误：用 ClusterRole 解决所有权限问题。
  能用 Role 就不要扩大到集群级。

## 七、本节达标标准

- 能解释 ServiceAccount、Role、RoleBinding 的关系。
- 能创建一个最小读取权限示例。
- 能区分 Role 和 ClusterRole 的适用场景。
- 能说明为什么应用运行权限和部署权限要分开。

