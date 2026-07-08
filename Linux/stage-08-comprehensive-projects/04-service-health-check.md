# 04：服务健康检查项目

本节目标：编写服务检查脚本和端口检查脚本，能判断服务是否运行、端口是否监听、HTTP 是否可访问。

本节命令顺序：

```text
1. systemctl is-active
2. systemctl status
3. ss -tuln
4. ss -tunlp
5. curl -I
6. bash -n
7. exit 0 / exit 1
```

说明：如果 WSL 不支持 systemd，可以在虚拟机或服务器中练习 `systemctl` 部分。

---

## 1. 手动检查服务状态

进入工具箱目录：

```bash
cd ~/linux-practice/stage-08/linux-toolkit
```

检查 Nginx：

```bash
systemctl is-active nginx
systemctl status nginx
```

如果没有安装 Nginx：

```bash
sudo apt update
sudo apt install nginx
```

启动：

```bash
sudo systemctl start nginx
```

---

## 2. 编写 check-service.sh

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
  echo "ERROR: systemctl not found"
  exit 1
fi

if systemctl is-active --quiet "$service_name"; then
  echo "OK: $service_name is active"
  exit 0
else
  echo "WARN: $service_name is not active"
  systemctl status "$service_name" --no-pager || true
  exit 1
fi
EOF
```

授权和检查：

```bash
chmod +x scripts/check-service.sh
bash -n scripts/check-service.sh
```

运行：

```bash
./scripts/check-service.sh nginx
echo "$?"
```

---

## 3. 手动检查端口

查看监听端口：

```bash
ss -tuln
```

查看 80 端口：

```bash
ss -tuln | grep ':80'
```

查看进程：

```bash
sudo ss -tunlp | grep ':80'
```

---

## 4. 编写 check-port.sh

```bash
cat > scripts/check-port.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <port>"
  exit 1
fi

port="$1"

if ss -tuln | awk '{print $5}' | grep -Eq "[:.]${port}$"; then
  echo "OK: port $port is listening"
  exit 0
else
  echo "WARN: port $port is not listening"
  exit 1
fi
EOF
```

授权和检查：

```bash
chmod +x scripts/check-port.sh
bash -n scripts/check-port.sh
```

运行：

```bash
./scripts/check-port.sh 80
./scripts/check-port.sh 22
```

---

## 5. HTTP 健康检查

手动检查：

```bash
curl -I http://localhost
```

只输出状态码：

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost
```

追加到输出文件：

```bash
{
  echo "Service nginx:"
  ./scripts/check-service.sh nginx || true
  echo "Port 80:"
  ./scripts/check-port.sh 80 || true
  echo "HTTP code:"
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost || true
} > output/health-check.txt
```

查看：

```bash
cat output/health-check.txt
```

---

## 6. 本节小结

必须掌握：

```bash
systemctl is-active nginx
systemctl status nginx
ss -tuln
sudo ss -tunlp
curl -I http://localhost
curl -s -o /dev/null -w "%{http_code}\n" http://localhost
./scripts/check-service.sh nginx
./scripts/check-port.sh 80
```

完成后进入下一节：`05-log-analysis-project.md`。

