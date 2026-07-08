# 03：备份与恢复项目

本节目标：编写一个可用的备份脚本，完成打包、校验、列出内容和恢复测试。

本节命令顺序：

```text
1. tar -czf
2. tar -tzf
3. sha256sum
4. mkdir -p
5. tar -xzf
6. diff -r
7. bash -n
8. bash -x
```

---

## 1. 手动理解备份流程

进入工具箱目录：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
```

手动打包 `data` 目录：

```bash
tar -czf backup/data-manual.tar.gz data
```

查看压缩包内容：

```bash
tar -tzf backup/data-manual.tar.gz
```

生成校验值：

```bash
sha256sum backup/data-manual.tar.gz > backup/data-manual.tar.gz.sha256
cat backup/data-manual.tar.gz.sha256
```

---

## 2. 编写 backup.sh

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

base_name="$(basename "$source_dir")"
archive="${backup_dir}/${base_name}-${timestamp}.tar.gz"
checksum="${archive}.sha256"

tar -czf "$archive" "$source_dir"
sha256sum "$archive" > "$checksum"

echo "Backup created: $archive"
echo "Checksum created: $checksum"
EOF
```

授权：

```bash
chmod +x scripts/backup.sh
```

检查语法：

```bash
bash -n scripts/backup.sh
```

---

## 3. 运行备份脚本

```bash
./scripts/backup.sh data
ls -lh backup
```

查看最新备份：

```bash
latest_archive="$(ls -t backup/data-*.tar.gz | head -n 1)"
echo "$latest_archive"
tar -tzf "$latest_archive"
```

校验：

```bash
sha256sum -c "${latest_archive}.sha256"
```

---

## 4. 恢复测试

创建恢复目录：

```bash
mkdir -p tmp/restore-test
```

解压：

```bash
tar -xzf "$latest_archive" -C tmp/restore-test
```

查看：

```bash
tree tmp/restore-test
```

比较原目录和恢复目录：

```bash
diff -r data tmp/restore-test/data
```

如果没有输出，说明内容一致。

---

## 5. 调试脚本

```bash
bash -x scripts/backup.sh data
```

观察：

- `source_dir` 是否正确。
- `archive` 文件名是否包含时间。
- `sha256sum` 是否生成成功。

---

## 6. 本节小结

必须掌握：

```bash
tar -czf backup.tar.gz dir
tar -tzf backup.tar.gz
tar -xzf backup.tar.gz -C restore-dir
sha256sum file
sha256sum -c file.sha256
diff -r old-dir new-dir
bash -n scripts/backup.sh
bash -x scripts/backup.sh data
```

完成后进入下一节：`04-service-health-check.md`。

