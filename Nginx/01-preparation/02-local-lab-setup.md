# 02 准备阶段：搭建本地实验环境

## 本节目标

搭建一套适合学习 Nginx 的实验环境。你可以使用 Linux 服务器、虚拟机、WSL、Docker 或云服务器。推荐顺序如下：

1. 有 Linux 基础：使用云服务器或本地虚拟机。
2. Windows 用户：使用 WSL2。
3. 想隔离环境：使用 Docker。

后续教程中的命令默认以 Linux 为主。

## 一、推荐目录结构

建议为学习创建一个目录：

```bash
mkdir -p ~/nginx-lab
cd ~/nginx-lab
```

后续可以按下面结构组织：

```text
nginx-lab/
  app/
    main.go
  static/
    index.html
    assets/
  nginx/
    nginx.conf
    conf.d/
```

这样做的好处是：

- Go 服务代码、静态资源、Nginx 配置分开。
- 项目实战时可以平滑迁移到 Docker Compose。
- 排查路径问题更清楚。

## 二、安装 Nginx

### Ubuntu 或 Debian

```bash
sudo apt update
sudo apt install -y nginx
nginx -v
```

### CentOS、Rocky Linux 或 AlmaLinux

```bash
sudo dnf install -y nginx
nginx -v
```

启动：

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

验证：

```bash
curl http://127.0.0.1
```

如果返回 Nginx 默认页面内容，说明安装成功。

## 三、认识常见配置目录

不同发行版路径略有不同，但常见位置如下：

```text
/etc/nginx/nginx.conf
/etc/nginx/conf.d/*.conf
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
/var/log/nginx/access.log
/var/log/nginx/error.log
/usr/share/nginx/html/
/var/www/html/
```

建议你执行：

```bash
ls -lah /etc/nginx
sudo nginx -T
```

`nginx -T` 会打印完整配置，包括被 `include` 引入的文件。学习阶段非常有用。

## 四、配置本地域名

学习时可以用本地域名模拟真实环境。

编辑 hosts 文件：

```bash
sudo vim /etc/hosts
```

加入：

```text
127.0.0.1 api.local
127.0.0.1 static.local
127.0.0.1 admin.local
```

验证：

```bash
curl http://api.local
```

如果你在 Windows 浏览器访问 WSL 里的服务，也可以在 Windows 的 hosts 文件里加同样的域名映射。

## 五、准备静态页面

创建目录：

```bash
mkdir -p ~/nginx-lab/static/assets
```

创建 `index.html`：

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nginx Lab</title>
  <link rel="stylesheet" href="/assets/style.css">
</head>
<body>
  <h1>Nginx Lab</h1>
  <p>静态资源服务已准备好。</p>
</body>
</html>
```

创建 `assets/style.css`：

```css
body {
  font-family: Arial, sans-serif;
  margin: 40px;
}
```

## 六、准备 Go 服务

创建目录：

```bash
mkdir -p ~/nginx-lab/app
cd ~/nginx-lab/app
go mod init nginx-lab-app
```

创建 `main.go`：

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
)

type response struct {
	Message string `json:"message"`
	Addr    string `json:"addr"`
	Host    string `json:"host"`
}

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	http.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(response{
			Message: "pong",
			Addr:    r.RemoteAddr,
			Host:    r.Host,
		})
	})

	log.Printf("listening on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

运行：

```bash
go run main.go
```

验证：

```bash
curl http://127.0.0.1:8080/api/ping
```

## 七、本节练习

1. 安装并启动 Nginx。
2. 找到 Nginx 主配置文件。
3. 准备静态页面目录。
4. 准备 Go 服务并访问 `/api/ping`。
5. 给 `api.local` 配置本地域名解析。

## 八、你应该掌握

学完本节，你应该已经有一个可以继续练习的环境，并知道：

- Nginx 配置文件在哪里。
- 日志文件在哪里。
- 静态资源放在哪里。
- Go 服务如何启动。
- 本地域名如何模拟真实域名。

