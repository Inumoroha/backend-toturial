# 07：ufw 防火墙基础

本节目标：理解防火墙规则，学会查看、允许、拒绝端口，并安全地管理 SSH 访问。

本节命令顺序：

```text
1. ufw status
2. ufw allow
3. ufw deny
4. ufw delete
5. ufw enable
6. ufw disable
7. ufw status numbered
```

本节建议在虚拟机中练习。云服务器还可能有“安全组”，它和系统内的 `ufw` 是两层不同的防火墙。

---

## 1. 防火墙是什么

防火墙用来控制哪些网络连接可以进入或离开主机。

常见场景：

```text
允许 SSH 22 端口
允许 HTTP 80 端口
允许 HTTPS 443 端口
拒绝数据库端口对公网开放
```

Ubuntu 常用简单防火墙工具：

```bash
ufw
```

---

## 2. 查看状态

执行：

```bash
sudo ufw status
```

更详细：

```bash
sudo ufw status verbose
```

带编号：

```bash
sudo ufw status numbered
```

---

## 3. 允许端口

允许 SSH：

```bash
sudo ufw allow 22/tcp
```

也可以使用服务名：

```bash
sudo ufw allow ssh
```

允许 HTTP 和 HTTPS：

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

允许 Nginx Full：

```bash
sudo ufw allow "Nginx Full"
```

查看应用配置：

```bash
sudo ufw app list
```

---

## 4. 拒绝端口

拒绝某个端口：

```bash
sudo ufw deny 3306/tcp
```

这常用于避免数据库端口暴露到外部。

---

## 5. 启用和禁用 ufw

启用前，先允许 SSH，避免远程连接被断开：

```bash
sudo ufw allow ssh
```

启用：

```bash
sudo ufw enable
```

查看：

```bash
sudo ufw status verbose
```

禁用：

```bash
sudo ufw disable
```

安全提醒：

```text
远程服务器上启用防火墙前，必须确认 SSH 端口已允许。
```

---

## 6. 删除规则

查看编号：

```bash
sudo ufw status numbered
```

按编号删除：

```bash
sudo ufw delete 1
```

也可以按规则删除：

```bash
sudo ufw delete allow 80/tcp
```

---

## 7. 常见规则组合

Web 服务器常见：

```bash
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

只允许本机访问数据库：

```text
数据库监听 127.0.0.1，比只靠防火墙更安全。
```

---

## 8. 本节小结

必须记住：

```bash
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 3306/tcp
sudo ufw delete 1
sudo ufw enable
sudo ufw disable
```

完成后进入下一节：`08-stage-practice.md`。

