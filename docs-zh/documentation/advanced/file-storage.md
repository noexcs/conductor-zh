---
description: 使用文件存储在工作流之间交换不透明的文件句柄，无需暴露存储桶、对象路径或凭据，由文件 API 管理元数据并强制工作流所有权。
---

# 文件存储

## 背景

文件存储让工作流能够交换不透明的文件句柄，而无需暴露存储提供商的存储桶、对象路径或凭据。工作流可见的值始终是：

```text
conductor://file/<id>
```

文件 API 负责存储元数据并强制工作流所有权。所配置的后端负责存储字节数据。

## 可用的部署需要哪些条件

1. 受所配置持久化实现支持的文件元数据 DAO。
2. 启用文件存储并配置了后端。
3. 使用匹配的 Java SDK `FileClient` 配置的工作者。
4. 对于 `conductor` 后端，需要每个可接收内容请求的 Conductor 服务器节点都能访问的目录。

## 功能开关

文件存储默认禁用。请与后端一起启用：

```properties
conductor.file-storage.enabled=true
conductor.file-storage.type=conductor
```

## 选择后端

| 后端 | `conductor.file-storage.type` | 传输路径 | 分片 |
|---|---|---|---|
| Conductor 托管文件系统 | `conductor` | 通过 Conductor 的 HTTP `PUT`/`GET` | 不支持 |
| Amazon S3 | `s3` | 直传，提供商签名 URL | 支持 |
| Azure Blob Storage | `azure-blob` | 直传，提供商签名 URL | 支持 |
| Google Cloud Storage | `gcs` | 直传，提供商签名 URL | 不支持 |

对于 Conductor 托管的文件系统存储，使用 `conductor`。

## 通用属性

| 属性 | 说明 |
|---|---|
| `conductor.file-storage.enabled` | 启用文件 API 和存储配置。 |
| `conductor.file-storage.type` | 选择 `conductor`、`s3`、`azure-blob` 或 `gcs`。 |
| `conductor.file-storage.signed-url-expiration` | API 签发上传或下载 URL 时使用的有效期。 |
| `conductor.file-storage.multipart-threshold` | 客户端对支持分片的后端选择分片上传的阈值大小。 |

## Conductor 托管存储（`type=conductor`）

Conductor 托管后端将对象存储在服务器端文件系统中，并返回由 Conductor 服务的 HTTP URL。客户端**不会**收到 `file:` URL，也无需直接访问存储目录。

| 属性 | 说明 | 默认值 |
|---|---|---|
| `conductor.file-storage.conductor.directory` | 包含已上传文件对象的目录。 | `${java.io.tmpdir}/conductor/files-uploaded` |
| `conductor.file-storage.conductor.base-url` | 可选的公共 Conductor 源（origin），当入站请求的源不适合时用于签发内容 URL。 | 从入站请求派生 |
| `conductor.file-storage.conductor.max-size` | 接受的最大上传大小（字节）。超出的上传将被拒绝，部分对象会被删除。 | `104857600`（100 MiB） |
| `conductor.file-storage.conductor.signing.enabled` | 要求使用签名的内容 URL。 | `false` |
| `conductor.file-storage.conductor.signing.keys[n].id` | 包含在签名 URL 中的标识符。 | — |
| `conductor.file-storage.conductor.signing.keys[n].secret` | 该密钥的 HMAC 密钥。 | — |

除非配置了 `base-url`，内容 URL 的源（origin）来自转发或入站请求的源（origin）。当 Conductor 位于不保留公共协议、主机和端口的代理或负载均衡器之后时，请将 `base-url` 设置为外部可达的源（origin）。它必须是一个 HTTP(S) 源（origin）；Conductor 会追加原始内容路径。

!!! warning "多节点部署需要共享存储"
    每个可提供文件内容的 Conductor 服务器节点都必须挂载相同的
    `conductor.file-storage.conductor.directory`。请使用 NFS、EFS 或
    `ReadWriteMany` 卷等共享存储。节点本地目录可能在一个节点上接受上传，
    但之后路由到另一个节点的下载会失败。

### 通过 Conductor 进行 HTTP 传输

对于此后端，上传和下载 URL 都指回 Conductor：

```text
PUT /api/files/content/{workflowId}/{fileId}
GET /api/files/content/{workflowId}/{fileId}
```

客户端通过 HTTP 发送和接收原始文件主体。Conductor 将内容流式传输到所配置的文件系统或从中读取，因此客户端需要能够访问 Conductor 的 HTTP 端点，而不是底层文件系统。有关确切的请求和响应契约，请参阅 [文件 API](../api/files.md#conductor-managed-raw-content)。

### 可选的 URL 签名与密钥轮换

签名默认禁用。启用后，内容 URL 将包含 `op`、`exp`、`kid` 和 `sig` 查询参数。它们是 bearer（持有即凭证）：请勿将其记录到日志或包含在异常消息中。

```properties
conductor.file-storage.conductor.signing.enabled=true
conductor.file-storage.conductor.signing.keys[0].id=2026-07
conductor.file-storage.conductor.signing.keys[0].secret=${CONDUCTOR_FILE_SIGNING_KEY_CURRENT}
conductor.file-storage.conductor.signing.keys[1].id=2026-04
conductor.file-storage.conductor.signing.keys[1].secret=${CONDUCTOR_FILE_SIGNING_KEY_PREVIOUS}
```

第一个密钥为新签发的 URL 签名。所有已配置的密钥都可以验证现有 URL，这支持轮换：先添加新密钥，保留旧密钥直到使用其签发的 URL 过期，然后删除旧密钥。每个密钥 ID 必须唯一，且签名至少需要一个密钥。

## 对象存储后端

S3、Azure Blob Storage 和 GCS 保留各自的提供商特定配置和直传行为。Conductor 创建并授权文件记录，然后客户端将字节传输到提供商 URL。通过文件 API，只有 S3 和 Azure Blob Storage 支持分片上传。

## 配置示例

### Docker Compose

```properties
# config-postgres.properties
conductor.file-storage.enabled=true
conductor.file-storage.type=conductor
conductor.file-storage.conductor.directory=/data/conductor-files
conductor.file-storage.conductor.base-url=http://conductor:8080
conductor.file-storage.conductor.max-size=104857600
```

将 `/data/conductor-files` 挂载到服务器容器中。如果有多个服务器容器，请在每个容器中的该路径挂载相同的共享文件系统。

### 使用共享卷的 Kubernetes

```yaml
env:
  - name: conductor.file-storage.enabled
    value: "true"
  - name: conductor.file-storage.type
    value: conductor
  - name: conductor.file-storage.conductor.directory
    value: /var/lib/conductor/files
  - name: conductor.file-storage.conductor.base-url
    value: https://conductor.example.com
volumeMounts:
  - name: conductor-files
    mountPath: /var/lib/conductor/files
volumes:
  - name: conductor-files
    persistentVolumeClaim:
      claimName: conductor-files-rwx
```

持久卷必须支持所有 Conductor 服务器副本的并发访问。

## 验证配置

创建一个文件记录，将字节上传到返回的 URL，然后确认上传：

```shell
curl -sS -X POST http://localhost:8080/api/files \
  -H 'Content-Type: application/json' \
  -d '{"workflowId":"wf-docs-demo","fileName":"report.pdf","contentType":"application/pdf"}'

curl -X PUT --data-binary @report.pdf \
  'http://localhost:8080/api/files/content/wf-docs-demo/<file-id>'

curl -sS -X POST \
  'http://localhost:8080/api/files/wf-docs-demo/<file-id>/upload-complete'
```

对于 `conductor` 后端，成功的原始上传返回 `204 No Content`。完成请求会记录实际大小和内容哈希，然后将文件移至 `UPLOADED`。

## 授权模型

每个文件都有一个所属工作流。授权是有意非对称的：

| 操作 | 访问规则 |
|---|---|
| 创建 | 提供的文件所属工作流成为所有者。 |
| 上传内容、刷新上传 URL、完成上传以及分片变更 | 仅限精确的所有者。 |
| 元数据和下载内容 | 所有者的工作流家族：自身、祖先和后代。 |

这允许父工作流与子工作流之间交换句柄，同时防止任何一方修改另一个执行的进行中的上传。

## Java 工作者用法

工作者注入 `FileClient`，接受并返回句柄字符串，并显式调用上传或下载。任务执行器不会扫描工作者的输入或输出以查找文件对象，也不会自动上传。

```java
public final class ResizeWorker {
    private final FileClient files;

    public ResizeWorker(FileClient files) {
        this.files = files;
    }

    @WorkerTask("resize_image")
    public @OutputParam("image") String resize(
            @InputParam("image") String inputHandle,
            @WorkflowInstanceIdInputParam String workflowId) throws IOException {
        Path input = Files.createTempFile("image-", ".bin");
        Path output = Files.createTempFile("resized-", ".png");
        try {
            files.download(workflowId, inputHandle, input);
            resize(input, output);
            return files.upload(
                    workflowId,
                    output,
                    new FileUploadOptions().setContentType("image/png"));
        } finally {
            Files.deleteIfExists(input);
            Files.deleteIfExists(output);
        }
    }
}
```

有关公开的上传和下载形式，请参阅 [Java SDK 文件处理](../clientsdks/java-sdk.md#file-handling)；有关直接的 REST 访问，请参阅 [文件 API](../api/files.md)。

## 传输行为

`FileClient` 负责编排：请求验证、文件记录创建、重试策略、URL 刷新、完成对账和清理。内部传输适配器各执行一次传输尝试。

- 流式上传在创建服务器记录之前，会被缓冲到一个可重复读取的临时文件中。
- `conductor` 后端始终使用一个代理的 HTTP 请求，并受 `max-size` 限制。
- S3 和 Azure Blob Storage 可以使用分片上传；GCS 和 `conductor` 使用单个请求。
- 下载会写入一个唯一的同级临时文件，并且只有在响应完整之后才原子地替换目标文件。
- 内容 URL 会从错误中屏蔽。启用签名时，URL 即为 bearer 凭证。

## 从智能文件对象迁移

当前契约使用显式的 `FileClient` 调用和原始句柄字符串，取代了 `FileHandler`、`ManagedFileHandler`、`FileUploader` 和 `WorkflowFileClient`。

旧输出：

```json
{"fileHandleId":"conductor://file/abc","fileName":"report.pdf","contentType":"application/pdf"}
```

当前输出：

```json
"conductor://file/abc"
```

因此，混合的工作者版本会产生不兼容的工作流数据形状。在切换格式之前，请排空正在运行的工作流，或协调服务器和工作者的发布节奏。
