# 05：日志分析项目

本节目标：编写日志分析脚本，统计访问日志中的状态码、热门 IP、错误请求，并汇总应用日志级别。

本节命令顺序：

```text
1. grep
2. awk
3. sort
4. uniq -c
5. sort -nr
6. head
7. tee
8. bash -n
```

---

## 1. 手动分析访问日志

进入工具箱目录：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
```

统计状态码：

```bash
awk '{print $5}' logs/access.log | sort | uniq -c | sort -nr
```

统计访问 IP：

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr
```

查看非 200 请求：

```bash
awk '$5 != 200 {print $0}' logs/access.log
```

---

## 2. 编写 top-ip.sh

```bash
cat > scripts/top-ip.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <access-log> [top-n]"
  exit 1
fi

log_file="$1"
top_n="${2:-10}"

if [ ! -f "$log_file" ]; then
  echo "Error: log file not found: $log_file"
  exit 1
fi

awk '{print $1}' "$log_file" | sort | uniq -c | sort -nr | head -n "$top_n"
EOF
```

授权和检查：

```bash
chmod +x scripts/top-ip.sh
bash -n scripts/top-ip.sh
```

运行：

```bash
./scripts/top-ip.sh logs/access.log
./scripts/top-ip.sh logs/access.log 3
```

---

## 3. 编写 log-summary.sh

```bash
cat > scripts/log-summary.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 2 ]; then
  echo "Usage: $0 <access-log> <app-log>"
  exit 1
fi

access_log="$1"
app_log="$2"

if [ ! -f "$access_log" ]; then
  echo "Error: access log not found: $access_log"
  exit 1
fi

if [ ! -f "$app_log" ]; then
  echo "Error: app log not found: $app_log"
  exit 1
fi

echo "## HTTP Status Count"
awk '{print $5}' "$access_log" | sort | uniq -c | sort -nr

echo
echo "## Top IP"
awk '{print $1}' "$access_log" | sort | uniq -c | sort -nr | head -n 10

echo
echo "## Non-200 Requests"
awk '$5 != 200 {print $0}' "$access_log"

echo
echo "## App Log Levels"
awk '{print $1}' "$app_log" | sort | uniq -c | sort -nr

echo
echo "## App Errors"
grep "ERROR" "$app_log" || true
EOF
```

授权和检查：

```bash
chmod +x scripts/log-summary.sh
bash -n scripts/log-summary.sh
```

运行并保存：

```bash
./scripts/log-summary.sh logs/access.log logs/app.log | tee output/log-summary.md
```

查看：

```bash
cat output/log-summary.md
```

---

## 4. 生成简短日报

```bash
{
  echo "# Daily Log Report"
  echo
  date
  echo
  ./scripts/log-summary.sh logs/access.log logs/app.log
} > output/daily-report.md
```

查看：

```bash
cat output/daily-report.md
```

---

## 5. 本节小结

必须掌握：

```bash
awk '{print $5}' logs/access.log | sort | uniq -c | sort -nr
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr
awk '$5 != 200 {print $0}' logs/access.log
awk '{print $1}' logs/app.log | sort | uniq -c | sort -nr
grep "ERROR" logs/app.log
./scripts/top-ip.sh logs/access.log 3
./scripts/log-summary.sh logs/access.log logs/app.log
```

完成后进入下一节：`06-nginx-deployment.md`。

