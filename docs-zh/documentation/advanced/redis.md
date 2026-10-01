---
description: "Redis 后端 —— 将 Redis Standalone、Cluster 或 Sentinel 配置为 Conductor 的数据库和队列后端。"
---
# Redis

通过设置以下属性，将 Redis 配置为数据库和队列后端。

## `conductor.db.type` 和 `conductor.queue.type`

| 取值                          | 说明                                                                            |
|--------------------------------|----------------------------------------------------------------------------------------|
| redis_standalone               | Redis Standalone 配置。                                                        |
| redis_cluster                  | Redis Cluster 配置。                                                           |
| redis_sentinel                 | Redis Sentinel 配置。                                                          |

## `conductor.redis.hosts`

期望的格式为以分号分隔的 `host:port:rack`，例如： 

```properties
conductor.redis.hosts=host0:6379:us-east-1c;host1:6379:us-east-1c;host2:6379:us-east-1c
```

## `conductor.redis.database`
sentinel 和 standalone 配置支持除默认值 0 以外的 Redis 数据库值。 
Redis cluster 模式仅使用 database 0，该配置将被忽略。

```properties
conductor.redis.database=1
```


## `conductor.redis.username`

现已支持使用用户名和密码认证的 [Redis ACL](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/)。

用户名属性应设置为 `conductor.redis.username`，例如：
```properties
conductor.redis.username=conductor
```
如果未设置，客户端使用 `default` 作为用户名。

密码应作为第一个主机的第 4 个参数 `host:port:rack:password` 设置，例如：

```properties
conductor.redis.hosts=host0:6379:us-east-1c:my_str0ng_pazz;host1:6379:us-east-1c;host2:6379:us-east-1c
```

**说明**

- 在 cluster 中，所有节点使用相同的用户名和密码。
- 在 sentinel 配置中，sentinel 和 redis 节点使用相同的数据库索引、用户名和密码。
