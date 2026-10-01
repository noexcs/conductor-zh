---
description: "将 Conductor 连接到它周围的系统：消息代理与 Webhook、MCP 工具服务器，以及远程 A2A 智能体。"
---

# 集成

集成将 Conductor 连接到它周围的系统。它们分为三类。事件驱动编排在工作流与外部世界之间传递消息。MCP 集成将智能体连接到工具。A2A 集成将 Conductor 连接到在其他地方运行的智能体。

- **[事件驱动编排](../how-tos/event-bus.md)**：把工作流消息发布到代理（broker），从传入消息和 Webhook 启动或推进工作流，向等待中的执行发送信号，并发出状态事件。
- **[MCP 集成](../ai/mcp-guide.md)**：从工作流和智能体中通过 Model Context Protocol 发现并调用工具。
- **[A2A 集成](../ai/a2a-integration.md)**：通过 Agent2Agent 协议把独立部署的智能体作为一个持久化工作流步骤来调用，或者暴露你自己的智能体。
