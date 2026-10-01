---
description: 文件 API 创建工作流作用域的元数据记录并签发上传与下载 URL，工作流通过 conductor://file/<id> 句柄交换文件。
---

# 文件 API

文件 API 创建工作流作用域的元数据记录，并签发上传和下载 URL。工作流可见的句柄是 `conductor://file/<file-id>`；路由路径变量使用裸的 `file-id`。

以下所有路由均相对于 Conductor 服务器 URL。

## 句柄与工作流作用域

每个文件都有一个拥有它的工作流。

- 创建文件以及每一次上传变更都需要精确的拥有者 `workflowId`。
- 元数据和下载接受拥有者工作流家族中的工作流，包括父工作流和子工作流。
- 在上传完成的记录将其标记为 `UPLOADED` 之前，文件不可下载。

## 单请求上传

### 1. 创建文件记录

```http
POST /api/files
Content-Type: application/json

{
  "workflowId": "3a5b8c2d-1234-5678-9abc-def012345678",
  "taskId": "optional-task-id",
  "fileName": "report.pdf",
  "contentType": "application/pdf"
}
```

`workflowId` 必填。`fileName`、`contentType` 和 `taskId` 是可选的元数据。

响应为 `201 Created`，其结构如下：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "fileName": "report.pdf",
  "contentType": "application/pdf",
  "storageType": "CONDUCTOR",
  "uploadStatus": "UPLOADING",
  "uploadUrl": "<backend upload URL>",
  "uploadUrlExpiresAt": 0,
  "createdAt": 0
}
```

`uploadUrlExpiresAt` 和 `createdAt` 是纪元毫秒。`storageType` 的值反映所配置的后端；对于 Conductor 管理的存储，其值为 `CONDUCTOR`。

### 2. 上传字节

将字节发送到 `uploadUrl`。传输协议取决于后端：

- `conductor`：该 URL 指向 Conductor 的原始内容端点。参见
  [Conductor 管理的原始内容](#conductor-managed-raw-content)。
- `s3`、`azure-blob` 和 `gcs`：该 URL 是提供方 URL，客户端直接传输到
  该对象存储。

### 3. 刷新已过期的上传 URL

```http
GET /api/files/{workflowId}/{fileId}/upload-url
```

响应为 `200 OK`：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "uploadUrl": "<backend upload URL>",
  "expiresAt": 0
}
```

`expiresAt` 是纪元毫秒。

### 4. 确认上传

字节存储完成后，确认上传：

```http
POST /api/files/{workflowId}/{fileId}/upload-complete
```

响应为 `200 OK`：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "uploadStatus": "UPLOADED",
  "contentHash": "<backend content hash>"
}
```

## Conductor 管理的原始内容 { #conductor-managed-raw-content }

当 `conductor.file-storage.type=conductor` 时，上传和下载 URL 使用 Conductor 的原始
内容路由。请求和响应体是原始文件字节，不是 JSON。

### 上传内容

```http
PUT /api/files/content/{workflowId}/{fileId}

<raw file bytes>
```

精确的响应为：

```http
204 No Content
```

没有响应体。需要拥有者的工作流 ID；其他工作流无法上传
或覆盖该文件。

### 下载内容

```http
GET /api/files/content/{workflowId}/{fileId}
```

精确的成功响应结构为：

```http
200 OK
Content-Type: <the stored file content type>
Content-Length: <the stored file size>

<raw file bytes>
```

控制器从存储流式传输响应体。它根据文件元数据设置 `Content-Type`，
根据存储的文件大小设置 `Content-Length`。文件必须已处于 `UPLOADED` 状态。

如果启用了内容 URL 签名，请使用创建、刷新或下载
URL 端点返回的签名 URL。签名的内容 URL 包含 `op`、`exp`、`kid` 和 `sig`；请勿将其写入日志。

## 读取元数据

```http
GET /api/files/{workflowId}/{fileId}
```

响应为 `200 OK`：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "fileName": "report.pdf",
  "contentType": "application/pdf",
  "fileSize": 0,
  "contentHash": "<backend content hash>",
  "storageType": "CONDUCTOR",
  "uploadStatus": "UPLOADED",
  "workflowId": "<owning workflow id>",
  "taskId": "optional-task-id",
  "createdAt": 0,
  "updatedAt": 0
}
```

`fileSize`、`createdAt` 和 `updatedAt` 是数值；时间戳为纪元毫秒。

## 下载

### 1. 获取下载 URL

```http
GET /api/files/{workflowId}/{fileId}/download-url
```

响应为 `200 OK`：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "downloadUrl": "<backend download URL>",
  "expiresAt": 0
}
```

对于 `conductor` 后端，`downloadUrl` 就是原始的 `GET /api/files/content/{workflowId}/{fileId}`
端点。对于对象存储后端，它是提供方 URL。

### 2. 下载字节

对 `downloadUrl` 发起 `GET`。对于 Conductor 管理的存储，响应是上面所示的原始流结构。
对于对象存储后端，请遵循提供方的签名 URL 约定。

## 分片上传

分片仅对实现了它的后端可用。S3 和 Azure Blob Storage 支持
分片路由；`conductor` 和 GCS 不支持。不要为 `conductor`
后端启动分片会话：Conductor 管理的上传是单请求的，并受
`conductor.file-storage.conductor.max-size` 限制。

### 1. 发起

```http
POST /api/files/{workflowId}/{fileId}/multipart
```

响应为 `200 OK`：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "uploadId": "<backend multipart upload id>"
}
```

### 2. 获取每个分片的 URL

```http
GET /api/files/{workflowId}/{fileId}/multipart/{uploadId}/part/{partNumber}
```

响应使用上传 URL 结构：

```json
{
  "fileHandleId": "conductor://file/<file-id>",
  "uploadUrl": "<provider part upload URL>",
  "expiresAt": 0
}
```

### 3. 完成

```http
POST /api/files/{workflowId}/{fileId}/multipart/{uploadId}/complete
Content-Type: application/json

{
  "partETags": ["<part 1 token>", "<part 2 token>"]
}
```

响应使用上文所示的上传完成结构。

### 中止失败的会话

```http
DELETE /api/files/{workflowId}/{fileId}/multipart/{uploadId}
```

响应为 `204 No Content`。

## 错误

| 状态 | 含义 |
|---|---|
| `400 Bad Request` | 请求无效、上传未完成，或所选后端不支持分片。 |
| `403 Forbidden` | 工作流不具备所需的拥有者或家族访问权限；启用签名时，签名的内容 URL 无效或已过期。 |
| `404 Not Found` | 文件 ID 不存在，或未启用文件存储。 |
| `405 Method Not Allowed` | 签名 URL 被用于错误的操作，例如对 `PUT` 使用了下载 URL。 |
| `413 Payload Too Large` | Conductor 管理的上传超出 `conductor.file-storage.conductor.max-size`。 |
