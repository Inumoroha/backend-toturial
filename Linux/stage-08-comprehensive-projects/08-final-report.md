# 08：生成最终报告

本节目标：把第八阶段的项目结果整理成一份 Markdown 报告，记录脚本、服务、日志分析、部署和故障排查成果。

本节命令顺序：

```text
1. cd
2. echo
3. cat
4. tee
5. tree
6. bash -n
7. systemctl
8. curl
9. df
10. du
```

---

## 1. 检查项目结构

```bash
cd ~/linux-practice/stage-08
tree -L 3
```

检查脚本：

```bash
cd linux-toolkit
for script in scripts/*.sh; do
  echo "Checking $script"
  bash -n "$script"
done
```

---

## 2. 生成 final-report.md

回到第八阶段目录：

```bash
cd ~/linux-practice/stage-08
```

生成报告：

```bash
cat > reports/final-report.md <<'EOF'
# Linux Stage 08 Final Report

## 1. Project Overview

This report records the final Linux comprehensive practice project.

## 2. Toolkit Structure

EOF
```

追加目录结构：

```bash
tree linux-toolkit -L 3 >> reports/final-report.md
```

追加脚本清单：

```bash
cat >> reports/final-report.md <<'EOF'

## 3. Scripts

EOF

ls -l linux-toolkit/scripts >> reports/final-report.md
```

---

## 3. 追加脚本检查结果

```bash
cat >> reports/final-report.md <<'EOF'

## 4. Script Syntax Check

EOF

cd linux-toolkit
for script in scripts/*.sh; do
  echo "Checking $script" >> ../reports/final-report.md
  bash -n "$script" >> ../reports/final-report.md 2>&1
done
cd ..
```

---

## 4. 追加日志分析结果

```bash
cat >> reports/final-report.md <<'EOF'

## 5. Log Analysis

EOF

cd linux-toolkit
./scripts/log-summary.sh logs/access.log logs/app.log >> ../reports/final-report.md
cd ..
```

---

## 5. 追加备份结果

```bash
cat >> reports/final-report.md <<'EOF'

## 6. Backup Files

EOF

ls -lh linux-toolkit/backup >> reports/final-report.md
```

---

## 6. 追加系统状态

```bash
cat >> reports/final-report.md <<'EOF'

## 7. System Checks

### Disk

EOF

df -h . >> reports/final-report.md

cat >> reports/final-report.md <<'EOF'

### Project Size

EOF

du -sh . >> reports/final-report.md
```

如果 Nginx 可用，追加：

```bash
cat >> reports/final-report.md <<'EOF'

### Nginx

EOF

systemctl is-active nginx >> reports/final-report.md 2>&1 || true
curl -I http://localhost >> reports/final-report.md 2>&1 || true
```

---

## 7. 追加故障排查记录

```bash
cat >> reports/final-report.md <<'EOF'

## 8. Troubleshooting Notes

EOF

cat reports/troubleshooting-notes.md >> reports/final-report.md 2>/dev/null || echo "No troubleshooting notes yet." >> reports/final-report.md
```

---

## 8. 查看最终报告

```bash
less reports/final-report.md
```

或者：

```bash
cat reports/final-report.md
```

---

## 9. 本节小结

必须完成：

```bash
tree ~/linux-practice/stage-08 -L 3
cat ~/linux-practice/stage-08/reports/final-report.md
```

完成后进入下一节：`09-graduation-checklist.md`。

