# 第四阶段：软件包、进程与服务管理教程

本阶段目标：学会安装、升级、卸载软件；查看和控制进程；理解前台、后台任务；使用 `systemctl` 管理服务；使用 `journalctl` 查看系统日志。

建议学习时间：1 到 2 周。  
建议学习方式：按文件顺序学习，每节先看命令顺序，再跟着练习。  
练习目录：`~/linux-practice/stage-04`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-package-management.md` | 软件包管理 | `apt`、`dpkg`、`dnf`、`yum` |
| 2 | `02-process-view.md` | 查看进程 | `ps`、`top`、`htop`、`pgrep`、`pidof` |
| 3 | `03-process-control.md` | 控制进程 | `kill`、`killall`、信号 |
| 4 | `04-foreground-background.md` | 前台、后台与持久运行 | `&`、`jobs`、`fg`、`bg`、`nohup` |
| 5 | `05-systemd-services.md` | systemd 服务管理 | `systemctl` |
| 6 | `06-journalctl-logs.md` | 系统日志查看 | `journalctl` |
| 7 | `07-stage-practice.md` | 第四阶段综合练习 | 安装 Nginx、查看进程、管理服务、排查日志 |

---

## 第四阶段命令学习总顺序

建议按下面顺序学习：

```text
1. apt update
2. apt search
3. apt show
4. apt install
5. apt remove
6. dpkg -l
7. ps
8. top
9. htop
10. pgrep
11. pidof
12. kill
13. killall
14. jobs
15. fg
16. bg
17. nohup
18. systemctl status
19. systemctl start
20. systemctl stop
21. systemctl restart
22. systemctl enable
23. systemctl disable
24. journalctl
```

---

## 开始前准备

创建第四阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-04/{scripts,logs,tmp,output}
cd ~/linux-practice/stage-04
pwd
```

创建一个用于进程实验的脚本：

```bash
cat > scripts/loop.sh <<'EOF'
#!/usr/bin/env bash

while true; do
  echo "$(date '+%F %T') loop is running"
  sleep 5
done
EOF
```

添加执行权限：

```bash
chmod 755 scripts/loop.sh
```

---

## 本阶段安全提醒

本阶段会使用 `sudo`、安装软件、停止服务和杀进程。请注意：

- 优先在 WSL、虚拟机或测试环境练习。
- 不要随意停止你不认识的系统服务。
- 不要随意 `kill -9`，先尝试普通 `kill`。
- 不要在重要服务器上随意卸载软件。
- 操作服务前先执行 `systemctl status 服务名` 看清楚状态。

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能使用 `apt` 安装、查看、卸载软件。
- 能用 `ps`、`top`、`htop` 查看进程。
- 能找到指定进程的 PID。
- 能理解前台任务、后台任务和 `nohup`。
- 能用 `kill` 正常结束进程。
- 能使用 `systemctl` 查看、启动、停止、重启服务。
- 能设置服务开机自启或取消自启。
- 能用 `journalctl` 查看服务日志。
- 能完成一次 Nginx 安装、启动、验证和日志查看。

