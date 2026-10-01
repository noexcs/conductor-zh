---
description: "Conductor LLM、向量、媒体、MCP 和 A2A 任务的 AI 系统任务参考。"
---

# AI 任务

AI 任务类型由 `ai` 模块注册。使用 `conductor.integrations.ai.enabled=true` 启用它们，然后配置相关的提供方、向量数据库、MCP 服务器或 A2A 端点。这些类型映射到服务器管理的任务；它们不是普通的用户自定义 `SIMPLE` 任务定义。

## LLM

| 类型 | 用途 |
|---|---|
| `LLM_CHAT_COMPLETE` | 聊天补全，包括模型工具调用支持。 |
| `LLM_TEXT_COMPLETE` | 单提示词文本补全。 |

两者都要求在任务输入中配置 LLM 提供方和模型。

## 嵌入与向量数据库

| 类型 | 用途 |
|---|---|
| `LLM_GENERATE_EMBEDDINGS` | 为提供的文本生成嵌入。 |
| `LLM_INDEX_TEXT` | 生成嵌入并索引文本。 |
| `LLM_STORE_EMBEDDINGS` | 存储预先计算的嵌入。 |
| `LLM_SEARCH_INDEX` | 嵌入查询并搜索索引。 |
| `LLM_SEARCH_EMBEDDINGS` | 使用提供的嵌入搜索索引。 |
| `LLM_GET_EMBEDDINGS` | 检索已存储的嵌入。 |

向量操作需要已配置的向量数据库；对于会生成向量的操作，还需要嵌入提供方/模型。

## 媒体与文档

| 类型 | 用途 |
|---|---|
| `GENERATE_IMAGE` | 从提示词生成图像。 |
| `GENERATE_AUDIO` | 从文本生成音频。 |
| `GENERATE_VIDEO` | 从支持的提示词或图像输入生成视频。 |
| `GENERATE_PDF` | 生成 PDF 文档。 |

提供方支持的媒体任务需要相应的提供方配置。PDF 生成使用 AI 模块注册的实现。

## MCP

| 类型 | 用途 |
|---|---|
| `LIST_MCP_TOOLS` | 发现 MCP 服务器暴露的工具。 |
| `CALL_MCP_TOOL` | 调用指定的 MCP 工具。 |

在任务输入中提供 MCP 服务器的连接信息。该服务器必须可从 Conductor 访问。

## A2A 智能体

| 类型 | 用途 |
|---|---|
| `GET_AGENT_CARD` | 从 `agentUrl` 获取 A2A 智能体卡片。 |
| `AGENT` | 向 Conductor 或远程 A2A 智能体发送工作并等待其结果。 |
| `CANCEL_AGENT` | 取消运行中的 Conductor 或远程 A2A 智能体任务。 |

这些任务工作者由 AI 集成注册。远程 A2A 调用需要 `agentUrl`；`CANCEL_AGENT` 对 Conductor 目标使用执行 ID，对远程目标使用智能体 URL 和任务 ID。协议详情请参阅[A2A 集成指南](../../../../devguide/ai/a2a-integration.md)。
