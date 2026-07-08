# 07：故障模拟与排查实验

本节目标：模拟常见 Linux 问题，并按固定流程排查：服务未启动、端口未监听、权限错误、磁盘空间异常、DNS 异常。

本节命令顺序：

```text
1. 观察现象
2. 查看状态
3. 查看日志
4. 验证假设
5. 修复问题
6. 记录复盘
```

---

## 1. 排查通用流程

每次排查都按这个流程：

```text
观察现象 -> 查看状态 -> 查看日志 -> 缩小范围 -> 验证假设 -> 修复问题 -> 记录复盘
```

建议记录到：

```bash
cd ~/linux-practice/stage-08
touch reports/troubleshooting-notes.md
```

---

## 2. 故障一：服务没有启动

模拟：

```bash
sudo systemctl stop nginx
```

观察：

```bash
curl -I http://localhost
```

排查：

```bash
systemctl status nginx
journalctl -u nginx -n 50 --no-pager
ss -tuln | grep ':80'
```

修复：

```bash
sudo systemctl start nginx
curl -I http://localhost
```

记录：

```bash
cat >> reports/troubleshooting-notes.md <<'EOF'
## Case 1: Nginx stopped

- Symptom: localhost HTTP request failed.
- Check: systemctl status nginx, journalctl, ss.
- Root cause: nginx service was stopped.
- Fix: sudo systemctl start nginx.
EOF
```

---

## 3. 故障二：文件权限错误

进入工具箱目录：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
```

模拟脚本不可执行：

```bash
chmod -x scripts/top-ip.sh
./scripts/top-ip.sh logs/access.log
```

排查：

```bash
ls -l scripts/top-ip.sh
file scripts/top-ip.sh
head -n 1 scripts/top-ip.sh
```

修复：

```bash
chmod +x scripts/top-ip.sh
./scripts/top-ip.sh logs/access.log
```

记录：

```bash
cat >> ../reports/troubleshooting-notes.md <<'EOF'

## Case 2: Script permission denied

- Symptom: script could not execute.
- Check: ls -l, file, shebang.
- Root cause: missing execute permission.
- Fix: chmod +x script.
EOF
```

---

## 4. 故障三：端口未监听

模拟：

```bash
sudo systemctl stop nginx
```

排查端口：

```bash
ss -tuln | grep ':80'
sudo lsof -i :80
systemctl status nginx
```

修复：

```bash
sudo systemctl start nginx
ss -tuln | grep ':80'
curl -I http://localhost
```

---

## 5. 故障四：磁盘空间异常

模拟一个大文件：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
truncate -s 100M tmp/big-cache.bin
```

排查：

```bash
df -h .
du -h --max-depth=1 .
du -ah . | sort -rh | head -n 10
```

确认后清理：

```bash
ls -lh tmp/big-cache.bin
rm -i tmp/big-cache.bin
du -sh .
```

记录：

```bash
cat >> ../reports/troubleshooting-notes.md <<'EOF'

## Case 3: Large cache file

- Symptom: project directory grew unexpectedly.
- Check: df, du, sort.
- Root cause: temporary large cache file.
- Fix: confirmed and removed tmp/big-cache.bin.
EOF
```

---

## 6. 故障五：DNS 或网络异常

观察：

```bash
ping -c 4 8.8.8.8
ping -c 4 example.com
dig example.com
curl -I https://example.com
```

判断：

| 现象 | 方向 |
| --- | --- |
| IP 不通 | 查 IP、路由、网关、防火墙 |
| IP 通但域名不通 | 查 DNS |
| 域名能解析但 HTTP 不通 | 查端口、服务、代理、防火墙 |

记录：

```bash
cat >> reports/troubleshooting-notes.md <<'EOF'

## Case 4: Network or DNS check

- Symptom: network access issue.
- Check: ping IP, ping domain, dig, curl.
- Root cause: fill in after real test.
- Fix: fill in after real test.
EOF
```

---

## 7. 故障排查记录模板

复制这个模板用于后续问题：

```markdown
## Case N: 问题标题

- 时间：
- 现象：
- 影响：
- 使用命令：
- 关键输出：
- 根因：
- 修复：
- 验证：
- 复盘：
```

---

## 8. 本节小结

必须掌握：

```bash
systemctl status nginx
journalctl -u nginx -n 50 --no-pager
ss -tuln | grep ':80'
sudo lsof -i :80
ls -l file
df -h .
du -ah . | sort -rh | head -n 10
ping -c 4 8.8.8.8
dig example.com
curl -I http://localhost
```

完成后进入下一节：`08-final-report.md`。

