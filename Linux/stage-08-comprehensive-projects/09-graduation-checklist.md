# 09：Linux 基础毕业验收清单

本节目标：用一份清单确认你是否真正完成了 Linux 基础阶段，并明确后续进阶方向。

---

## 1. 最终项目验收

进入目录：

```bash
cd ~/linux-practice/stage-08
```

检查最终结构：

```bash
tree -L 3
```

你应该至少有：

```text
linux-toolkit/scripts/backup.sh
linux-toolkit/scripts/check-service.sh
linux-toolkit/scripts/check-port.sh
linux-toolkit/scripts/top-ip.sh
linux-toolkit/scripts/log-summary.sh
reports/final-report.md
reports/troubleshooting-notes.md
```

---

## 2. 脚本验收

执行：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
for script in scripts/*.sh; do
  echo "Checking $script"
  bash -n "$script"
done
```

运行核心脚本：

```bash
./scripts/backup.sh data
./scripts/top-ip.sh logs/access.log 3
./scripts/log-summary.sh logs/access.log logs/app.log
./scripts/check-port.sh 80 || true
./scripts/check-service.sh nginx || true
```

---

## 3. Linux 基础能力清单

你应该能独立完成：

- [ ] 使用命令行管理文件和目录。
- [ ] 理解绝对路径、相对路径、`~`、`.`、`..`。
- [ ] 使用 `grep`、`find`、`awk`、`sed` 处理文本。
- [ ] 使用管道组合多个命令。
- [ ] 设置文件权限、所有者和所属组。
- [ ] 使用 `sudo`，并知道它的风险。
- [ ] 安装、卸载、查询软件包。
- [ ] 查看和结束进程。
- [ ] 管理 systemd 服务。
- [ ] 查看系统和服务日志。
- [ ] 编写简单 Shell 脚本。
- [ ] 使用 SSH 登录远程 Linux。
- [ ] 使用 `scp` 或 `rsync` 传输文件。
- [ ] 查看磁盘空间、内存、CPU 和 IO。
- [ ] 排查权限、端口、服务、磁盘、DNS 常见问题。

---

## 4. 常用排查模板

### 服务访问不了

```bash
systemctl status 服务名
journalctl -u 服务名 -n 50 --no-pager
ss -tuln
curl -I http://localhost
```

### 磁盘空间不足

```bash
df -h
df -i
du -h --max-depth=1 目标目录
du -ah 目标目录 | sort -rh | head -n 20
sudo lsof | grep deleted
```

### 网络不通

```bash
ip addr
ip route
ping -c 4 8.8.8.8
ping -c 4 example.com
dig example.com
curl -I https://example.com
```

### 权限错误

```bash
whoami
id
pwd
ls -ld 目录
ls -l 文件
stat 文件
```

---

## 5. 后续进阶方向

如果你想走运维 / SRE：

```text
Nginx 深入 -> Docker -> Prometheus/Grafana -> CI/CD -> Kubernetes
```

如果你想走后端开发：

```text
Git -> Linux 开发环境 -> Makefile -> Docker -> 服务部署 -> 性能排查
```

如果你想走安全方向：

```text
Linux 加固 -> 日志分析 -> 网络扫描 -> 权限审计 -> Web 安全基础
```

如果你想走嵌入式 Linux：

```text
C 语言 -> Makefile -> 交叉编译 -> BusyBox -> Linux 内核基础 -> 驱动入门
```

---

## 6. 最终复盘问题

请写入你的学习笔记：

```text
1. 我最熟悉的 20 个 Linux 命令是什么？
2. 我最容易忘的 10 个命令是什么？
3. 我独立解决过哪些错误？
4. 我现在能写哪些 Shell 脚本？
5. 我能否独立部署并排查一个 Nginx 服务？
6. 下一步我要选择哪个方向深入？
```

---

## 7. 毕业标准

当你能不看教程完成下面任务，就说明 Linux 基础阶段已经完成：

```text
1. 创建一个项目目录。
2. 编写至少 3 个可用 Shell 脚本。
3. 分析一份日志并生成报告。
4. 部署并验证一个 Nginx 服务。
5. 模拟并排查 5 个常见故障。
6. 写出完整的 final-report.md。
```

完成这些后，你就可以从“学习 Linux 命令”进入“用 Linux 解决真实问题”的阶段了。

