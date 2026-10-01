---
description: "OpenSearch 集成 —— 将 OpenSearch 配置为 Conductor 的索引后端，用于搜索工作流和任务。"
---
# OpenSearch

Conductor 支持使用 OpenSearch 作为索引后端，以便通过 UI 搜索工作流和任务。
针对 OpenSearch 2.x 和 3.x 分别提供了版本特定的模块。

## 快速开始

选择与你的 OpenSearch 集群版本匹配的模块，并设置 `conductor.indexing.type`：

```properties
# For OpenSearch 2.x
conductor.indexing.enabled=true
conductor.indexing.type=opensearch2
conductor.opensearch.url=http://localhost:9200

# For OpenSearch 3.x
conductor.indexing.enabled=true
conductor.indexing.type=opensearch3
conductor.opensearch.url=http://localhost:9200
```

Conductor 会在首次启动时创建其索引，并开始为工作流和任务建立索引。

## 支持的版本

| 模块 | `conductor.indexing.type` | OpenSearch 版本 | 客户端库 |
|---|---|---|---|
| `os-persistence-v2` | `opensearch2` | 2.x (2.0 – 2.18+) | opensearch-java 2.18.0 |
| `os-persistence-v3` | `opensearch3` | 3.x (3.0+) | opensearch-java 3.0.0 |

不再支持 OpenSearch 1.x。如果需要 1.x 支持，请参阅
[已归档的 os-persistence-v1 模块](https://github.com/conductor-oss/conductor-os-persistence-v1)。

## 配置参考

所有 OpenSearch 配置都使用 `conductor.opensearch.*` 命名空间。v2 和 v3
模块共享相同的属性名——只有 `conductor.indexing.type` 不同。

### 连接

| 属性 | 默认值 | 说明 |
|---|---|---|
| `conductor.opensearch.url` | `localhost:9201` | 以逗号分隔的 OpenSearch 节点 URL。同时支持 HTTP 和 HTTPS。 |
| `conductor.opensearch.username` | _(无)_ | 基本认证的用户名。 |
| `conductor.opensearch.password` | _(无)_ | 基本认证的密码。 |

多节点示例：

```properties
conductor.opensearch.url=http://os-node1:9200,http://os-node2:9200,http://os-node3:9200
```

### 索引管理

| 属性 | 默认值 | 说明 |
|---|---|---|
| `conductor.opensearch.indexPrefix` | `conductor` | 所有 Conductor 管理的索引的前缀。 |
| `conductor.opensearch.indexShardCount` | `5` | 每个索引的主分片数。 |
| `conductor.opensearch.indexReplicasCount` | `0` | 每个索引的副本分片数。 |
| `conductor.opensearch.autoIndexManagementEnabled` | `true` | Conductor 是否自动创建和管理索引。设置为 `false` 可在外部管理索引。 |
| `conductor.opensearch.clusterHealthColor` | `green` | Conductor 启动前等待的集群健康颜色。单节点集群请使用 `yellow`。 |

### 性能调优

| 属性 | 默认值 | 说明 |
|---|---|---|
| `conductor.opensearch.indexBatchSize` | `1` | 异步模式下每批的文档数。 |
| `conductor.opensearch.asyncWorkerQueueSize` | `100` | 异步索引任务队列深度。 |
| `conductor.opensearch.asyncMaxPoolSize` | `12` | 异步索引线程最大数。 |
| `conductor.opensearch.asyncBufferFlushTimeout` | `10s` | 异步缓冲被刷新前持有的最长时间。 |
| `conductor.opensearch.taskLogResultLimit` | `10` | 每次搜索返回的最大任务日志条目数。 |
| `conductor.opensearch.restClientConnectionRequestTimeout` | `-1` | REST 客户端连接请求超时时间（毫秒）。`-1` 表示不限制。 |

## 配置示例

### 开发环境（单节点，无认证）

```properties
conductor.indexing.enabled=true
conductor.indexing.type=opensearch2
conductor.opensearch.url=http://localhost:9200
conductor.opensearch.indexPrefix=conductor
conductor.opensearch.indexReplicasCount=0
conductor.opensearch.clusterHealthColor=yellow
```

### 生产环境（多节点、认证、OpenSearch 2.x）

```properties
conductor.indexing.enabled=true
conductor.indexing.type=opensearch2
conductor.opensearch.url=https://os-node1:9200,https://os-node2:9200,https://os-node3:9200
conductor.opensearch.username=conductor_user
conductor.opensearch.password=secure_password
conductor.opensearch.indexPrefix=conductor
conductor.opensearch.indexShardCount=5
conductor.opensearch.indexReplicasCount=1
conductor.opensearch.clusterHealthColor=green
conductor.opensearch.asyncWorkerQueueSize=500
conductor.opensearch.asyncMaxPoolSize=24
conductor.opensearch.indexBatchSize=10
```

### OpenSearch 3.x

```properties
conductor.indexing.enabled=true
conductor.indexing.type=opensearch3
conductor.opensearch.url=http://localhost:9200
conductor.opensearch.indexPrefix=conductor
conductor.opensearch.indexReplicasCount=0
conductor.opensearch.clusterHealthColor=yellow
```

## 使用 Docker Compose 运行

两个版本都提供了预构建的 Docker Compose 配置：

```shell
# OpenSearch 2.x
docker compose -f docker/docker-compose-redis-os2.yaml up

# OpenSearch 3.x
docker compose -f docker/docker-compose-redis-os3.yaml up
```

两者都会启动 Conductor、Redis 和相应版本的 OpenSearch。

## 从旧的 `opensearch` 类型迁移

通用的 `conductor.indexing.type=opensearch` 已被弃用。使用此值启动服务器将显示一条错误消息，指引你使用新的配置。

**迁移前：**

```properties
conductor.indexing.type=opensearch
conductor.elasticsearch.url=http://localhost:9200
conductor.elasticsearch.indexName=conductor
```

**迁移后：**

```properties
conductor.indexing.type=opensearch2   # or opensearch3
conductor.opensearch.url=http://localhost:9200
conductor.opensearch.indexPrefix=conductor
```

`conductor.elasticsearch.*` 命名空间仍会被接受，以保持向后兼容。检测到这些
值时，它们会被使用，并在启动时记录弃用警告。请在下一个主要版本发布前迁移到
`conductor.opensearch.*`。

### 旧属性映射

| 旧属性（`conductor.elasticsearch.*`） | 新属性（`conductor.opensearch.*`） |
|---|---|
| `url` | `url` |
| `indexName` | `indexPrefix` |
| `clusterHealthColor` | `clusterHealthColor` |
| `indexBatchSize` | `indexBatchSize` |
| `asyncWorkerQueueSize` | `asyncWorkerQueueSize` |
| `asyncMaxPoolSize` | `asyncMaxPoolSize` |
| `indexShardCount` | `indexShardCount` |
| `indexReplicasCount` | `indexReplicasCount` |
| `taskLogResultLimit` | `taskLogResultLimit` |
| `username` | `username` |
| `password` | `password` |

## 禁用索引

要在不进行搜索索引的情况下运行 Conductor（这将禁用 UI 中的工作流搜索）：

```properties
conductor.indexing.enabled=false
```

## 故障排查

### Conductor 启动失败：集群健康超时

对于单节点开发集群，请设置：

```properties
conductor.opensearch.clusterHealthColor=yellow
```

单节点集群无法达到 `green` 健康状态，因为副本分片无法被分配。

### Conductor 启动失败：`NoClassDefFoundError: org.opensearch.Version`

此错误出现在旧版本的 `os-persistence` 模块中，已在当前按版本划分的模块中解决。请确保 `conductor.indexing.type` 设置为 `opensearch2` 或 `opensearch3`。

### Docker 中配置更改不生效

配置文件在构建时被打包进 Docker 镜像。更改 `config-*.properties` 之后：

```shell
docker compose -f docker/docker-compose-redis-os2.yaml build
docker compose -f docker/docker-compose-redis-os2.yaml up
```

或者，将配置文件作为 Docker 卷挂载，以便在不重新构建的情况下获取更改。

## 参见

- [os-persistence-v2 README](https://github.com/conductor-oss/conductor/blob/main/os-persistence-v2/README.md)
- [os-persistence-v3 README](https://github.com/conductor-oss/conductor/blob/main/os-persistence-v3/README.md)
- [Issue #678](https://github.com/conductor-oss/conductor/issues/678) —— OpenSearch 改进 epic
- [OpenSearch 文档](https://opensearch.org/docs/latest/)
