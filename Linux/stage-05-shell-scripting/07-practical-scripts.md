# 07：实用 Shell 脚本

本节目标：写出三个真正有用的小脚本：目录备份、服务检查、日志 IP 统计。

本节脚本顺序：

```text
1. backup.sh
2. check-service.sh
3. top-ip.sh
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. 备份脚本：backup.sh

目标：把指定目录打包备份到 `backup/` 目录。

创建脚本：

```bash
nano scripts/backup.sh
```

写入：

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <source-dir>"
  exit 1
fi

source_dir="$1"
backup_dir="backup"
timestamp="$(date '+%Y%m%d-%H%M%S')"

if [ ! -d "$source_dir" ]; then
  echo "Error: source directory not found: $source_dir"
  exit 1
fi

mkdir -p "$backup_dir"

base_name="$(basename "$source_dir")"
target_file="${backup_dir}/${base_name}-${timestamp}.tar.gz"

tar -czf "$target_file" "$source_dir"

echo "Backup created: $target_file"
```

添加执行权限：

```bash
chmod +x scripts/backup.sh
```

运行：

```bash
./scripts/backup.sh data
ls -lh backup
```

检查压缩包内容：

```bash
tar -tzf backup/*.tar.gz
```

---

## 2. 服务检查脚本：check-service.sh

目标：检查某个 systemd 服务是否正在运行。

创建脚本：

```bash
nano scripts/check-service.sh
```

写入：

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <service-name>"
  exit 1
fi

service_name="$1"

if systemctl is-active --quiet "$service_name"; then
  echo "OK: $service_name is running"
  exit 0
else
  echo "WARN: $service_name is not running"
  exit 1
fi
```

添加执行权限：

```bash
chmod +x scripts/check-service.sh
```

运行：

```bash
./scripts/check-service.sh nginx
echo "$?"
```

如果你在 WSL 中没有启用 systemd，这个脚本可能无法正常检查服务。可以在虚拟机里练习。

---

## 3. 日志 IP 统计脚本：top-ip.sh

目标：统计访问日志中出现最多的 IP。

创建脚本：

```bash
nano scripts/top-ip.sh
```

写入：

```bash
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
```

添加执行权限：

```bash
chmod +x scripts/top-ip.sh
```

运行：

```bash
./scripts/top-ip.sh logs/access.log
./scripts/top-ip.sh logs/access.log 3
```

这里：

```bash
top_n="${2:-10}"
```

表示如果第 2 个参数不存在，就默认使用 `10`。

---

## 4. 给脚本统一做语法检查

检查所有脚本：

```bash
for script in scripts/*.sh; do
  echo "Checking $script"
  bash -n "$script"
done
```

如果没有报错，说明语法基本没问题。

---

## 5. 用 bash -x 调试脚本

调试备份脚本：

```bash
bash -x scripts/backup.sh data
```

调试 IP 统计脚本：

```bash
bash -x scripts/top-ip.sh logs/access.log 3
```

调试时重点看：

- 变量是否符合预期。
- 条件判断有没有走错分支。
- 命令参数是否正确。

---

## 6. 本节小结

本节完成后，你应该拥有：

```text
scripts/backup.sh
scripts/check-service.sh
scripts/top-ip.sh
```

必须掌握的脚本结构：

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <arg>"
  exit 1
fi

arg="$1"

if [ ! -e "$arg" ]; then
  echo "Error: not found: $arg"
  exit 1
fi
```

完成后进入下一节：`08-stage-practice.md`。

