# 0. Go 服务 Nginx 网关项目总览

本阶段目标：把前面学过的 Nginx 配置、Go 服务、多实例、HTTPS、日志、限流、静态资源和排障串起来，完成一个可运行的小型后端网关项目。

前面阶段学到的是单点能力：

```text
会启动 Nginx。
会写 server 和 location。
会部署静态页面。
会 proxy_pass 到 Go。
会配置 upstream。
会配置 HTTPS。
会看 access.log 和 error.log。
会配置限流。
```

第十阶段要做的是把这些能力放到一个真实项目里。

---

## 一、项目要做什么

最终架构：

```text
浏览器 / curl
  |
  | HTTPS
  v
Nginx
  |-- /              -> 静态首页
  |-- /assets/       -> 静态资源，带缓存
  |-- /health        -> Nginx 直接返回
  |-- /api/          -> Go API upstream
  |-- /api/upload    -> 上传接口，单独 body size
  |
  v
Go API 三实例
  |-- 127.0.0.1:8080
  |-- 127.0.0.1:8081
  |-- 127.0.0.1:8082
```

---

## 二、项目最终能力

完成本阶段后，你应该能独立完成：

- 写一个 Go API 服务。
- 启动三个不同端口的 Go 实例。
- 用 Nginx `upstream` 做负载均衡。
- 用 Nginx 托管静态页面。
- 配置 HTTPS。
- HTTP 自动跳转 HTTPS。
- 传递真实 IP、协议和请求 ID。
- 自定义 access log。
- 使用日志定位 502 和 504。
- 给 `/api/` 加限流。
- 给 `/api/upload` 配置上传大小。
- 用 systemd 或 Docker Compose 部署。

---

## 三、推荐项目结构

```text
nginx-gateway-demo/
  app/
    go.mod
    main.go
  static/
    index.html
    assets/
      style.css
  nginx/
    nginx.conf
    conf.d/
      00-log-format.conf
      10-upstream.conf
      20-app.conf
  scripts/
    curl-check.sh
  README.md
```

这个结构的好处：

- `app` 放 Go 代码。
- `static` 放静态页面。
- `nginx` 放 Nginx 配置。
- `conf.d` 按职责拆分。
- `scripts` 放验证命令。

---

## 四、本阶段文档安排

建议按下面顺序学习：

1. `0_Go服务Nginx网关项目总览.md`
2. `1_需求分析与接口设计.md`
3. `2_Go服务实现.md`
4. `3_静态页面与目录结构.md`
5. `4_Nginx配置拆分.md`
6. `5_HTTPS限流日志与请求ID.md`
7. `6_502与504故障演练.md`
8. `7_DockerCompose部署.md`
9. `8_综合验收清单.md`

---

## 五、项目最小闭环

先不要一口气做完所有功能。最小闭环是：

```text
启动 Go :8080
-> Nginx proxy_pass 到 Go
-> curl /api/ping 成功
-> access.log 有记录
```

然后逐步加：

```text
三实例 upstream
-> 静态页面
-> HTTPS
-> 请求 ID
-> 限流
-> 上传
-> 故障演练
```

---

## 六、为什么这个项目适合练习 Nginx

这个项目很小，但覆盖了 Nginx 在 Go 后端中的核心用法：

- 对外入口。
- 静态资源。
- API 代理。
- 多实例负载均衡。
- HTTPS。
- 日志。
- 限流。
- 排障。

它不会把复杂业务逻辑混进来，重点放在 Nginx 和 Go 服务之间的协作。

---

## 七、本阶段完成标准

如果你能做到下面这些，就说明 Nginx 基础已经比较扎实：

1. 能解释请求从浏览器到 Go handler 的完整链路。
2. 能解释每个 `server` 和 `location` 的作用。
3. 能解释 `proxy_pass` 到底转发到了哪里。
4. 能通过日志找到实际 upstream。
5. 能复现并解释 502。
6. 能复现并解释 504。
7. 能说明哪些逻辑放 Nginx，哪些逻辑放 Go。

