# 08：第六阶段综合练习

本节目标：完成一次完整的网络排查和远程管理练习，串起 IP、路由、DNS、HTTP、端口、SSH、文件传输和防火墙。

如果你只有 WSL，也可以完成大部分本机网络命令；SSH 和防火墙部分更推荐在虚拟机或云服务器中完成。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. ip addr
2. hostname -I
3. ip route
4. ping
5. dig
6. nslookup
7. traceroute
8. curl
9. wget
10. python3 -m http.server
11. ss
12. lsof
13. ssh
14. ssh-keygen
15. ssh-copy-id
16. scp
17. rsync
18. ufw
```

---

## 2. 初始化练习目录

```bash
mkdir -p ~/linux-practice/stage-06/final-lab/{data,downloads,logs,output,site}
cd ~/linux-practice/stage-06/final-lab
pwd
```

创建本地网页：

```bash
cat > site/index.html <<'EOF'
<!doctype html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Linux Network Lab</title>
  </head>
  <body>
    <h1>Linux Network Lab</h1>
    <p>Hello from local HTTP server.</p>
  </body>
</html>
EOF
```

创建传输测试文件：

```bash
echo "network transfer test" > data/message.txt
```

---

## 3. 本机网络信息检查

查看网卡和 IP：

```bash
ip addr
hostname -I
```

查看路由：

```bash
ip route
ip route get 8.8.8.8
```

写入报告：

```bash
echo "# Stage 06 Network Report" > output/report.md
echo "## IP" >> output/report.md
hostname -I >> output/report.md
echo "## Route" >> output/report.md
ip route >> output/report.md
```

---

## 4. 连通性和 DNS 检查

测试公网 IP：

```bash
ping -c 4 8.8.8.8
```

测试域名：

```bash
ping -c 4 example.com
```

DNS 查询：

```bash
dig +short example.com
nslookup example.com
```

路径追踪：

```bash
traceroute example.com
```

追加报告：

```bash
echo "## DNS example.com" >> output/report.md
dig +short example.com >> output/report.md
```

---

## 5. HTTP 请求测试

查看响应头：

```bash
curl -I https://example.com
```

下载网页：

```bash
curl -o downloads/example.html https://example.com
wget https://example.com -O downloads/example-wget.html
```

查看文件：

```bash
ls -lh downloads
```

---

## 6. 启动本地 HTTP 服务并检查端口

进入站点目录：

```bash
cd ~/linux-practice/stage-06/final-lab/site
```

启动 HTTP 服务：

```bash
python3 -m http.server 8080
```

另开一个终端，执行：

```bash
ss -tunlp | grep 8080
curl -I http://localhost:8080
curl http://localhost:8080
```

查看端口占用：

```bash
sudo lsof -i :8080
```

回到运行 HTTP 服务的终端，按：

```text
Ctrl + C
```

停止服务。

---

## 7. SSH 登录练习

准备一台远程 Linux，记录：

```text
远程用户名：__________
远程 IP：_____________
SSH 端口：____________
```

测试连接：

```bash
ssh user@host
```

如果端口不是 22：

```bash
ssh -p 2222 user@host
```

登录后执行：

```bash
whoami
hostname -I
pwd
exit
```

---

## 8. SSH 密钥练习

如果还没有密钥：

```bash
ssh-keygen -t ed25519 -C "linux-stage-06"
```

复制公钥：

```bash
ssh-copy-id user@host
```

再次连接：

```bash
ssh user@host
```

如果你使用非默认端口：

```bash
ssh-copy-id -p 2222 user@host
ssh -p 2222 user@host
```

---

## 9. scp 文件复制练习

从本地复制到远程：

```bash
cd ~/linux-practice/stage-06/final-lab
scp data/message.txt user@host:/tmp/
```

从远程复制回本地：

```bash
scp user@host:/tmp/message.txt downloads/message-from-remote.txt
```

检查：

```bash
cat downloads/message-from-remote.txt
```

---

## 10. rsync 目录同步练习

同步站点目录到远程：

```bash
rsync -av site/ user@host:/tmp/linux-network-site/
```

预演同步：

```bash
rsync -av --dry-run site/ user@host:/tmp/linux-network-site/
```

如果你确认要让远程目录和本地完全一致：

```bash
rsync -av --delete --dry-run site/ user@host:/tmp/linux-network-site/
```

确认无误后再去掉 `--dry-run`。

---

## 11. 防火墙基础练习

只建议在虚拟机或测试服务器中练习。

查看状态：

```bash
sudo ufw status
```

允许 SSH：

```bash
sudo ufw allow ssh
```

允许 HTTP：

```bash
sudo ufw allow 80/tcp
```

查看带编号规则：

```bash
sudo ufw status numbered
```

启用前再次确认 SSH 已允许：

```bash
sudo ufw status
```

启用：

```bash
sudo ufw enable
```

如需关闭：

```bash
sudo ufw disable
```

---

## 12. 网络问题排查模板

当一个服务访问不了时，按顺序执行：

```bash
ip addr
ip route
ping -c 4 目标IP
dig 目标域名
curl -I http://目标地址
ss -tunlp
sudo ufw status
```

如果是 SSH 访问不了：

```bash
ping -c 4 服务器IP
ssh -v user@host
sudo ss -tunlp | grep :22
sudo ufw status
```

`ssh -v` 会显示详细连接过程，排查认证和连接问题很有用。

---

## 13. 生成练习报告

追加 HTTP 检查：

```bash
echo "## HTTP example.com" >> output/report.md
curl -I https://example.com >> output/report.md
```

追加端口检查：

```bash
echo "## Listening Ports" >> output/report.md
ss -tuln >> output/report.md
```

查看报告：

```bash
cat output/report.md
```

---

## 14. 阶段验收题

请你不看答案，回答下面问题：

1. `ip addr` 和 `hostname -I` 有什么区别？
2. `ip route` 中的 `default via` 表示什么？
3. IP 能 ping 通但域名 ping 不通，优先怀疑什么？
4. `dig +short example.com` 是做什么的？
5. `curl -I` 和 `curl -v` 有什么区别？
6. `ss -tunlp` 各参数是什么意思？
7. `127.0.0.1:8080` 和 `0.0.0.0:8080` 有什么区别？
8. SSH 默认端口是多少？
9. SSH 私钥和公钥分别放在哪里？
10. `scp -P` 和 `ssh -p` 为什么大小写不同？
11. `rsync` 源目录后面带 `/` 和不带 `/` 有什么区别？
12. 云服务器安全组和系统内 `ufw` 是不是同一层？

---

## 15. 阶段完成标准

当你能独立完成下面任务，就可以进入第七阶段：

- [ ] 查看本机 IP、网卡和默认路由。
- [ ] 使用 `ping` 测试 IP 和域名连通性。
- [ ] 使用 `dig` 或 `nslookup` 查询 DNS。
- [ ] 使用 `curl` 查看 HTTP 响应头。
- [ ] 使用 `wget` 下载文件。
- [ ] 使用 `ss` 查看监听端口。
- [ ] 使用 `lsof` 查看端口占用。
- [ ] 使用 SSH 登录远程 Linux。
- [ ] 创建 SSH 密钥并完成密钥登录。
- [ ] 使用 `scp` 复制文件。
- [ ] 使用 `rsync` 同步目录。
- [ ] 使用 `ufw` 查看和添加基础防火墙规则。

---

## 16. 建议写入学习笔记

记录下面内容：

```text
1. 我的网络排查命令清单
2. IP、网关、DNS、端口分别解决什么问题
3. curl 常用参数
4. ss 输出中 LISTEN、ESTAB 的含义
5. SSH 密钥登录配置过程
6. scp 和 rsync 的区别
7. 一次完整的网络问题排查流程
```

完成这些记录后，第六阶段就真正学完了。

