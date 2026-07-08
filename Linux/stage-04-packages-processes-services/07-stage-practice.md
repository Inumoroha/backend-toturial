# 07：第四阶段综合练习

本节目标：综合使用软件包、进程、后台任务、服务和日志命令，完成一次接近真实工作的系统管理练习。

请在 WSL、虚拟机或测试环境中练习。涉及 `systemctl` 的部分，如果 WSL 不支持 systemd，可以在虚拟机 Ubuntu Server 中完成。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. cd
2. apt update
3. apt install
4. dpkg -l
5. ps
6. pgrep
7. top
8. htop
9. kill
10. jobs
11. fg
12. bg
13. nohup
14. systemctl status
15. systemctl start
16. systemctl restart
17. systemctl enable
18. systemctl is-active
19. journalctl
20. curl
```

---

## 2. 初始化练习目录

```bash
mkdir -p ~/linux-practice/stage-04/final-lab/{scripts,logs,output}
cd ~/linux-practice/stage-04/final-lab
pwd
```

创建循环脚本：

```bash
cat > scripts/worker.sh <<'EOF'
#!/usr/bin/env bash

while true; do
  echo "$(date '+%F %T') worker running"
  sleep 3
done
EOF
```

添加执行权限：

```bash
chmod 755 scripts/worker.sh
```

---

## 3. 软件包管理练习

更新软件源：

```bash
sudo apt update
```

安装工具：

```bash
sudo apt install tree htop curl
```

确认安装：

```bash
dpkg -l | grep -E "tree|htop|curl"
```

查看命令路径：

```bash
which tree
which htop
which curl
```

---

## 4. 进程查看练习

启动后台脚本：

```bash
./scripts/worker.sh > logs/worker.log 2>&1 &
```

查看当前 Shell 后台任务：

```bash
jobs
```

查找进程：

```bash
pgrep -af worker.sh
ps aux | grep worker.sh | grep -v grep
```

查看日志：

```bash
tail -n 5 logs/worker.log
```

使用 `top` 或 `htop` 观察进程：

```bash
top
```

在 `top` 中按 `q` 退出。

---

## 5. 进程控制练习

找到 PID：

```bash
pgrep -af worker.sh
```

正常结束进程。把 `PID` 替换成真实数字：

```bash
kill PID
```

确认：

```bash
pgrep -af worker.sh
```

如果没有输出，说明已经结束。

---

## 6. 前台和后台练习

启动一个前台任务：

```bash
sleep 300
```

按：

```text
Ctrl + Z
```

查看任务：

```bash
jobs
```

让任务后台继续：

```bash
bg
```

调回前台：

```bash
fg
```

按 `Ctrl + C` 结束。

---

## 7. nohup 持久运行练习

启动：

```bash
nohup ./scripts/worker.sh > logs/nohup-worker.log 2>&1 &
```

查看进程：

```bash
pgrep -af worker.sh
```

查看日志：

```bash
tail -f logs/nohup-worker.log
```

按 `Ctrl + C` 停止查看日志。

结束脚本：

```bash
pkill -f worker.sh
```

确认：

```bash
pgrep -af worker.sh
```

---

## 8. Nginx 服务练习

安装 Nginx：

```bash
sudo apt install nginx
```

查看服务状态：

```bash
systemctl status nginx
```

启动服务：

```bash
sudo systemctl start nginx
```

确认是否运行：

```bash
systemctl is-active nginx
```

访问本机：

```bash
curl -I http://localhost
```

重启服务：

```bash
sudo systemctl restart nginx
```

设置开机自启：

```bash
sudo systemctl enable nginx
```

查看是否自启：

```bash
systemctl is-enabled nginx
```

---

## 9. Nginx 进程和端口观察

查看 Nginx 进程：

```bash
ps aux | grep nginx | grep -v grep
pgrep -af nginx
```

如果系统有 `ss`，查看监听端口：

```bash
ss -tunlp | grep nginx
```

访问几次：

```bash
curl http://localhost
curl -I http://localhost
```

---

## 10. 服务日志查看

查看 Nginx 最近日志：

```bash
journalctl -u nginx -n 50
```

查看最近 1 小时日志：

```bash
journalctl -u nginx --since "1 hour ago"
```

实时查看日志：

```bash
journalctl -u nginx -f
```

按 `Ctrl + C` 退出。

查看错误日志：

```bash
journalctl -u nginx -p err
```

---

## 11. 生成练习报告

创建报告：

```bash
echo "# Stage 04 Report" > output/report.md
```

写入软件包信息：

```bash
echo "## Installed Packages" >> output/report.md
dpkg -l | grep -E "tree|htop|curl|nginx" >> output/report.md
```

写入服务状态：

```bash
echo "## Nginx Status" >> output/report.md
systemctl is-active nginx >> output/report.md
systemctl is-enabled nginx >> output/report.md
```

写入进程信息：

```bash
echo "## Nginx Processes" >> output/report.md
pgrep -af nginx >> output/report.md
```

查看报告：

```bash
cat output/report.md
```

---

## 12. 阶段验收题

请你不看答案，回答下面问题：

1. `apt update` 和 `apt upgrade` 有什么区别？
2. `apt install` 和 `apt remove` 分别做什么？
3. PID 是什么？
4. `ps aux` 和 `top` 有什么区别？
5. `pgrep -af nginx` 是做什么的？
6. `kill PID` 默认发送什么信号？
7. 为什么不要一开始就用 `kill -9`？
8. `command &` 的作用是什么？
9. `jobs` 和 `ps` 的查看范围有什么区别？
10. `nohup command > log 2>&1 &` 适合什么场景？
11. `systemctl start` 和 `systemctl enable` 有什么区别？
12. `journalctl -u nginx -f` 是做什么的？

---

## 13. 阶段完成标准

当你能独立完成下面任务，就可以进入第五阶段：

- [ ] 使用 `apt` 安装和卸载软件。
- [ ] 使用 `dpkg -l` 查询已安装软件。
- [ ] 使用 `ps`、`top`、`htop` 查看进程。
- [ ] 使用 `pgrep` 找到指定进程 PID。
- [ ] 使用 `kill` 正常结束进程。
- [ ] 使用 `jobs`、`fg`、`bg` 管理前后台任务。
- [ ] 使用 `nohup` 让脚本在后台持续运行。
- [ ] 使用 `systemctl` 启动、停止、重启服务。
- [ ] 使用 `systemctl enable` 和 `disable` 管理开机自启。
- [ ] 使用 `journalctl` 查看服务日志。
- [ ] 完成 Nginx 安装、启动、访问和日志查看。

---

## 14. 建议写入学习笔记

记录下面内容：

```text
1. apt 常用命令清单
2. ps、top、htop、pgrep 的区别
3. kill 常见信号和使用顺序
4. 前台、后台、nohup 的区别
5. systemctl start 和 enable 的区别
6. journalctl 查看服务日志的常用命令
7. 一次 Nginx 服务管理实验记录
```

完成这些记录后，第四阶段就真正学完了。

