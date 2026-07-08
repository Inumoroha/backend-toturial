# 03 生产部署：配置拆分、发布流程与回滚

## 本节目标

学习如何把 Nginx 配置组织得更适合生产维护，并建立发布和回滚流程。

## 一、为什么要拆配置

单个巨大 `nginx.conf` 会越来越难维护。建议拆成：

```text
/etc/nginx/
  nginx.conf
  conf.d/
    00-log-format.conf
    10-upstream-go-api.conf
    20-app-http.conf
    30-app-https.conf
    40-static.conf
```

拆分原则：

- 全局配置放主文件。
- 日志格式独立。
- upstream 独立。
- 每个域名或服务一个 server 文件。
- 复杂站点可以按功能继续拆。

## 二、命名顺序

使用数字前缀：

```text
00-
10-
20-
30-
```

好处：

- 一眼看出加载和阅读顺序。
- 后续可以插入 `15-xxx.conf`。
- 团队协作更清楚。

## 三、每次发布前必须检查

```bash
sudo nginx -t
```

如果失败，不要 reload。

查看完整配置：

```bash
sudo nginx -T > /tmp/nginx.full.conf
```

对比配置变更：

```bash
diff -u /backup/nginx.full.conf /tmp/nginx.full.conf
```

## 四、配置备份

简单备份：

```bash
sudo tar czf ~/nginx-backup-$(date +%Y%m%d-%H%M%S).tar.gz /etc/nginx
```

更推荐：

- 把 Nginx 配置纳入 Git。
- 生产发布只从仓库或 CI 产物同步。
- 每次发布有版本号。

## 五、发布流程模板

1. 本地或测试环境修改配置。
2. 执行 `nginx -t`。
3. 提交 Git。
4. 同步到服务器。
5. 服务器再次执行 `nginx -t`。
6. reload Nginx。
7. 执行健康检查。
8. 查看 access log 和 error log。
9. 如异常，立即回滚。

## 六、回滚流程模板

如果使用 Git：

```bash
git checkout 上一个稳定版本
sudo nginx -t
sudo systemctl reload nginx
```

如果使用备份包：

```bash
sudo cp -a /etc/nginx /etc/nginx.bad.$(date +%Y%m%d-%H%M%S)
sudo tar xzf nginx-backup-xxx.tar.gz -C /
sudo nginx -t
sudo systemctl reload nginx
```

回滚后验证：

```bash
curl -I https://your-domain
sudo tail -n 50 /var/log/nginx/error.log
```

## 七、零停机 reload 的注意事项

Nginx reload 通常比较平滑，但这些情况仍要小心：

- 新配置语法正确但 upstream 写错。
- 证书文件路径正确但证书内容不匹配。
- location 匹配改变导致 API 路由失效。
- 缓存配置改变导致用户看到旧数据。

所以 reload 后必须做业务验证，不只是 `nginx -t`。

## 八、上线验证清单

- 首页是否正常。
- API 健康检查是否正常。
- HTTPS 证书是否正常。
- 日志是否还在写入。
- error log 是否新增异常。
- 关键接口是否能访问。
- 502、504 是否上升。

## 九、本节练习

1. 把已有配置拆成 `00`、`10`、`20` 文件。
2. 用 `nginx -T` 导出完整配置。
3. 做一次配置备份。
4. 模拟错误配置发布失败。
5. 模拟回滚到旧配置。

## 十、你应该掌握

学完本节，你应该能把 Nginx 配置当成正式工程资产管理，而不是在服务器上随手改文件。

