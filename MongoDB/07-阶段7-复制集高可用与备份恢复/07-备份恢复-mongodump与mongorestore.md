# 07. 备份恢复：mongodump 与 mongorestore

备份的价值不在“备了”，而在“能恢复”。本节用 MongoDB Database Tools 中的 `mongodump` 和 `mongorestore` 做一次本地演练。

## 1. 安装 Database Tools

`mongodump` 和 `mongorestore` 属于 MongoDB Database Tools。

下载入口：

<https://www.mongodb.com/docs/database-tools/installation/>

验证：

```powershell
mongodump --version
mongorestore --version
```

如果命令不存在，需要把工具目录加入 PATH。

## 2. 准备数据

```javascript
use backup_stage7
db.users.insertMany([
  { name: "Alice", email: "alice@example.com", created_at: new Date() },
  { name: "Bob", email: "bob@example.com", created_at: new Date() }
])
db.orders.insertOne({
  order_no: "O-BACKUP-001",
  status: "paid",
  total_amount: 9900,
  created_at: new Date()
})
```

## 3. 备份单个数据库

```powershell
mkdir C:\Users\HP\Desktop\mongodb-backups

mongodump `
  --uri="mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0" `
  --db=backup_stage7 `
  --out="C:\Users\HP\Desktop\mongodb-backups\dump-001"
```

查看输出目录：

```powershell
Get-ChildItem -Recurse C:\Users\HP\Desktop\mongodb-backups\dump-001
```

## 4. 模拟误删

```javascript
use backup_stage7
db.users.drop()
db.orders.drop()
show collections
```

## 5. 恢复数据库

```powershell
mongorestore `
  --uri="mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0" `
  --db=backup_stage7 `
  "C:\Users\HP\Desktop\mongodb-backups\dump-001\backup_stage7"
```

验证：

```javascript
use backup_stage7
db.users.find().pretty()
db.orders.find().pretty()
```

## 6. 使用 --drop

如果恢复前希望先删除目标集合：

```powershell
mongorestore `
  --uri="mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0" `
  --drop `
  --db=backup_stage7 `
  "C:\Users\HP\Desktop\mongodb-backups\dump-001\backup_stage7"
```

注意：`--drop` 会先删除目标集合再恢复，生产环境要非常谨慎。

## 7. 备份注意事项

`mongodump` 适合：

- 小到中等数据量。
- 逻辑备份。
- 开发测试环境。
- 单库或单集合导出。

生产环境还要考虑：

- 数据量。
- 备份窗口。
- 一致性。
- 恢复时间目标 RTO。
- 恢复点目标 RPO。
- 加密和权限。
- 定期恢复演练。

## 8. RPO 和 RTO

RPO：

```text
最多能接受丢失多久的数据？
```

例如 RPO 5 分钟，表示最多接受丢失 5 分钟数据。

RTO：

```text
多久内必须恢复服务？
```

例如 RTO 30 分钟，表示 30 分钟内要恢复。

备份策略必须围绕 RPO/RTO 设计。

## 9. 本节练习

完成：

1. 安装并验证 Database Tools。
2. 创建 `backup_stage7` 数据库。
3. 使用 `mongodump` 备份。
4. 删除集合。
5. 使用 `mongorestore` 恢复。
6. 写一份恢复记录，包括耗时、命令和结果。

## 10. 本节小结

你需要记住：

- 备份必须能恢复才有意义。
- `mongodump` 做逻辑备份。
- `mongorestore` 做恢复。
- `--drop` 很危险，要谨慎。
- 生产备份要定义 RPO 和 RTO。

