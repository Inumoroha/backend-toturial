# 4. GEO：附近门店与距离查询

Redis GEO 用来处理地理位置。

它适合轻量“附近的人”“附近门店”“距离计算”等场景。

学完这一节后，你应该能够：

- 使用 `GEOADD` 添加位置。
- 使用 `GEOSEARCH` 查询附近对象。
- 使用 `GEODIST` 计算距离。
- 在 Go 中实现附近门店查询。
- 知道 Redis GEO 的适用边界。

---

## 一、GEO 保存什么

GEO 保存：

```text
经度 longitude
纬度 latitude
成员 member
```

例如门店：

```redis
GEOADD shop:geo 116.397128 39.916527 shop:1001
```

含义：

```text
shop:1001 位于经度 116.397128，纬度 39.916527
```

注意顺序是：

```text
经度在前，纬度在后。
```

很多人会写反。

---

## 二、查询附近门店

查询某坐标 3 公里内门店：

```redis
GEOSEARCH shop:geo FROMLONLAT 116.40 39.90 BYRADIUS 3 km WITHDIST COUNT 20 ASC
```

含义：

- 以指定经纬度为中心。
- 半径 3 km。
- 返回距离。
- 最多 20 个。
- 按距离升序。

返回结果类似：

```text
shop:1001 0.8
shop:1002 1.4
```

---

## 三、计算两个门店距离

```redis
GEODIST shop:geo shop:1001 shop:1002 km
```

单位可以是：

```text
m
km
mi
ft
```

业务里常用 `m` 或 `km`。

---

## 四、获取门店坐标

```redis
GEOPOS shop:geo shop:1001
```

返回经纬度。

注意：Redis GEO 存储会有编码转换，返回值可能和原始值有微小差异。

不要用它做高精度测绘。

---

## 五、Go 添加门店位置

```go
func AddShopLocation(ctx context.Context, rdb *redis.Client, shopID int64, longitude, latitude float64) error {
    return rdb.GeoAdd(ctx, "shop:geo", &redis.GeoLocation{
        Name:      fmt.Sprintf("shop:%d", shopID),
        Longitude: longitude,
        Latitude:  latitude,
    }).Err()
}
```

调用：

```go
err := AddShopLocation(ctx, rdb, 1001, 116.397128, 39.916527)
```

---

## 六、Go 查询附近门店

```go
func SearchNearbyShops(ctx context.Context, rdb *redis.Client, longitude, latitude float64, radiusKm float64, limit int) ([]redis.GeoLocation, error) {
    return rdb.GeoSearchLocation(ctx, "shop:geo", &redis.GeoSearchLocationQuery{
        GeoSearchQuery: redis.GeoSearchQuery{
            Longitude:  longitude,
            Latitude:   latitude,
            Radius:     radiusKm,
            RadiusUnit: "km",
            Sort:       "ASC",
            Count:      limit,
        },
        WithDist: true,
    }).Result()
}
```

返回结果中包含：

```go
Name
Longitude
Latitude
Dist
```

查询到门店 ID 后，再批量查询门店详情缓存或数据库。

---

## 七、GEO 中只存 ID

GEO member 推荐只存 ID：

```text
shop:1001
```

不要把完整门店 JSON 放进 GEO。

门店详情应该放在：

```text
shop:detail:1001
```

或数据库中。

附近查询流程：

```text
1. GEOSEARCH 找到附近 shop_id。
2. MGET 门店详情缓存。
3. 缓存未命中查数据库。
4. 按距离排序返回。
```

---

## 八、城市分 key

如果全国门店都放一个 key：

```text
shop:geo
```

数据量很大时会难以管理。

可以按城市拆：

```text
shop:geo:beijing
shop:geo:shanghai
shop:geo:shenzhen
```

好处：

- 单 key 更小。
- 查询范围更明确。
- 清理和维护更方便。

缺点：

- 用户在城市边界附近时，可能需要查多个城市。
- 城市归属判断要额外处理。

---

## 九、附近的人要谨慎

附近的人和附近门店不同。

门店位置相对稳定，适合 Redis GEO。

用户位置频繁变化：

- 更新频率高。
- 隐私风险高。
- 需要在线状态。
- 需要过期清理。
- 可能产生大量写入。

如果做附近的人，至少要：

- 设置位置过期策略。
- 只保存在线用户。
- 按区域分片。
- 做隐私保护。
- 避免精确暴露坐标。

---

## 十、删除位置

GEO 底层是 ZSet。

删除成员用：

```redis
ZREM shop:geo shop:1001
```

Go：

```go
func RemoveShopLocation(ctx context.Context, rdb *redis.Client, shopID int64) error {
    return rdb.ZRem(ctx, "shop:geo", fmt.Sprintf("shop:%d", shopID)).Err()
}
```

门店下线或关闭时要删除位置。

---

## 十一、GEO 适合和不适合

适合：

- 附近门店。
- 附近设备。
- 简单距离查询。
- 城市内轻量位置检索。

不适合：

- 复杂多边形查询。
- 路线规划。
- 高精度 GIS。
- 复杂地图分析。
- 大规模实时轨迹计算。

复杂地理业务应该考虑专业 GIS 数据库或搜索引擎。

---

## 十二、常见错误

### 1. 经纬度写反

Redis GEO 是经度在前，纬度在后。

### 2. GEO member 存完整 JSON

详情更新困难，也浪费内存。

### 3. 所有城市放一个巨大 key

大规模数据下维护困难。

### 4. 门店关闭后不删除位置

用户会查到不可用门店。

### 5. 把 GEO 当专业 GIS

Redis GEO 适合轻量附近查询，不适合复杂地理计算。

---

## 十三、本节练习

请完成下面练习：

1. 写出添加门店坐标的 `GEOADD` 命令。
2. 写出查询 3 公里内门店的 `GEOSEARCH` 命令。
3. 写出计算两个门店距离的 `GEODIST` 命令。
4. 用 Go 实现 `AddShopLocation`。
5. 用 Go 实现 `SearchNearbyShops`。
6. 设计门店详情缓存 key。
7. 思考全国门店是否应该放在一个 GEO key。
8. 思考附近的人为什么比附近门店复杂。

---

## 十四、本节小结

这一节你学习了 Redis GEO。

你需要记住：

- `GEOADD` 添加经纬度，顺序是经度在前、纬度在后。
- `GEOSEARCH` 查询附近对象，`GEODIST` 计算距离。
- GEO member 推荐保存 ID，详情另查。
- 大规模数据可以按城市或区域拆 key。
- Redis GEO 适合轻量附近查询，不适合复杂 GIS。

下一节我们学习 Pub/Sub，理解轻量通知和它的不可靠边界。

