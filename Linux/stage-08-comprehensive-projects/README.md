# 第八阶段：综合实战项目教程

本阶段目标：把前 7 个阶段学过的 Linux 基础能力串起来，完成一组接近真实工作的综合项目：个人 Linux 工具箱、备份与恢复、服务健康检查、日志分析、Nginx 部署、故障模拟与排查。

建议学习时间：1 到 2 周。  
建议学习方式：按文件顺序学习，每节先看“本节命令顺序”，再完成项目任务。  
练习目录：`~/linux-practice/stage-08`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点 |
| --- | --- | --- | --- |
| 1 | `01-final-lab-setup.md` | 综合实验准备 | 建目录、准备日志和数据 |
| 2 | `02-linux-toolkit-structure.md` | 个人 Linux 工具箱结构 | README、目录结构、脚本规范 |
| 3 | `03-backup-and-restore.md` | 备份与恢复项目 | `tar`、校验、恢复测试 |
| 4 | `04-service-health-check.md` | 服务健康检查项目 | `systemctl`、`ss`、`curl` |
| 5 | `05-log-analysis-project.md` | 日志分析项目 | `grep`、`awk`、`sort`、`uniq` |
| 6 | `06-nginx-deployment.md` | Nginx 部署项目 | `apt`、`systemctl`、`curl`、日志 |
| 7 | `07-troubleshooting-labs.md` | 故障模拟与排查 | 权限、端口、磁盘、服务、DNS |
| 8 | `08-final-report.md` | 生成最终报告 | 报告模板、验收记录 |
| 9 | `09-graduation-checklist.md` | 毕业验收清单 | 阶段复盘、后续方向 |

---

## 第八阶段命令学习总顺序

本阶段不是学习全新命令，而是综合使用前面命令。建议按下面顺序复用：

```text
1. mkdir
2. touch
3. chmod
4. cat
5. tee
6. tar
7. sha256sum
8. grep
9. awk
10. sort
11. uniq
12. head
13. tail
14. systemctl
15. journalctl
16. ss
17. curl
18. df
19. du
20. find
21. lsof
22. ping
23. dig
```

---

## 开始前准备

创建第八阶段目录：

```bash
mkdir -p ~/linux-practice/stage-08
cd ~/linux-practice/stage-08
pwd
```

建议你在 WSL、虚拟机或测试服务器中完成。涉及 `systemctl` 和 Nginx 的部分，如果 WSL 不支持 systemd，请在虚拟机 Ubuntu Server 中练习。

---

## 本阶段最终产物

完成后，你应该拥有：

```text
stage-08/
  linux-toolkit/
    scripts/
      backup.sh
      check-service.sh
      check-port.sh
      top-ip.sh
      log-summary.sh
    data/
    logs/
    backup/
    output/
    README.md
  reports/
    final-report.md
    troubleshooting-notes.md
```

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 独立创建一个清晰的 Linux 工具箱项目。
- 写出可执行脚本，并进行参数检查。
- 完成备份、校验、恢复测试。
- 检查服务状态、端口监听和 HTTP 响应。
- 分析访问日志，统计状态码和 IP。
- 部署并验证一个 Nginx 服务。
- 模拟并排查至少 5 类常见故障。
- 写出一份完整的 Linux 学习阶段报告。

