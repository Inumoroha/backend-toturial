# 04：准备单台 Linux VM

## 1. 本节目标

这一节准备部署目标。

我们使用一台 Linux VM 作为 staging 环境。

目标：

- 安装 Docker 和 Docker Compose 插件。
- 创建部署用户。
- 创建应用目录。
- 配置 SSH 登录。
- 登录镜像仓库。
- 放置 Compose 文件和环境文件。

## 2. VM 最小要求

学习环境建议：

- Ubuntu LTS。
- 1 到 2 CPU。
- 1 到 2 GB 内存。
- 20 GB 磁盘。
- 能从 GitHub Actions runner SSH 访问。
- 能访问镜像仓库，例如 GHCR。

如果使用云服务器，安全组至少开放：

```text
22/tcp    SSH
8080/tcp  学习阶段 API 端口，可后续换成 80/443
```

真实生产不要直接暴露不必要端口。

## 3. 创建部署用户

在服务器上：

```bash
sudo adduser deploy
sudo usermod -aG docker deploy
```

让 `deploy` 用户使用 Docker。

重新登录后生效。

检查：

```bash
id deploy
```

## 4. 配置 SSH key

在本地生成部署专用 key：

```bash
ssh-keygen -t ed25519 -C "go-cicd-lab-deploy" -f ./go-cicd-lab-deploy
```

把公钥加入服务器：

```bash
ssh-copy-id -i ./go-cicd-lab-deploy.pub deploy@your-server-ip
```

测试：

```bash
ssh -i ./go-cicd-lab-deploy deploy@your-server-ip
```

把私钥内容放入 GitHub environment secret，例如：

```text
DEPLOY_SSH_KEY
```

不要把私钥提交到 Git。

## 5. 安装 Docker

按 Docker 官方文档安装 Docker Engine。

安装后检查：

```bash
docker version
docker compose version
```

如果 `deploy` 用户无法运行 Docker：

```bash
groups
```

确认它在 `docker` 组中，并重新登录。

## 6. 创建应用目录

```bash
sudo mkdir -p /opt/go-cicd-lab
sudo chown -R deploy:deploy /opt/go-cicd-lab
```

切换用户：

```bash
sudo -iu deploy
cd /opt/go-cicd-lab
```

创建目录：

```bash
mkdir -p releases
```

## 7. 登录 GHCR

如果镜像是私有的，服务器需要 pull 权限。

创建一个只读 token，或者使用平台推荐的最小权限凭证。

登录：

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u your-github-username --password-stdin
```

测试拉取：

```bash
docker pull ghcr.io/your-name/go-cicd-lab:sha-abc123
```

如果镜像公开，可以不登录，但真实业务服务通常私有。

## 8. 放置 Compose 文件

在服务器 `/opt/go-cicd-lab` 放：

```text
compose.yaml
.env
app.env
deploy.sh
```

初次可以手动复制，后续通过 GitHub Actions 同步或在部署脚本中生成。

确保权限：

```bash
chmod 600 .env app.env
chmod +x deploy.sh
```

## 9. SSH 安全建议

学习阶段至少做到：

- 禁止使用 root 部署。
- 使用专用 deploy 用户。
- 使用专用 SSH key。
- 私钥只放 GitHub environment secret。
- 服务器安全组只开放必要端口。

更进一步：

- 禁用密码登录。
- 限制 SSH 来源 IP。
- 使用 fail2ban。
- 使用短期凭证或自托管 runner。

## 10. 小练习

完成：

1. 创建 `deploy` 用户。
2. 测试 SSH 登录。
3. 安装 Docker。
4. 创建 `/opt/go-cicd-lab`。
5. 登录镜像仓库。
6. 手动拉取一次镜像。
7. 手动执行 `docker compose up -d`。

## 11. 本节小结

你现在应该理解：

- CD 需要一个准备好的部署目标。
- 不要用 root 用户作为部署用户。
- 服务器需要能 pull 镜像。
- 私钥和 registry token 都是敏感凭证。
- 应用目录和环境文件要有清晰权限。

