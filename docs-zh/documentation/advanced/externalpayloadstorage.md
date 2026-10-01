---
description: "外部载荷存储 —— 将大型 Conductor 工作流和任务载荷卸载到 S3 等外部存储中。"
---
# 外部载荷存储

!!!warning
    外部载荷存储目前仅实现为供 Java 客户端使用。其他语言的客户端库需要修改后才能启用此功能。  
    欢迎贡献代码。

## 背景
可以配置 Conductor，对工作流和任务载荷的输入与输出大小强制执行屏障（barriers）。  
这些屏障可以用作保护措施，防止将 conductor 用作数据持久化系统，并减轻其数据存储的压力。

## 屏障
Conductor 通常应用两种屏障：

* 软屏障
* 硬屏障


#### 软屏障

软屏障用于减轻 conductor 数据存储的压力。在某些特殊的工作流使用场景中，载荷的尺寸大到足以合理地被存储为工作流执行的一部分。  
在这种情况下，conductor 会将此类载荷的存储外部化到 S3，并在执行期间按需上传/下载到 S3。此过程对用户/工作者进程完全透明。  


#### 硬屏障
强制硬屏障是为了保护 conductor 后端，使其免受持久化和处理大量与工作流执行无关的数据所带来的压力。
在这种情况下，conductor 会拒绝此类载荷，并使工作流执行终止/失败，将 reasonForIncompletion 设置为详细说明载荷大小的相应错误消息。

## 使用

### 屏障设置

在 JVM 系统属性中将以下属性设置为期望的值：

| 属性 | 说明 | 默认值 |
| -- | -- | -- |
| conductor.app.workflowInputPayloadSizeThreshold | 工作流输入载荷的软屏障（KB） | 5120 |
| conductor.app.maxWorkflowInputPayloadSizeThreshold | 工作流输入载荷的硬屏障（KB） | 10240 |
| conductor.app.workflowOutputPayloadSizeThreshold | 工作流输出载荷的软屏障（KB） | 5120 |
| conductor.app.maxWorkflowOutputPayloadSizeThreshold | 工作流输出载荷的硬屏障（KB） | 10240 |
| conductor.app.taskInputPayloadSizeThreshold | 任务输入载荷的软屏障（KB） | 3072 |
| conductor.app.maxTaskInputPayloadSizeThreshold | 任务输入载荷的硬屏障（KB） | 10240 |
| conductor.app.taskOutputPayloadSizeThreshold | 任务输出载荷的软屏障（KB） | 3072 |
| conductor.app.maxTaskOutputPayloadSizeThreshold | 任务输出载荷的硬屏障（KB） | 10240 |

### Amazon S3

Conductor 提供了 [Amazon S3](https://aws.amazon.com/s3/) 的实现，用于外部化大型载荷存储。  
在 JVM 系统属性中设置以下属性：
```
conductor.external-payload-storage.type=S3
```

!!! note
    此[实现](https://github.com/conductor-oss/conductor/blob/main/awss3-storage/src/main/java/com/netflix/conductor/s3/storage/S3PayloadStorage.java#L44-L45)假定实例上已配置了 S3 访问权限。

在 JVM 系统属性中将以下属性设置为期望的值：

| 属性 | 说明 | 默认值 |
| --- | --- | --- |
| conductor.external-payload-storage.s3.bucketName | 存储载荷的 S3 存储桶 | |
| conductor.external-payload-storage.s3.signedUrlExpirationDuration | 载荷签名 URL 的过期时间（秒） | 5 |

载荷将以 `UUID.json` 文件的形式存储在上方配置的存储桶中，存储位置由载荷类型决定。关于对象键的确定方式，请参阅 [S3PayloadStorage 源代码](https://github.com/conductor-oss/conductor/blob/main/awss3-storage/src/main/java/com/netflix/conductor/s3/storage/S3PayloadStorage.java#L149-L167)。

### Azure Blob Storage

!!!note
    此实现假定你拥有一个 [Azure Blob Storage 账户的连接字符串或 SAS Token](https://github.com/Azure/azure-sdk-for-java/blob/master/sdk/storage/azure-storage-blob/README.md)。
    如果你希望签名 URL 会过期，则必须指定 Connection String。 

在 JVM 系统属性中将以下属性设置为期望的值：

| 属性 | 说明 | 默认值 |
| --- | --- | --- |
| workflow.external.payload.storage.azure_blob.connection_string | Azure Blob Storage 连接字符串。对 Url 签名时必需。 | |
| workflow.external.payload.storage.azure_blob.endpoint | Azure Blob Storage 端点。如果已设置 connection_string 则为可选项。 | |
| workflow.external.payload.storage.azure_blob.sas_token | Azure Blob Storage SAS Token。必须对 Service `Blob` 下的 Resource `Object` 拥有 `Read` 和 `Write` 权限。如果已设置 connection_string 则为可选项。 | |
| workflow.external.payload.storage.azure_blob.container_name | 存储载荷的 Azure Blob Storage 容器 | `conductor-payloads` |
| workflow.external.payload.storage.azure_blob.signedurlexpirationseconds | 载荷签名 URL 的过期时间（秒） | 5 |
| workflow.external.payload.storage.azure_blob.workflow_input_path | 工作流输入存储的路径前缀（使用随机 UUID 文件名） | workflow/input/ |
| workflow.external.payload.storage.azure_blob.workflow_output_path | 工作流输出存储的路径前缀（使用随机 UUID 文件名） | workflow/output/ |
| workflow.external.payload.storage.azure_blob.task_input_path | 任务输入存储的路径前缀（使用随机 UUID 文件名） | task/input/ |
| workflow.external.payload.storage.azure_blob.task_output_path | 任务输出存储的路径前缀（使用随机 UUID 文件名） | task/output/ |

载荷将存储在与 [Amazon S3](https://github.com/conductor-oss/conductor/blob/main/awss3-storage/src/main/java/com/netflix/conductor/s3/storage/S3PayloadStorage.java#L149-L167) 相同的路径结构中。

#### 使用 Azurite 测试

你可以使用 [Azurite](https://github.com/Azure/Azurite) 在本地模拟 Azure Storage，用于开发和测试。

#### 故障排查

使用 Elasticsearch 持久化时，你可能会收到 `java.lang.IllegalStateException`，因为 Netty 库两次调用了 `setAvailableProcessors`。要解决此问题，请设置：

```properties
es.set.netty.runtime.available.processors=false
```

如果要使用 `okhttp` 替代默认的 Netty HTTP 客户端，请添加以下依赖：

```
com.azure:azure-core-http-okhttp:${compatible version}
```

### PostgreSQL 存储

Frinx 提供了 [PostgreSQL Storage](https://www.postgresql.org/) 的实现，用于外部化大型载荷存储。

!!!note
    此实现假定你拥有一个[具有所有必需凭据的 PostgreSQL 数据库服务器](https://jdbc.postgresql.org/documentation/use/)。

将以下属性设置到你的 application.properties 中：

| 属性 | 说明 | 默认值 |
| --- | --- | --- |
| conductor.external-payload-storage.postgres.conductor-url | 可用于拉取 JSON 配置的 URL，这些配置将从 PostgreSQL 下载到 conductor 服务器。例如：本地开发时为 `{{ server_host }}` | `""` |
| conductor.external-payload-storage.postgres.url | PostgreSQL 数据库连接 URL。连接数据库时必需。 | |
| conductor.external-payload-storage.postgres.username | 连接 PostgreSQL 数据库的用户名。连接数据库时必需。 | |
| conductor.external-payload-storage.postgres.password | 连接 PostgreSQL 数据库的密码。连接数据库时必需。 | |
| conductor.external-payload-storage.postgres.table-name | 存储载荷的 PostgreSQL schema 和表名 | `external.external_payload` |
| conductor.external-payload-storage.postgres.max-data-rows | PostgreSQL 数据库中数据行的最大数量。超过此限制后，最旧的数据将被删除。 | Long.MAX_VALUE (9223372036854775807L) |
| conductor.external-payload-storage.postgres.max-data-days | PostgreSQL 数据库中数据年龄的最大天数。超过限制后，最旧的数据将被删除。 | 0 |
| conductor.external-payload-storage.postgres.max-data-months | PostgreSQL 数据库中数据年龄的最大月数。超过限制后，最旧的数据将被删除。 | 0 |
| conductor.external-payload-storage.postgres.max-data-years | PostgreSQL 数据库中数据年龄的最大年数。超过限制后，最旧的数据将被删除。 | 1 |

数据库中字段的最大数据年龄为：`years + months + days`  
载荷将以键（externalPayloadPath）`UUID.json` 存储在 PostgreSQL 数据库中，并且你可以
使用 `external-postgres-payload-resource` rest 控制器为此数据生成 URI。   
要使此 URI 正常工作，你必须正确设置 conductor-url 属性。
