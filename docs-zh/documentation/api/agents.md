---
description: "Conductor Agents REST API — 编译、部署、启动、观察和控制由 SDK 编写的持久化 Agent 执行，并向其响应。"
---

# Conductor Agents API

Conductor Agents 控制平面将 SDK 编写的 Conductor Agents 或框架 Agent（包括 OpenAI Agents、Google ADK、LangChain 和 LangGraph）编译为持久化的 Conductor 图，然后部署并运行这些图。框架搭建和交互式开发请使用 SDK；当你需要 CI/CD、运维控制台或自定义集成时，请使用这些 REST 端点。

仅当嵌入式 Conductor Agents 运行时通过 `conductor.integrations.ai.enabled=true` 启用时，这些端点才可用。

## 基础路径

```
http://localhost:8080/api/agent
```

## Agent 生命周期

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/compile` | 将内联 Agent 请求编译为计划（plan），但不部署也不运行。 |
| `POST` | `/inspect-plan` | 针对 Agent 配置校验并检查确定性计划。 |
| `POST` | `/deploy` | 编译并注册 Agent 定义，供后续运行。 |
| `POST` | `/start` | 启动已部署的 Agent 或内联的 Agent 配置。 |
| `GET` | `/list` | 列出已注册的 Agent。 |
| `GET` | `/{name}?version=` | 获取已注册的 Agent 定义。 |
| `DELETE` | `/{name}?version=` | 删除已注册的 Agent 定义。 |

`/compile`、`/deploy` 和 `/start` 接受 `AgentStartRequest`。要使用之前部署的 Agent，请提供 `name`，并可选项提供 `version`。要内联创建 Agent，请提供 Conductor Agent 的 `agentConfig`，或支持框架的 `framework` 加上框架特定的 `rawConfig`。

```json
{
  "name": "customer-support-agent",
  "version": 1,
  "prompt": "Summarize the customer's latest support case.",
  "sessionId": "case-1234"
}
```

启动已部署的 Agent：

```shell
curl -X POST 'http://localhost:8080/api/agent/start' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "customer-support-agent",
    "prompt": "Summarize the customer case.",
    "sessionId": "case-1234"
  }'
```

响应包含 `executionId`、`agentName`，以及 SDK 需要注册的 `requiredWorkers`（如有）。

## 观察与交互执行

| 方法 | 路径 | 用途 |
|---|---|---|
| `GET` | `/executions` | 搜索 Agent 执行。支持 `start`、`size`、`sort`、`freeText`、`status`、`agentName` 和 `classifier`。 |
| `GET` | `/executions/{executionId}` | 获取详细的执行状态。 |
| `GET` | `/{executionId}/status` | 获取用于轮询的轻量执行状态。 |
| `GET` | `/stream/{executionId}` | 打开 Agent 事件的 SSE 流。支持 `Last-Event-ID` 重连。 |
| `POST` | `/{executionId}/respond` | 向待处理的人机交互（human-in-the-loop）请求提供输出。 |
| `POST` | `/{executionId}/signal` | 向活跃 Agent 的上下文添加持久化消息。 |
| `POST` | `/events/{executionId}` | 接收框架工作者事件，例如 LangChain 或 LangGraph 事件。 |

当 Agent 正在等待人工输入时，使用 `respond`：

```shell
curl -X POST 'http://localhost:8080/api/agent/EXECUTION_ID/respond' \
  -H 'Content-Type: application/json' \
  -d '{"approved": true}'
```

## 控制执行

| 方法 | 路径 | 用途 |
|---|---|---|
| `PUT` | `/{executionId}/pause` | 暂停运行中的 Agent。 |
| `PUT` | `/{executionId}/resume` | 恢复已暂停的 Agent。 |
| `DELETE` | `/{executionId}/cancel?reason=` | 取消 Agent，并沿其图传播取消。 |
| `POST` | `/{executionId}/stop` | 请求在当前迭代结束后优雅停止。 |

## 提供方与技能端点

| 方法 | 路径 | 用途 |
|---|---|---|
| `GET` | `/api/providers/status` | 报告已配置哪些服务器端 AI 提供方；Ollama 还会报告其解析出的 URL 和可达性。 |
| `POST` | `/api/skills/register` | 上传技能包及其清单（manifest）。 |
| `GET` | `/api/skills` | 列出技能包。 |
| `GET` | `/api/skills/{name}` | 获取技能包的最新版本。 |
| `POST` | `/api/skills/{name}/versions/{version}/deploy` | 将特定技能包部署为 Agent。 |
| `DELETE` | `/api/skills/{name}/versions/{version}` | 删除技能包的某个版本。 |

仅当服务器上启用了技能包时，才存在技能端点。

## 相关指南

- [Conductor Agents](../../devguide/ai/conductor-agents.md) — SDK 创建、部署/服务（serve）生命周期，以及作为 `AGENT` 任务的使用。
- [框架 Agent](../../devguide/ai/agent-framework-recipes.md) — OpenAI Agents、Google ADK、LangChain、LangGraph、Vercel AI SDK 和 Conductor Agent 路径。
- [A2A 集成](../../devguide/ai/a2a-integration.md) — 远程 A2A Agent；这是独立的 `AGENT` 模式。
