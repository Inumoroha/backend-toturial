# 02：个人 Linux 工具箱结构

本节目标：把 `linux-toolkit` 整理成一个清晰的小项目，建立脚本规范、README 和输出目录。

本节命令顺序：

```text
1. cd
2. tree
3. touch
4. chmod +x
5. cat > README.md
6. bash -n
```

---

## 1. 进入工具箱目录

```bash
cd ~/linux-practice/stage-08/linux-toolkit
pwd
tree
```

---

## 2. 创建脚本文件

```bash
touch scripts/backup.sh
touch scripts/check-service.sh
touch scripts/check-port.sh
touch scripts/top-ip.sh
touch scripts/log-summary.sh
```

添加执行权限：

```bash
chmod +x scripts/*.sh
```

查看：

```bash
ls -l scripts
```

---

## 3. 约定脚本规范

本工具箱里的脚本统一使用：

```bash
#!/usr/bin/env bash
set -euo pipefail
```

每个脚本都要包含：

```text
1. Usage 提示
2. 参数数量检查
3. 文件或服务是否存在的检查
4. 清晰的成功或失败输出
5. 正确的 exit 状态
```

---

## 4. 创建 README.md

```bash
cat > README.md <<'EOF'
# Linux Toolkit

This toolkit is a collection of Linux practice scripts.

## Scripts

- `backup.sh`: backup a directory and create checksum.
- `check-service.sh`: check whether a systemd service is active.
- `check-port.sh`: check whether a TCP port is listening.
- `top-ip.sh`: show top IP addresses from an access log.
- `log-summary.sh`: summarize status codes and app log levels.

## Directories

- `data/`: sample data files.
- `logs/`: sample logs.
- `backup/`: backup archives.
- `output/`: command outputs and reports.
- `tmp/`: temporary files.

## Basic Checks

Run syntax checks:

    for script in scripts/*.sh; do
      bash -n "$script"
    done
EOF
```

查看：

```bash
cat README.md
```

---

## 5. 创建脚本检查命令

目前脚本还是空文件，后面会逐个写入。先记住统一检查方式：

```bash
for script in scripts/*.sh; do
  echo "Checking $script"
  bash -n "$script"
done
```

---

## 6. 本节小结

必须完成：

```bash
tree ~/linux-practice/stage-08/linux-toolkit
ls -l ~/linux-practice/stage-08/linux-toolkit/scripts
cat ~/linux-practice/stage-08/linux-toolkit/README.md
```

完成后进入下一节：`03-backup-and-restore.md`。

