# 06：journalctl 查看系统日志

本节目标：学会使用 `journalctl` 查看系统日志、服务日志、最近日志和实时日志。

本节命令顺序：

```text
1. journalctl
2. journalctl -n
3. journalctl -f
4. journalctl -u service
5. journalctl --since
6. journalctl --until
7. journalctl -p
8. journalctl -xe
```

说明：`journalctl` 依赖 systemd 日志系统。WSL 环境中可能不可用或日志较少，虚拟机/服务器环境更适合练习。

---

## 1. journalctl 是什么

`journalctl` 用来查看 systemd 收集的日志。

它能查看：

- 系统启动日志。
- 服务日志。
- 内核日志。
- 指定时间范围的日志。
- 指定优先级的日志。

---

## 2. 查看全部日志

执行：

```bash
journalctl
```

常用按键：

| 按键 | 作用 |
| --- | --- |
| `空格` | 向下翻页 |
| `b` | 向上翻页 |
| `/关键词` | 搜索 |
| `n` | 下一个匹配 |
| `q` | 退出 |

---

## 3. -n：查看最近日志

查看最近 20 行：

```bash
journalctl -n 20
```

查看最近 100 行：

```bash
journalctl -n 100
```

不分页直接输出：

```bash
journalctl -n 50 --no-pager
```

---

## 4. -f：实时跟踪日志

实时查看新日志：

```bash
journalctl -f
```

停止：

```text
Ctrl + C
```

这类似：

```bash
tail -f file.log
```

---

## 5. -u：查看指定服务日志

查看 Nginx 日志：

```bash
journalctl -u nginx
```

查看最近 50 行：

```bash
journalctl -u nginx -n 50
```

实时查看 Nginx 日志：

```bash
journalctl -u nginx -f
```

---

## 6. --since 和 --until：按时间过滤

查看今天日志：

```bash
journalctl --since today
```

查看最近 1 小时：

```bash
journalctl --since "1 hour ago"
```

查看指定时间范围：

```bash
journalctl --since "2026-07-03 09:00:00" --until "2026-07-03 10:00:00"
```

查看 Nginx 最近 1 小时日志：

```bash
journalctl -u nginx --since "1 hour ago"
```

---

## 7. -p：按日志级别过滤

查看错误及更严重日志：

```bash
journalctl -p err
```

常见级别：

| 级别 | 含义 |
| --- | --- |
| `emerg` | 系统不可用 |
| `alert` | 必须立即处理 |
| `crit` | 严重 |
| `err` | 错误 |
| `warning` | 警告 |
| `info` | 信息 |
| `debug` | 调试 |

查看 Nginx 错误：

```bash
journalctl -u nginx -p err
```

---

## 8. -xe：查看最近错误上下文

执行：

```bash
journalctl -xe
```

它常用于服务启动失败后的排查。

例如：

```bash
sudo systemctl restart nginx
systemctl status nginx
journalctl -xe
```

---

## 9. 本节小结

必须记住：

```bash
journalctl
journalctl -n 50
journalctl -f
journalctl -u nginx
journalctl -u nginx -n 50
journalctl -u nginx -f
journalctl --since "1 hour ago"
journalctl -u nginx --since today
journalctl -p err
journalctl -xe
```

完成后进入下一节：`07-stage-practice.md`。

