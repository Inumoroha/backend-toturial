# 10：ApplicationSet 与 App of Apps

## 1. 本节目标

当服务和环境变多，一个个手写 Application 会变繁琐。

这一节了解两种扩展模式：

- App of Apps。
- ApplicationSet。

本节先建立概念，不要求立刻落地。

## 2. App of Apps

App of Apps 是让一个根 Application 管理多个子 Application。

结构：

```text
root application
  -> app-a staging
  -> app-a production
  -> app-b staging
  -> app-b production
```

根 Application 指向一个目录，目录里放多个 Application YAML。

优点：

- 所有应用可以从一个入口管理。
- bootstrap 新集群方便。

缺点：

- 层级增加。
- 初学时容易绕。

## 3. App of Apps 目录示例

```text
go-cicd-platform-deploy/
  applications/
    go-cicd-lab-staging.yaml
    go-cicd-lab-production.yaml
    another-service-staging.yaml
  root-application.yaml
```

root application 指向：

```text
applications/
```

Argo CD 先同步 root，再创建子 Applications。

## 4. ApplicationSet

ApplicationSet 可以根据模板和生成器批量生成 Applications。

适合：

- 多集群。
- 多环境。
- 多服务。
- 目录驱动应用生成。

例如一个服务目录生成一个 Application。

## 5. ApplicationSet 示例概念

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: go-services
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: go-cicd-lab
            env: staging
          - name: go-cicd-lab
            env: production
  template:
    metadata:
      name: "{{name}}-{{env}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/your-name/go-cicd-lab-deploy.git
        targetRevision: main
        path: charts/{{name}}
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{name}}-{{env}}"
```

实际使用时还要配置 Helm values。

## 6. 初学怎么选

单服务单环境：

```text
直接 Application。
```

单服务多环境：

```text
两个 Application：staging、production。
```

多服务多环境：

```text
考虑 App of Apps 或 ApplicationSet。
```

不要太早引入 ApplicationSet。

先把基本 Application 彻底学会。

## 7. 小练习

回答：

1. 如果有 1 个服务、2 个环境，你会用几个 Application？
2. 如果有 20 个服务、3 个环境，手写 Application 会有什么问题？
3. App of Apps 和 ApplicationSet 分别解决什么问题？

## 8. 本节小结

你现在应该理解：

- App of Apps 用根 Application 管理子 Application。
- ApplicationSet 用模板批量生成 Application。
- 初学阶段先用普通 Application。
- 多服务多环境后再考虑扩展模式。

