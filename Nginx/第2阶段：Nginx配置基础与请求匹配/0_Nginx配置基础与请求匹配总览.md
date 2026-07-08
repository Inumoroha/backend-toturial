# 0. Nginx 配置基础与请求匹配总览

本阶段目标：理解 Nginx 配置文件的层级结构，以及一个请求进入 Nginx 后如何匹配到具体 `server` 和 `location`。

第一阶段你已经能启动 Nginx、修改配置、查看日志。第二阶段要解决的问题是：

```text
为什么这个请求会进入这个 server？
为什么这个 URI 会匹配这个 location？
为什么配置写在这里生效，写在那里报错？
```

---

## 一、本阶段核心概念

你需要掌握：

- 配置上下文：`main`、`events`、`http`、`server`、`location`。
- `include` 如何把多个配置文件合成最终配置。
- `listen` 与端口。
- `server_name` 与 Host。
- `location` 匹配规则。
- `return` 直接返回。
- `rewrite` 路径改写。
- 常用变量：`$host`、`$uri`、`$request_uri`、`$remote_addr`。

---

## 二、请求匹配的基本过程

一个请求进入 Nginx，大致经历：

```text
收到 TCP 连接
-> 读取 HTTP 请求
-> 根据端口和 Host 选择 server
-> 根据 URI 选择 location
-> 执行 location 中的处理逻辑
-> 写 access log
```

例如：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/users
```

Nginx 会先找：

```nginx
server {
    listen 80;
    server_name api.local;
}
```

再在这个 server 里找：

```nginx
location /api/ {
}
```

---

## 三、本阶段文档顺序

建议按下面顺序学习：

1. `1_理解配置上下文与include机制.md`
2. `2_理解server_name与虚拟主机.md`
3. `3_location匹配规则详解.md`
4. `4_return_rewrite与常用变量.md`

---

## 四、为什么这个阶段很重要

后面所有能力都依赖这个阶段：

- 静态资源依赖 location。
- Go 反向代理依赖 location。
- HTTPS 依赖 server。
- 多域名部署依赖 server_name。
- 限流和缓存通常按 location 配置。
- 502/404/403 排障经常要判断请求到底进入了哪个 location。

如果你 location 匹配规则不清楚，后面会出现大量“配置明明写了但不生效”的问题。

---

## 五、本阶段验收

完成后你应该能：

1. 画出 Nginx 配置层级。
2. 解释 `server` 必须写在哪个上下文。
3. 用 `curl -H "Host: ..."` 测试虚拟主机。
4. 写出 `/`、`/api/`、`= /health` 三种 location。
5. 解释 `$uri` 和 `$request_uri` 的区别。

