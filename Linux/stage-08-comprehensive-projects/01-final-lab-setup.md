# 01：综合实验准备

本节目标：创建第八阶段最终实验目录，准备日志、数据、报告目录，并明确后续项目的工作位置。

本节命令顺序：

```text
1. mkdir -p
2. cd
3. pwd
4. cat > file <<'EOF'
5. tree
6. ls -lah
7. chmod
```

---

## 1. 创建最终实验目录

执行：

```bash
mkdir -p ~/linux-practice/stage-08
cd ~/linux-practice/stage-08
pwd
```

创建项目结构：

```bash
mkdir -p linux-toolkit/{scripts,data,logs,backup,output,tmp}
mkdir -p reports
```

查看：

```bash
tree
```

预期结构：

```text
.
├── linux-toolkit
│   ├── backup
│   ├── data
│   ├── logs
│   ├── output
│   ├── scripts
│   └── tmp
└── reports
```

---

## 2. 准备访问日志

进入工具箱目录：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
```

创建访问日志：

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
192.168.1.15 - GET /api/users 200
192.168.1.16 - POST /api/orders 500
EOF
```

查看：

```bash
cat logs/access.log
wc -l logs/access.log
```

---

## 3. 准备应用日志

```bash
cat > logs/app.log <<'EOF'
INFO 2026-07-04 09:00:01 service started
INFO 2026-07-04 09:01:10 user alice login
WARN 2026-07-04 09:02:20 disk usage high
ERROR 2026-07-04 09:03:30 database timeout
INFO 2026-07-04 09:04:40 user bob login
ERROR 2026-07-04 09:05:50 config missing
WARN 2026-07-04 09:06:00 memory usage high
INFO 2026-07-04 09:07:10 service stopped
EOF
```

查看：

```bash
cat logs/app.log
grep "ERROR" logs/app.log
```

---

## 4. 准备用户数据

```bash
cat > data/users.csv <<'EOF'
id,name,role,city
1,Alice,admin,Beijing
2,Bob,developer,Shanghai
3,Carol,developer,Beijing
4,David,ops,Shenzhen
5,Eve,admin,Hangzhou
6,Frank,developer,Shanghai
EOF
```

查看：

```bash
cat data/users.csv
```

---

## 5. 创建项目说明占位文件

```bash
touch README.md
touch reports-placeholder.txt
```

后面会正式编写 `README.md` 和 `reports/final-report.md`。

---

## 6. 本节小结

必须完成：

```bash
tree ~/linux-practice/stage-08
cat ~/linux-practice/stage-08/linux-toolkit/logs/access.log
cat ~/linux-practice/stage-08/linux-toolkit/logs/app.log
cat ~/linux-practice/stage-08/linux-toolkit/data/users.csv
```

完成后进入下一节：`02-linux-toolkit-structure.md`。

