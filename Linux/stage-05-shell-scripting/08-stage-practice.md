# 08：第五阶段综合练习

本节目标：完成一个个人 Shell 脚本工具箱，把第五阶段学到的语法串起来。

最终目录：

```text
shell-toolkit/
  scripts/
    backup.sh
    check-file.sh
    check-service.sh
    top-ip.sh
    batch-count.sh
  logs/
    access.log
  data/
    users.txt
  backup/
  output/
  README.md
```

---

## 1. 本练习会用到的语法和命令

按出现顺序：

```text
1. mkdir
2. cat > file <<'EOF'
3. chmod +x
4. variable=value
5. "$1"
6. "$@"
7. if
8. [ -f "$file" ]
9. [ -d "$dir" ]
10. for
11. while read -r
12. function
13. local
14. exit
15. bash -n
16. bash -x
17. tar
18. awk
19. sort
20. uniq
```

---

## 2. 创建工具箱目录

```bash
cd ~/linux-practice/stage-05
mkdir -p shell-toolkit/{scripts,logs,data,backup,output}
cd shell-toolkit
pwd
```

准备数据：

```bash
cat > data/users.txt <<'EOF'
alice
bob
carol
david
eve
EOF
```

准备日志：

```bash
cat > logs/access.log <<'EOF'
192.168.1.10 - GET /index.html 200
192.168.1.11 - GET /login 200
192.168.1.12 - POST /login 403
192.168.1.10 - GET /dashboard 200
192.168.1.13 - GET /missing 404
192.168.1.11 - GET /index.html 200
192.168.1.14 - GET /missing 404
192.168.1.12 - POST /login 403
192.168.1.10 - GET /settings 500
EOF
```

---

## 3. 脚本一：check-file.sh

目标：检查参数是文件、目录，还是不存在。

```bash
cat > scripts/check-file.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <path>"
  exit 1
fi

path="$1"

if [ -f "$path" ]; then
  echo "FILE: $path"
elif [ -d "$path" ]; then
  echo "DIR: $path"
else
  echo "NOT FOUND: $path"
  exit 1
fi
EOF
```

授权并运行：

```bash
chmod +x scripts/check-file.sh
./scripts/check-file.sh data/users.txt
./scripts/check-file.sh logs
./scripts/check-file.sh not-exist
```

---

## 4. 脚本二：batch-count.sh

目标：批量统计多个文件的行数。

```bash
cat > scripts/batch-count.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -eq 0 ]; then
  echo "Usage: $0 <file1> [file2...]"
  exit 1
fi

for file in "$@"; do
  if [ -f "$file" ]; then
    lines="$(wc -l < "$file")"
    echo "$file: $lines lines"
  else
    echo "Skip, not a file: $file"
  fi
done
EOF
```

授权并运行：

```bash
chmod +x scripts/batch-count.sh
./scripts/batch-count.sh data/users.txt logs/access.log not-exist
```

---

## 5. 脚本三：backup.sh

目标：备份指定目录。

```bash
cat > scripts/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 <source-dir>"
}

if [ "$#" -lt 1 ]; then
  usage
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
target="${backup_dir}/$(basename "$source_dir")-${timestamp}.tar.gz"

tar -czf "$target" "$source_dir"
echo "Backup created: $target"
EOF
```

授权并运行：

```bash
chmod +x scripts/backup.sh
./scripts/backup.sh data
ls -lh backup
```

---

## 6. 脚本四：top-ip.sh

目标：统计访问日志中出现最多的 IP。

```bash
cat > scripts/top-ip.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 <access-log> [top-n]"
}

if [ "$#" -lt 1 ]; then
  usage
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

授权并运行：

```bash
chmod +x scripts/top-ip.sh
./scripts/top-ip.sh logs/access.log
./scripts/top-ip.sh logs/access.log 3
```

---

## 7. 脚本五：check-service.sh

目标：检查 systemd 服务是否运行。

```bash
cat > scripts/check-service.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <service-name>"
  exit 1
fi

service_name="$1"

if ! command -v systemctl >/dev/null 2>&1; then
  echo "Error: systemctl not found"
  exit 1
fi

if systemctl is-active --quiet "$service_name"; then
  echo "OK: $service_name is running"
else
  echo "WARN: $service_name is not running"
  exit 1
fi
EOF
```

授权并运行：

```bash
chmod +x scripts/check-service.sh
./scripts/check-service.sh nginx
```

如果当前环境不支持 systemd，这个脚本可能返回错误，这是正常的。

---

## 8. 统一检查脚本语法

```bash
for script in scripts/*.sh; do
  echo "Checking $script"
  bash -n "$script"
done
```

没有输出错误，就说明语法检查通过。

---

## 9. 统一运行部分脚本

```bash
./scripts/check-file.sh data/users.txt
./scripts/batch-count.sh data/users.txt logs/access.log
./scripts/top-ip.sh logs/access.log 5
./scripts/backup.sh data
```

查看结果：

```bash
tree
ls -lh backup
```

---

## 10. 生成 README.md

````bash
cat > README.md <<'EOF'
# Shell Toolkit

## Scripts

- `check-file.sh`: check whether a path is a file or directory.
- `batch-count.sh`: count lines for multiple files.
- `backup.sh`: backup a directory to `backup/`.
- `top-ip.sh`: show top IP addresses from an access log.
- `check-service.sh`: check whether a systemd service is running.

## Examples

```bash
./scripts/check-file.sh data/users.txt
./scripts/batch-count.sh data/users.txt logs/access.log
./scripts/backup.sh data
./scripts/top-ip.sh logs/access.log 3
./scripts/check-service.sh nginx
```
EOF
````

查看：

```bash
cat README.md
```

---

## 11. 调试练习

用 `bash -x` 调试：

```bash
bash -x scripts/top-ip.sh logs/access.log 3
bash -x scripts/backup.sh data
```

观察：

- 参数是否正确。
- 变量是否正确。
- 条件判断是否走到预期分支。
- 管道命令是否输出正确结果。

---

## 12. 阶段验收题

请你不看答案，回答下面问题：

1. Shebang 的作用是什么？
2. `bash script.sh` 和 `./script.sh` 有什么区别？
3. 变量赋值时等号两边能不能有空格？
4. 为什么脚本中变量通常要写成 `"$var"`？
5. `$1`、`$#`、`"$@"` 分别是什么意思？
6. `$?` 表示什么？
7. `exit 0` 和 `exit 1` 有什么区别？
8. `[ -f "$file" ]` 是判断什么？
9. `for file in "$@"; do ... done` 是做什么？
10. `while read -r line` 适合什么场景？
11. 函数中 `local` 的作用是什么？
12. `bash -n` 和 `bash -x` 分别用来做什么？
13. `set -euo pipefail` 的作用是什么？

---

## 13. 阶段完成标准

当你能独立完成下面任务，就可以进入第六阶段：

- [ ] 写出带 Shebang 的脚本。
- [ ] 使用 `chmod +x` 让脚本可执行。
- [ ] 使用变量保存路径、日期、参数。
- [ ] 使用 `"$@"` 遍历所有参数。
- [ ] 使用 `if` 判断文件、目录和参数。
- [ ] 使用 `for` 批量处理文件。
- [ ] 使用 `while read -r` 逐行读取文件。
- [ ] 使用函数封装重复逻辑。
- [ ] 使用 `exit` 返回成功或失败状态。
- [ ] 使用 `bash -n` 检查语法。
- [ ] 使用 `bash -x` 调试脚本。
- [ ] 写出备份脚本、服务检查脚本和日志统计脚本。

---

## 14. 建议写入学习笔记

记录下面内容：

```text
1. 我写过的 5 个脚本及用途
2. Shell 脚本中最容易写错的 5 个地方
3. "$@" 和 $@ 的区别
4. [ ] 条件判断常用写法
5. bash -n 和 bash -x 的使用场景
6. 一次脚本报错的排查过程
```

完成这些记录后，第五阶段就真正学完了。
